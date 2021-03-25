# Chapter 3. Watermarks
So far, we have been looking at stream processing from the perspective of the pipeline author or data scientist. Chapter 2 introduced watermarks as part of the answer to the fundamental questions of where in event-time processing is taking place and when in processing time results are materialized. In this chapter, we approach the same questions, but instead from the perspective of the underlying mechanics of the stream processing system. Looking at these mechanics will help us motivate, understand, and apply the concepts around watermarks. We discuss how watermarks are created at the point of data ingress, how they propagate through a data processing pipeline, and how they affect output timestamps. We also demonstrate how watermarks preserve the guarantees that are necessary for answering the questions of where in event-time data are processed and when it is materialized, while dealing with unbounded data.

Definition
Consider any pipeline that ingests data and outputs results continuously. We wish to solve the general problem of when it is safe to call an event-time window closed, meaning that the window does not expect any more data. To do so we would like to characterize the progress that the pipeline is making relative to its unbounded input.

One naive approach for solving the event-time windowing problem would be to simply base our event-time windows on the current processing time. As we saw in Chapter 1, we quickly run into trouble—data processing and transport is not instantaneous, so processing and event times are almost never equal. Any hiccup or spike in our pipeline might cause us to incorrectly assign messages to windows. Ultimately, this strategy fails because we have no robust way to make any guarantees about such windows.

Another intuitive, but ultimately incorrect, approach would be to consider the rate of messages processed by the pipeline. Although this is an interesting metric, the rate may vary arbitrarily with changes in input, variability of expected results, resources available for processing, and so on. Even more important, rate does not help answer the fundamental questions of completeness. Specifically, rate does not tell us when we have seen all of the messages for a particular time interval. In a real-world system, there will be situations in which messages are not making progress through the system. This could be the result of transient errors (such as crashes, network failures, machine downtime), or the result of persistent errors such as application-level failures that require changes to the application logic or other manual intervention to resolve. Of course, if lots of failures are occurring, a rate-of-processing metric might be a good proxy for detecting this. However a rate metric could never tell us that a single message is failing to make progress through our pipeline. Even a single such message, however, can arbitrarily affect the correctness of the output results.

We require a more robust measure of progress. To arrive there, we make one fundamental assumption about our streaming data: each message has an associated logical event timestamp. This assumption is reasonable in the context of continuously arriving unbounded data because this implies the continuous generation of input data. In most cases, we can take the time of the original event’s occurrence as its logical event timestamp. With all input messages containing an event timestamp, we can then examine the distribution of such timestamps in any pipeline. Such a pipeline might be distributed to process in parallel over many agents and consuming input messages with no guarantee of ordering between individual shards. Thus, the set of event timestamps for active in-flight messages in this pipeline will form a distribution, as illustrated in Figure 3-1.

Messages are ingested by the pipeline, processed, and eventually marked completed. Each message is either “in-flight,” meaning that it has been received but not yet completed, or “completed,” meaning that no more processing on behalf of this message is required. If we examine the distribution of messages by event time, it will look something like Figure 3-1. As time advances, more messages will be added to the “in-flight” distribution on the right, and more of those messages from the “in-flight” part of the distribution will be completed and moved into the “completed” distribution.

Figure 3-1. Distribution of in-flight and completed message event times within a streaming pipeline. New messages arrive as input and remain “in-flight” until processing for them completes. The leftmost edge of the “in-flight” distribution corresponds to the oldest unprocessed element at any given moment.
There is a key point on this distribution, located at the leftmost edge of the “in-flight” distribution, corresponding to the oldest event timestamp of any unprocessed message of our pipeline. We use this value to define the watermark:

The watermark is a monotonically1 increasing timestamp of the oldest work not yet completed.

There are two fundamental properties that are provided by this definition that make it useful:

Completeness
If the watermark has advanced past some timestamp T, we are guaranteed by its monotonic property that no more processing will occur for on-time (nonlate data) events at or before T. Therefore, we can correctly emit any aggregations at or before T. In other words, the watermark allows us to know when it is correct to close a window.

Visibility
If a message is stuck in our pipeline for any reason, the watermark cannot advance. Furthermore, we will be able to find the source of the problem by examining the message that is preventing the watermark from advancing.

Source Watermark Creation
Where do these watermarks come from? To establish a watermark for a data source, we must assign a logical event timestamp to every message entering the pipeline from that source. As Chapter 2 informs us, all watermark creation falls into one of two broad categories: perfect or heuristic. To remind ourselves about the difference between perfect and heuristic watermarks, let’s look at Figure 3-2, which presents the windowed summation example from Chapter 2.

Figure 3-2. Windowed summation with perfect (left) and heuristic (right) watermarks
Notice that the distinguishing feature is that perfect watermarks ensure that the watermark accounts for all data, whereas heuristic watermarks admit some late-data elements.

After the watermark is created as either perfect or heuristic, watermarks remain so throughout the rest of the pipeline. As to what makes watermark creation perfect or heuristic, it depends a great deal on the nature of the source that’s being consumed. To see why, let’s look at a few examples of each type of watermark creation.

Perfect Watermark Creation
Perfect watermark creation assigns timestamps to incoming messages in such a way that the resulting watermark is a strict guarantee that no data with event times less than the watermark will ever be seen again from this source. Pipelines using perfect watermark creation never have to deal with late data; that is, data that arrive after the watermark has advanced past the event times of newly arriving messages. However, perfect watermark creation requires perfect knowledge of the input, and thus is impractical for many real-world distributed input sources. Here are a couple of examples of use cases that can create perfect watermarks:

Ingress timestamping
A source that assigns ingress times as the event times for data entering the system can create a perfect watermark. In this case, the source watermark simply tracks the current processing time as observed by the pipeline. This is essentially the method that nearly all streaming systems supporting windowing prior to 2016 used.

Because event times are assigned from a single, monotonically increasing source (actual processing time), the system thus has perfect knowledge about which timestamps will come next in the stream of data. As a result, event-time progress and windowing semantics become vastly easier to reason about. The downside, of course, is that the watermark has no correlation to the event times of the data themselves; those event times were effectively discarded, and the watermark instead merely tracks the progress of data relative to its arrival in the system.

Static sets of time-ordered logs
A statically sized2 input source of time-ordered logs (e.g., an Apache Kafka topic with a static set of partitions, where each partition of the source contains monotonically increasing event times) would be relatively straightforward source atop which to create a perfect watermark. To do so, the source would simply track the minimum event time of unprocessed data across the known and static set of source partitions (i.e., the minimum of the event times of the most recently read record in each of the partitions).

Similar to the aforementioned ingress timestamps, the system has perfect knowledge about which timestamps will come next, thanks to the fact that event times across the static set of partitions are known to increase monotonically. This is effectively a form of bounded out-of-order processing; the amount of disorder across the known set of partitions is bounded by the minimum observed event time among those partitions.

Typically, the only way you can guarantee monotonically increasing timestamps within partitions is if the timestamps within those partitions are assigned as data are written to it; for example, by web frontends logging events directly into Kafka. Though still a limited use case, this is definitely a much more useful one than ingress timestamping upon arrival at the data processing system because the watermark tracks meaningful event times of the underlying data.

Heuristic Watermark Creation
Heuristic watermark creation, on the other hand, creates a watermark that is merely an estimate that no data with event times less than the watermark will ever be seen again. Pipelines using heuristic watermark creation might need to deal with some amount of late data. Late data is any data that arrives after the watermark has advanced past the event time of this data. Late data is only possible with heuristic watermark creation. If the heuristic is a reasonably good one, the amount of late data might be very small, and the watermark remains useful as a completion estimate. The system still needs to provide a way for the user to cope with late data if it’s to support use cases requiring correctness (e.g., things like billing).

For many real-world, distributed input sources, it’s computationally or operationally impractical to construct a perfect watermark, but still possible to build a highly accurate heuristic watermark by taking advantage of structural features of the input data source. Following are two example for which heuristic watermarks (of varying quality) are possible:

Dynamic sets of time-ordered logs
Consider a dynamic set of structured log files (each individual file containing records with monotonically increasing event times relative to other records in the same file but with no fixed relationship of event times between files), where the full set of expected log files (i.e., partitions, in Kafka parlance) is not known at runtime. Such inputs are often found in global-scale services constructed and managed by a number of independent teams. In such a use case, creating a perfect watermark over the input is intractable, but creating an accurate heuristic watermark is quite possible.

By tracking the minimum event times of unprocessed data in the existing set of log files, monitoring growth rates, and utilizing external information like network topology and bandwidth availability, you can create a remarkably accurate watermark, even given the lack of perfect knowledge of all the inputs. This type of input source is one of the most common types of unbounded datasets found at Google, so we have extensive experience with creating and analyzing watermark quality for such scenarios and have seen them used to good effect across a number of use cases.

Google Cloud Pub/Sub
Cloud Pub/Sub is an interesting use case. Pub/Sub currently makes no guarantees on in-order delivery; even if a single publisher publishes two messages in order, there’s a chance (usually small) that they might be delivered out of order (this is due to the dynamic nature of the underlying architecture, which allows for transparent scaling up to very high levels of throughput with zero user intervention). As a result, there’s no way to guarantee a perfect watermark for Cloud Pub/Sub. The Cloud Dataflow team has, however, built a reasonably accurate heuristic watermark by taking advantage of what knowledge is available about the data in Cloud Pub/Sub. The implementation of this heuristic is discussed at length as a case study later in this chapter.

Consider an example where users play a mobile game, and their scores are sent to our pipeline for processing: you can generally assume that for any source utilizing mobile devices for input it will be generally impossible to provide a perfect watermark. Due to the problem of devices that go offline for extended periods of time, there’s just no way to provide any sort of reasonable estimate of absolute completeness for such a data source. You can, however, imagine building a watermark that accurately tracks input completeness for devices that are currently online, similar to the Google Pub/Sub watermark described a moment ago. Users who are actively online are likely the most relevant subset of users from the perspective of providing low-latency results anyway, so this often isn’t as much of a shortcoming as you might initially think.

With heuristic watermark creation, broadly speaking, the more that is known about the source, the better the heuristic, and the fewer late data items will be seen. There is no one-size-fits-all solution, given that the types of sources, distributions of events, and usage patterns will vary greatly. But in either case (perfect or heuristic), after a watermark is created at the input source, the system can propagate the watermark through the pipeline perfectly. This means perfect watermarks will remain perfect downstream, and heuristic watermarks will remain strictly as heuristic as they were when established. This is the benefit of the watermark approach: you can reduce the complexity of tracking completeness in a pipeline entirely to the problem of creating a watermark at the source.

Watermark Propagation
So far, we have considered only the watermark for the inputs within the context of a single operation or stage. However, most real-world pipelines consist of multiple stages. Understanding how watermarks propagate across independent stages is important in understanding how they affect the pipeline as a whole and the observed latency of its results.

PIPELINE STAGES
Different stages are typically necessary every time your pipeline groups data together by some new dimension. For example, if you had a pipeline that consumed raw data, computed some per-user aggregates, and then used those per-user aggregates to compute some per-team aggregates, you’d likely end up with a three-stage pipeline:

One consuming the raw, ungrouped data

One grouping the data by user and computing per-user aggregates

One grouping the data by team and computing per-team aggregates

We learn more about the effects of grouping on pipeline shapes in Chapter 6.

Watermarks are created at input sources, as discussed in the preceding section. They then conceptually flow through the system as data progress through it.3 You can track watermarks at varying levels of granularity. For pipelines comprising multiple distinct stages, each stage likely tracks its own watermark, whose value is a function of all the inputs and stages that come before it. Therefore, stages that come later in the pipeline will have watermarks that are further in the past (because they’ve seen less of the overall input).

We can define watermarks at the boundaries of any single operation, or stage, in the pipeline. This is useful not only in understanding the relative progress that each stage in the pipeline is making, but for dispatching timely results independently and as soon as possible for each individual stage. We give the following definitions for the watermarks at the boundaries of stages:

An input watermark, which captures the progress of everything upstream of that stage (i.e., how complete the input is for that stage). For sources, the input watermark is a source-specific function creating the watermark for the input data. For nonsource stages, the input watermark is defined as the minimum of the output watermarks of all shards/partitions/instances of all of its upstream sources and stages.

An output watermark, which captures the progress of the stage itself, and is essentially defined as the minimum of the stage’s input watermark and the event times of all nonlate data active messages within the stage. Exactly what “active” encompasses is somewhat dependent upon the operations a given stage actually performs, and the implementation of the stream processing system. It typically includes data buffered for aggregation but not yet materialized downstream, pending output data in flight to downstream stages, and so on.

One nice feature of defining an input and output watermark for a specific stage is that we can use these to calculate the amount of event-time latency introduced by a stage. Subtracting the value of a stage’s output watermark from the value of its input watermark gives the amount of event-time latency or lag introduced by the stage. This lag is the notion of how far delayed behind real time the output of each stage will be. As an example, a stage performing 10-second windowed aggregations will have a lag of 10 seconds or more, meaning that the output of the stage will be at least that much delayed behind the input and real time. Definitions of input and output watermarks provide a recursive relationship of watermarks throughout a pipeline. Each subsequent stage in a pipeline delays the watermark as necessary, based on event-time lag of the stage.

Processing within each stage is also not monolithic. We can segment the processing within one stage into a flow with several conceptual components, each of which contributes to the output watermark. As mentioned previously, the exact nature of these components depends on the operations the stage performs and the implementation of the system. Conceptually, each such component serves as a buffer where active messages can reside until some operation has completed. For example, as data arrives, it is buffered for processing. Processing might then write the data to state for later delayed aggregation. Delayed aggregation, when triggered, might write the results to an output buffer awaiting consumption from a downstream stage, as shown in Figure 3-3.


Figure 3-3. Example system components of a streaming system stage, containing buffers of in-flight data. Each will have associated watermark tracking, and the overall output watermark of the stage will be the minimum of the watermarks across all such buffers.
We can track each such buffer with its own watermark. The minimum of the watermarks across the buffers of each stage forms the output watermark of the stage. Thus the output watermark could be the minimum of the following:

Per-source watermark—for each sending stage.

Per-external input watermark—for sources external to the pipeline

Per-state component watermark—for each type of state that can be written

Per-output buffer watermark—for each receiving stage

Making watermarks available at this level of granularity also provides better visibility into the behavior of the system. The watermarks track locations of messages across various buffers in the system, allowing for easier diagnosis of stuckness.

Understanding Watermark Propagation
To get a better sense for the relationship between input and output watermarks and how they affect watermark propagation, let’s look at an example. Let’s consider gaming scores, but instead of computing sums of team scores, we’re going to take a stab at measuring user engagement levels. We’ll do this by first calculating per-user session lengths, under the assumption that the amount of time a user stays engaged with the game is a reasonable proxy for how much they’re enjoying it. After answering our four questions once to calculate sessions lengths, we’ll then answer them a second time to calculate average session lengths within fixed periods of time.

To make our example even more interesting, lets say that we are working with two datasets, one for Mobile Scores and one for Console Scores. We would like to perform identical score calculations via integer summation in parallel over these two independant datasets. One pipeline is calculating scores for users playing on mobile devices, whereas the other is for users playing on home gaming consoles, perhaps due to different data collection strategies employed for the different platforms. The important point is that these two stages are performing the same operation but over different data, and thus with very different output watermarks.

To begin, let’s take a look at Example 3-1 to see what the abbreviated code for what the first section of this pipeline might be like.

Example 3-1. Calculating session lengths
PCollection<Double> mobileSessions = IO.read(new MobileInputSource())
  .apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
               .triggering(AtWatermark())
               .discardingFiredPanes())
  .apply(CalculateWindowLength());

PCollection<Double> consoleSessions = IO.read(new ConsoleInputSource())
  .apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
               .triggering(AtWatermark())
               .discardingFiredPanes())
  .apply(CalculateWindowLength());
Here, we read in each of our inputs independently, and whereas previously we were keying our collections by team, in this example we key by user. After that, for the first stage of each pipeline, we window into sessions and then call a custom PTransform named CalculateWindowLength. This PTransform simply groups by key (i.e., User) and then computes the per-user session length by treating the size of the current window as the value for that window. In this case, we’re fine with the default trigger (AtWatermark) and accumulation mode (discardingFiredPanes) settings, but I’ve listed them explicitly for completeness. The output for each pipeline for two particular users might look something like Figure 3-4.

Figure 3-4. Per-user session lengths across two different input pipelines
Because we need to track data across multiple stages, we track everything related to Mobile Scores in red, everything related to Console Scores in blue, while the watermark and output for Average Session Lengths in Figure 3-5 are yellow.

We have answered the four questions of what, where, when, and how to compute individual session lengths. Next we’ll answer them a second time to transform those session lengths into global session-length averages within fixed windows of time. This requires us to first flatten our two data sources into one, and then re-window into fixed windows; we’ve already captured the important essence of the session in the session-length value we computed, and we now want to compute a global average of those sessions within consistent windows of time over the course of the day. Example 3-2 shows the code for this.

Example 3-2. Calculating session lengths
PCollection<Double> mobileSessions = IO.read(new MobileInputSource())
  .apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
               .triggering(AtWatermark())
               .discardingFiredPanes())
  .apply(CalculateWindowLength());

PCollection<Double> consoleSessions = IO.read(new ConsoleInputSource())
  .apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
               .triggering(AtWatermark())
               .discardingFiredPanes())
  .apply(CalculateWindowLength());

PCollection<Float> averageSessionLengths = PCollectionList
  .of(mobileSessions).and(consoleSessions)
  .apply(Flatten.pCollections())
  .apply(Window.into(FixedWindows.of(Duration.standardMinutes(2)))
               .triggering(AtWatermark())
  .apply(Mean.globally());
If we were to see this pipeline in action, it would look something like Figure 3-5. As before, the two input pipelines are computing individual session lengths for mobile and console players. Those session lengths then feed into the second stage of the pipeline, where global session-length averages are computed in fixed windows.

Figure 3-5. Average session lengths of mobile and console gaming sessions
Let’s walk through some of this example, given that there’s a lot going on. The two important points here are:

The output watermark for each of the Mobile Sessions and Console Sessions stages is at least as old as the corresponding input watermark of each, and in reality a little bit older. This is because in a real system computing answers takes time, and we don’t allow the output watermark to advance until processing for a given input has completed.

The input watermark for the Average Session Lengths stage is the minimum of the output watermarks for the two stages directly upstream.

The result is that the downstream input watermark is an alias for the minimum composition of the upstream output watermarks. Note that this matches the definitions for those two types of watermarks earlier in the chapter. Also notice how watermarks further downstream are further in the past, capturing the intuitive notion that upstream stages are going to be further ahead in time than the stages that follow them.

One observation worth making here is just how cleanly we were able to ask the questions again in Example 3-1 to substantially alter the results of the pipeline. Whereas before we simply computed per-user session lengths, we now compute two-minute global session-length averages. This provides a much more insightful look into the overall behaviors of the users playing our games and gives you a tiny glimpse of the difference between simple data transformations and real data science.

Even better, now that we understand the basics of how this pipeline operates, we can look more closely at one of the more subtle issues related to asking the four questions over again: output timestamps.

Watermark Propagation and Output Timestamps
In Figure 3-5, I glossed over some of the details of output timestamps. But if you look closely at the second stage in the diagram, you can see that each of the outputs from the first stage was assigned a timestamp that matched the end of its window. Although that’s a fairly natural choice for output timestamps, it’s not the only valid choice. As you know from earlier in this chapter, watermarks are never allowed to move backward. Given that restriction, you can infer that the range of valid timestamps for a given window begins with the timestamp of the earliest nonlate record in the window (because only nonlate records are guaranteed to hold a watermark up) and extends all the way to positive infinity. That’s quite a lot of options. In practice, however, there tend to be only a few choices that make sense in most circumstances:

End of the window4
Using the end of the window is the only safe choice if you want the output timestamp to be representative of the window bounds. As we’ll see in a moment, it also allows the smoothest watermark progression out of all of the options.

Timestamp of first nonlate element
Using the timestamp of the first nonlate element is a good choice when you want to keep your watermarks as conservative as possible. The trade-off, however, is that watermark progress will likely be more hindered, as we’ll also see shortly.

Timestamp of a specific element
For certain use cases, the timestamp of some other arbitrary (from the system’s perspective) element is the right choice. Imagine a use case in which you’re joining a stream of queries to a stream of clicks on results for that query. After performing the join, some systems will find the timestamp of the query to be more useful; others will prefer the timestamp of the click. Any such timestamp is valid from a watermark correctness perspective, as long as it corresponded to an element that did not arrive late.

Having thought a bit about some alternate options for output timestamps, let’s look at what effects the choice of output timestamp can have on the overall pipeline. To make the changes as dramatic as possible, in Example 3-3 and Figure 3-6, we’ll switch to using the earliest timestamp possible for the window: the timestamp of the first nonlate element as the timestamp for the window. 

Example 3-3. Average session lengths pipeline, that output timestamps for session windows set at earliest element
PCollection<Double> mobileSessions = IO.read(new MobileInputSource())
  .apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
               .triggering(AtWatermark())
               .withTimestampCombiner(EARLIEST)
               .discardingFiredPanes())
  .apply(CalculateWindowLength());

PCollection<Double> consoleSessions = IO.read(new ConsoleInputSource())
  .apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
               .triggering(AtWatermark())
               .withTimestampCombiner(EARLIEST)
               .discardingFiredPanes())
  .apply(CalculateWindowLength());

PCollection<Float> averageSessionLengths = PCollectionList
  .of(mobileSessions).and(consoleSessions)
  .apply(Flatten.pCollections())
  .apply(Window.into(FixedWindows.of(Duration.standardMinutes(2)))
               .triggering(AtWatermark())
  .apply(Mean.globally());
Figure 3-6. Average session lengths for sessions that are output at the timestamp of the earliest element
To help call out the effect of the output timestamp choice, look at the dashed lines in the first stages showing what the output watermark for each stage is being held to. The output watermark is delayed by our choice of timestamp, as compared to Figures 3-7 and 3-8, in which the output timestamp was chosen to be the end of the window. You can see from this diagram that the input watermark of the second stage is thus subsequently also delayed.


Figure 3-7. Comparison of watermarks and results with different choice of window outout timestamps. The watermarks in this figure correspond to output timestamps at the end of the session windows (i.e., Figure 3-5).

Figure 3-8. In this figure, the watermarks are at the beginning of the session windows (i.e., Figure 3-6). We can see that the watermark line in this figure is more delayed, and the resulting average session lengths are different.
As far as differences in this version compared to Figure 3-7, two are worth noting:

Watermark delay
Compared to Figure 3-5, the watermark proceeds much more slowly in Figure 3-6. This is because the output watermark for the first stage is held back to the timestamp of the first element in every window until the input for that window becomes complete. Only after a given window has been materialized is the output watermark (and thus the downstream input watermark) allowed to advance.

Semantic differences
Because the session timestamps are now assigned to match the earliest nonlate element in the session, the individual sessions often end up in different fixed window buckets when we then calculate the session-length averages in the next stage. There’s nothing inherently right or wrong about either of the two options we’ve seen so far; they’re just different. But it’s important to understand that they will be different as well as have an intuition for the way in which they’ll be different so that you can make the correct choice for your specific use case when the time comes.

The Tricky Case of Overlapping Windows
One additional subtle but important issue regarding output timestamps is how to handle sliding windows. The naive approach of setting the output timestamp to the earliest element can very easily lead to delays downstream due to watermarks being (correctly) held back. To see why, consider an example pipeline with two stages, each using the same type of sliding windows. Suppose that each element ends up in three successive windows. As the input watermark advances, the desired semantics for sliding windows in this case would be as follows:

The first window completes in the first stage and is emitted downstream.

The first window then completes in the second stage and can also be emitted downstream.

Some time later, the second window completes in the first stage… and so on.

However, if output timestamps are chosen to be the timestamp of the first nonlate element in the pane, what actually happens is the following:

The first window completes in the first stage and is emitted downstream.

The first window in the second stage remains unable to complete because its input watermark is being held up by the output watermark of the second and third windows upstream. Those watermarks are rightly being held back because the earliest element timestamp is being used as the output timestamp for those windows.

The second window completes in the first stage and is emitted downstream.

The first and second windows in the second stage remain unable to complete, held up by the third window upstream.

The third window completes in the first stage and is emitted downstream.

The first, second, and third windows in the second stage are now all able to complete, finally emitting all three in one swoop.

Although the results of this windowing are correct, this leads to the results being materialized in an unnecessarily delayed way. Because of this, Beam has special logic for overlapping windows that ensures the output timestamp for window N+1 is always greater than the end of window N.

Percentile Watermarks
So far, we have concerned ourselves with watermarks as measured by the minimum event time of active messages in a stage. Tracking the minimum allows the system to know when all earlier timestamps have been accounted for. On the other hand, we could consider the entire distribution of event timestamps for active messages and make use of it to create finer-grained triggering conditions.

Instead of considering the minimum point of the distribution, we could take any percentile of the distribution and say that we are guaranteed to have processed this percentage of all events with earlier timestamps.5

What is the advantage of this scheme? If for the business logic “mostly” correct is sufficient, percentile watermarks provide a mechanism by which the watermark can advance more quickly and more smoothly than if we were tracking the minimum event time by discarding outliers in the long tail of the distribution from the watermark. Figure 3-9 shows a compact distribution of event times where the 90th percentile watermark is close to the 100th percentile. Figure 3-10 demonstrates a case where the outlier is further behind, so the 90th percentile watermark is significantly ahead of the 100th percentile. By discarding the outlier data from the watermark, the percentile watermark can still keep track of the bulk of the distribution without being delayed by the outliers.


Figure 3-9. Normal-looking watermark histogram

Figure 3-10. Watermark histogram with outliers
Figure 3-11 shows an example of percentile watermarks used to draw window boundaries for two-minute fixed windows. We can draw early boundaries based on the percentile of timestamps of arrived data as tracked by the percentile watermark.

Figure 3-11. Effects of varying watermark percentiles. As the percentile increases, more events are included in the window: however, the processing time delay to materialize the window also increases.
Figure 3-11 shows the 33rd percentile, 66th percentile, and 100th percentile (full) watermark, tracking the respective timestamp percentiles in the data distribution. As expected, these allow boundaries to be drawn earlier than tracking the full 100th percentile watermark. Notice that the 33rd and 66th percentile watermarks each allow earlier triggering of windows but with the trade-off of marking more data as late. For example, for the first window, [12:00, 12:02), a window closed based on the 33rd percentile watermark would include only four events and materialize the result at 12:06 processing time. If we use the 66th percentile watermark, the same event-time window would include seven events, and materialize at 12:07 processing time. Using the 100th percentile watermark includes all ten events and delays materializing the results until 12:08 processing time. Thus, percentile watermarks provide a way to tune the trade-off between latency of materializing results and precision of the results.

Processing-Time Watermarks
Until now, we have been looking at watermarks as they relate to the data flowing through our system. We have seen how looking at the watermark can help us identify the overall delay between our oldest data and real time. However, this is not enough to distinguish between old data and a delayed system. In other words, by only examining the event-time watermark as we have defined it up until now, we cannot distinguish between a system that is processing data from an hour ago quickly and without delay, and a system that is attempting to process real-time data and has been delayed for an hour while doing so.

To make this distinction, we need something more: processing-time watermarks. We have already seen that there are two time domains in a streaming system: processing time and event time. Until now, we have defined the watermark entirely in the event-time domain, as a function of timestamps of the data flowing through the system. This is an event-time watermark. We will now apply the same model to the processing-time domain to define a processing-time watermark.

Our stream processing system is constantly performing operations such as shuffling messages between stages, reading or writing messages to persistent state, or triggering delayed aggregations based on watermark progress. All of these operations are performed in response to previous operations done at the current or upstream stage of the pipeline. Thus, just as data elements “flow” through the system, a cascade of operations involved in processing these elements also “flows” through the system.

We define the processing-time watermark in the exact same way as we have defined the event-time watermark, except instead of using the event-time timestamp of oldest work not yet completed, we use the processing-time timestamp of the oldest operation not yet completed. An example of delay to the processing-time watermark could be a stuck message delivery from one stage to another, a stuck I/O call to read state or external data, or an exception while processing that prevents processing from completing.

The processing-time watermark, therefore, provides a notion of processing delay separate from the data delay. To understand the value of this distinction, consider the graph in Figure 3-12 where we look at the event-time watermark delay.

We see that the data delay is monotonically increasing, but there is not enough information to distinguish between the cases of a stuck system and stuck data. Only by looking at the processing-time watermark, shown in Figure 3-13, can we distinguish the cases.


Figure 3-12. Event-time watermark increasing. It is not possible to know from this information whether this is due to data buffering or system processing delay.

Figure 3-13. Processing-time watermark also increasing. This indicates that the system processing is delayed.
In the first case (Figure 3-12), when we examine the processing-time watermark delay we see that it too is increasing. This tells us that an operation in our system is stuck, and the stuckness is also causing the data delay to fall behind. Some real-world examples of situations in which this might occur are when there is a network issue preventing message delivery between stages of a pipeline or if a failure has occurred and is being retried. In general, a growing processing-time watermark indicates a problem that is preventing operations from completing that are necessary to the system’s function, and often involves user or administrator intervention to resolve.

In this second case, as seen in Figure 3-14, the processing-time watermark delay is small. This tells us that there are no stuck operations. The event-time watermark delay is still increasing, which indicates that we have some buffered state that we are waiting to drain. This is possible, for example, if we are buffering some state while waiting for a window boundary to emit an aggregation, and corresponds to a normal operation of the pipeline, as in Figure 3-15.


Figure 3-14. Event-time watermark delay increasing, processing-time watermark stable. This is an indication that data are buffered in the system and waiting to be processed, rather than an indication that a system operation is preventing data processing from completing.

Figure 3-15. Watermark delay for fixed windows. The event-time watermark delay increases as elements are buffered for each window, and decreases as each window’s aggregate is emitted via an on-time trigger, whereas the processing-time watermark simply tracks system-level delays (which remain relatively steady in a healthy pipeline).
Therefore, the processing-time watermark is a useful tool in distinguishing system latency from data latency. In addition to visibility, we can use the processing-time watermark at the system-implementation level for tasks such as garbage collection of temporary state (Reuven talks more about an example of this in Chapter 5).

Case Studies
Now that we’ve laid the groundwork for how watermarks ought to behave, it’s time to take a look at some real systems to understand how different mechanisms of the watermark are implemented. We hope that these shed some light on the trade-offs that are possible between latency and correctness as well as scalability and availability for watermarks in real-world systems.

Case Study: Watermarks in Google Cloud Dataflow
There are many possible approaches to implementing watermarks in a stream processing system. Here, we present a quick survey of the implementation in Google Cloud Dataflow, a fully managed service for executing Apache Beam pipelines. Dataflow includes SDKs for defining data processing workflows, and a Cloud Platform managed service to run those workflows on Google Cloud Platform resources.

Dataflow stripes (shards) each of the data processing steps in its data processing graph across multiple physical workers by splitting the available keyspace of each worker into key ranges and assigning each range to a worker. Whenever a GroupByKey operation with distinct keys is encountered, data must be shuffled to corresponding keys.

Figure 3-16 depicts a logical representation of the processing graph with a GroupByKey.


Figure 3-16. A GroupByKey step consumes data from another DoFn. This means that there is a data shuffle between the keys of the first step and the keys of the second step.
Whereas the physical assignment of key ranges to workers might look Figure 3-17.


Figure 3-17. Key ranges of both steps are assigned (striped) across the available workers.
In the watermark propagation section, we discussed that the watermark is maintained for multiple subcomponents of each step. Dataflow keeps track of the per-range watermarks of each of these components. Watermark aggregation then involves computing the minimum of each watermark across all ranges, ensuring that the following guarantees are met:

All ranges must be reporting a watermark. If a watermark is not present for a range, we cannot advance the watermark, because a range not reporting must be treated as unknown.

Ensure that the watermark is monotonically increasing. Because late data is possible, we must not update the watermark if it would cause the watermark to move backward.

Google Cloud Dataflow performs aggregation via a centralized aggregator agent. We can shard this agent for efficiency. From a correctness standpoint, the watermark aggregator serves as a “single source of truth” about the watermark.

Ensuring correctness in distributed watermark aggregation poses certain challenges. It is paramount that watermarks are not advanced prematurely because advancing the watermark prematurely will turn on-time data into late data. Specifically, as physical assignments are actuated to workers, the workers maintain leases on the persistent state attached to the key ranges, ensuring that only a single worker may mutate the persistent state for a key. To guarantee watermark correctness, we must ensure that each watermark update from a worker process is admitted into the aggregate only if the worker process still maintains a lease on its persistent state; therefore, the watermark update protocol must take state ownership lease validation into account.

Case Study: Watermarks in Apache Flink
Apache Flink is an open source stream processing framework for distributed, high-performing, always-available, and accurate data streaming applications. It is possible to run Beam programs using a Flink runner. In doing so, Beam relies on the implementation of stream processing concepts such as watermarks within Flink. Unlike Google Cloud Dataflow, which implements watermark aggregation via a centralized watermark aggregator agent, Flink performs watermark tracking and aggregation in-band.6

To understand how this works, let’s look at a Flink pipeline, as shown in Figure 3-18.


Figure 3-18. A Flink pipeline with two sources and event-time watermarks propagating in-band
In this pipeline data is generated at two sources. These sources also both generate watermark “checkpoints” that are sent synchronously in-band with the data stream. This means that when a watermark checkpoint from source A for timestamp “53” is emitted, it guarantees that no nonlate data messages will be emitted from source A with timestamp behind “53”. The downstream “keyBy” operators consume the input data and the watermark checkpoints. As new watermark checkpoints are consumed, the downstream operators’ view of the watermark is advanced, and a new watermark checkpoint for downstream operators can be emitted.

This choice to send watermark checkpoints in-band with the data stream differs from the Cloud Dataflow approach that relies on central aggregation and leads to a few interesting trade-offs.

Following are some advantages of in-band watermarks:

Reduced watermark propagation latency, and very low-latency watermarks
Because it is not necessary to have watermark data traverse multiple hops and await central aggregation, it is possible to achieve very low latency more easily with the in-band approach.

No single point of failure for watermark aggregation
Unavailability in the central watermark aggregation agent will lead to a delay in watermarks across the entire pipeline. With the in-band approach, unavailability of part of the pipeline cannot cause watermark delay to the entire pipeline.

Inherent scalability
Although Cloud Dataflow scales well in practice, more complexity is needed to achieve scalability with a centralized watermark aggregation service versus implicit scalability with in-band watermarks.

Here are some advantages of out-of-band watermark aggregation:

Single source of “truth”
For debuggability, monitoring, and other applications such as throttling inputs based on pipeline progress, it is advantageous to have a service that can vend the values of watermarks rather than having watermarks implicit in the streams, with each component of the system having its own partial view.

Source watermark creation
Some source watermarks require global information. For example, sources might be temprarily idle, have low data rates, or require out-of-band information about the source or other system components to generate the watermarks. This is easier to achieve in a central service. For an example see the case study that follows on source watermarks for Google Cloud Pub/Sub.

Case Study: Source Watermarks for Google Cloud Pub/Sub
Google Cloud Pub/Sub is a fully managed real-time messaging service that allows you to send and receive messages between independent applications. Here, we discuss how to create a reasonable heuristic watermark for data sent into a pipeline via Cloud Pub/Sub.

First, we need to describe a little about how Pub/Sub works. Messages are published on Pub/Sub topics. A particular topic can be subscribed to by any number of Pub/Sub subscriptions. The same messages are delivered on all subscriptions subscribed to a given topic. The method of delivery is for clients to pull messages off the subscription, and to ack the receipt of particular messages via provided IDs. Clients do not get to choose which messages are pulled, although Pub/Sub does attempt to provide oldest messages first, with no hard guarantees around this.

To build a heuristic, we make some assumptions about the source that is sending data into Pub/Sub. Specifically, we assume that the timestamps of the original data are “well behaved”; in other words, we expect a bounded amount of out-of-order timestamps on the source data, before it is sent to Pub/Sub. Any data that are sent with timestamps outside the allowed out-of-order bounds will be considered late data. In our current implementation, this bound is at least 10 seconds, meaning reordering of timestamps up to 10 seconds before sending to Pub/Sub will not create late data. We call this value the estimation band. Another way to look at this is that when the pipepline is perfectly caught up with the input, the watermark will be 10 seconds behind real time to allow for possible reorderings from the source. If the pipeline is backlogged, all of the backlog (not just the 10-second band) is used for estimating the watermark.

What are the challenges we face with Pub/Sub? Because Pub/Sub does not guarantee ordering, we must have some kind of additional metadata to know enough about the backlog. Luckily, Pub/Sub provides a measurement of backlog in terms of the “oldest unacknowledged publish timestamp.” This is not the same as the event timestamp of our message, because Pub/Sub is agnostic to the application-level metadata being sent through it; instead, this is the timestamp of when the message was ingested by Pub/Sub.

This measurement is not the same as an event-time watermark. It is in fact the processing-time watermark for Pub/Sub message delivery. The Pub/Sub publish timestamps are not equal to the event timestamps, and in the case that historical (past) data are being sent, it might be arbitrarily far away. The ordering on these timestamps might also be different because, as mentioned earlier, we allow a limited amount of reordering.

However, we can use this as a measure of backlog to learn enough information about the event timestamps present in the backlog so that we can create a reasonable watermark as follows.

We create two subscriptions to the topic containing the input messages: a base subscription that the pipeline will actually use to read the data to be processed, and a tracking subscription, which is used for metadata only, to perform the watermark estimation.

Taking a look at our base subscription in Figure 3-19, we see that messages might arrive out of order. We label each message with its Pub/Sub publish timestamp “pt” and its event-time timestamp “et.” Note that the two time domains can be unrelated.


Figure 3-19. Processing-time and event-time timestamps of messages arriving on a Pub/Sub subscription
Some messages on the base subscription are unacknowledged forming a backlog. This might be due to them not yet being delivered or they might have been delivered but not yet processed. Remember also that pulls from this subscription are distributed across multiple shards. Thus, it is not possible to say just by looking at the base subscription what our watermark should be.

The tracking subscription, seen in Figure 3-20, is used to effectively inspect the backlog of the base subscription and take the minimum of the event timestamps in the backlog. By maintaining little or no backlog on the tracking subscription, we can inspect the messages ahead of the base subsciption’s oldest unacknowledged message.


Figure 3-20. An additional “tracking” subscription receiving the same messages as the “base” subscription
We stay caught up on the tracking subscription by ensuring that pulling from this subscription is computationally inexpensive. Conversely, if we fall sufficiently behind on the tracking subscription, we will stop advancing the watermark. To do so, we ensure that at least one of the following conditions is met:

The tracking subscription is sufficiently ahead of the base subscription. Sufficiently ahead means that the tracking subscription is ahead by at least the estimation band. This ensures that any bounded reorder within the estimation band is taken into account.

The tracking subscription is sufficiently close to real time. In other words, there is no backlog on the tracking subscription.

We acknowledge the messages on the tracking subscription as soon as possible, after we have durably saved metadata about the publish and event timestamps of the messages. We store this metadata in a sparse histogram format to minimize the amount of space used and the size of the durable writes.

Finally, we ensure that we have enough data to make a reasonable watermark estimate. We take a band of event timestamps we’ve read from our tracking subscription with publish timestamps newer than the oldest unacknowledged of the base subscription, or the width of the estimation band. This ensures that we consider all event timestamps in the backlog, or if the backlog is small, the most recent estimation band, to make a watermark estimate.

Finally, the watermark value is computed to be the minimum event time in the band.

This method is correct in the sense that all timestamps within the reordering limit of 10 seconds at the input will be accounted for by the watermark and not appear as late data. However, it produces possibly an overly conservative watermark, one that advances “too slowly” in the sense described in Chapter 2. Because we consider all messages ahead of the base subscription’s oldest unacknowledged message on the tracking subscription, we can include event timestamps in the watermark estimate for messages that have already been acknowledged.

Additionally, there are a few heuristics to ensure progress. This method works well in the case of dense, frequently arriving data. In the case of sparse or infrequent data, there might not be enough recent messages to build a reasonable estimate. In the case that we have not seen data on the subscription in more than two minutes (and there’s no backlog), we advance the watermark to near real time. This ensures that the watermark and the pipeline continue to make progress even if no more messages are forthcoming.

All of the above ensures that as long as source data-event timestamp reordering is within the estimation band, there will be no additional late data. 

Summary
At this point, we have explored how we can use the event times of messages to give a robust definition of progress in a stream processing system. We saw how this notion of progress can subsequently help us answer the question of where in event time processing is taking place and when in processing time results are materialized. Specifically, we looked at how watermarks are created at the sources, the points of data ingestion into a pipeline, and then propagated throughout the pipeline to preserve the essential guarantees that allow the questions of where and when to be answered. We also looked at the implications of changing the output window timestamps on watermarks. Finally, we explored some real-world system considerations when building watermarks at scale.

Now that we have a firm footing in how watermarks work under the covers, we can take a dive into what they can do for us as we use windowing and triggering to answer more complex queries in Chapter 4.

1 Note the additional mention of monotonicity; we have not yet discussed how to achieve this. Indeed the discussion thus far makes no mention of monotonicity. If we considered exclusively the oldest in-flight event time, the watermark would not always be monotonic, as we have made no assumptions about our input. We return to this discussion later on.

2 To be precise, it’s not so much that the number of logs need be static as it is that the number of logs at any given time be known a priori by the system. A more sophisticated input source composed of a dynamically chosen number of inputs logs, such as Pravega, could just as well be used for constructing a perfect watermark. It’s only when the number of logs that exist in the dynamic set at any given time is unknown (as in the example in the next section) that one must fall back on a heuristic watermark.

3 Note that by saying “flow through the system,” I don’t necessarily imply they flow along the same path as normal data. They might (as in Apache Flink), but they might also be transmitted out-of-band (as in MillWheel/Cloud Dataflow).

4 The start of the window is not a safe choice from a watermark correctness perspective because the first element in the window often comes after the beginning of the window itself, which means that the watermark is not guaranteed to have been held back as far as the start of the window.

5 The percentile watermark triggering scheme described here is not currently implemented by Beam; however, other systems such as MillWheel implement this.

6 For more information on Flink watermarks, see the Flink documentation on the subject.