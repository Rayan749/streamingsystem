# Chapter 7. The Practicalities of Persistent State
Why do people write books? When you factor out the joy of creativity, a certain fondness for grammar and punctuation, and perhaps the occasional touch of narcissism, you’re basically left with the desire to capture an otherwise ephemeral idea so that it can be revisited in the future. At a very high level, I’ve just motivated and explained persistent state in data processing pipelines.

Persistent state is, quite literally, the tables we just talked about in Chapter 6, with the additional requirement that the tables be robustly stored in a media relatively immune to loss. Stored on local disk counts, as long as you don’t ask your Site Reliability Engineers. Stored on a replicated set of disks is better. Stored on a replicated set of disks in distinct physical locations is better still. Stored in memory once definitely doesn’t count. Stored in replicated memory across multiple machines with UPS power backup and generators onsite maybe does. You get the picture.

In this chapter, our objective is to do the following:

Motivate the need for persistent state within pipelines

Look at two forms of implicit state often found within pipelines

Consider a real-world use case (advertising conversion attribution) that lends itself poorly to implicit state, use that to motivate the salient features of a general, explicit form of persistent state management

Explore a concrete manifestation of one such state API, as found in Apache Beam

Motivation
To begin, let’s more precisely motivate persistent state. We know from Chapter 6 that grouping is what gives us tables. And the core of what I postulated at the beginning of this chapter was correct: the point of persisting these tables is to capture the otherwise ephemeral data contained therein. But why is that necessary?

The Inevitability of Failure
The answer to that question is most clearly seen in the case of processing unbounded input data, so we’ll start there. The main issue is that pipelines processing unbounded data are effectively intended to run forever. But running forever is a far more demanding Service-Level Objective than can be achieved by the environments in which these pipelines typically execute. Long-running pipelines will inevitably see interruptions thanks to machine failures, planned maintenance, code changes, and the occasional misconfigured command that takes down an entire cluster of production pipelines. To ensure that they can resume where they left off when these kinds of things happen, long-running pipelines need some sort of durable recollection of where they were before the interruption. That’s where persistent state comes in.

Let’s expand on that idea a bit beyond unbounded data. Is this only relevant in the unbounded case? Do batch pipelines use persistent state, and why or why not? As with nearly every other batch-versus-streaming question we’ve come across, the answer has less to do with the nature of batch and streaming systems themselves (perhaps unsurprising given what we learned in Chapter 6), and more to do with the types of datasets they historically have been used to process.

Bounded datasets by nature are finite in size. As a result, systems that process bounded data (historically batch systems) have been tailored to that use case. They often assume that the input can be reprocessed in its entirety upon failure. In other words, if some piece of the processing pipeline fails and if the input data are still available, we can simply restart the appropriate piece of the processing pipeline and let it read the same input again. This is called reprocessing the input.

They might also assume failures are infrequent and thus optimize for the common case by persisting as little as possible, accepting the extra cost of recomputation upon failure. For particularly expensive, multistage pipelines, there might be some sort of per-stage global checkpointing that allows for more efficiently resuming execution (typically as part of a shuffle), but it’s not a strict requirement and might not be present in many systems.

Unbounded datasets, on the other hand, must be assumed to have infinite size. As a result, systems that process unbounded data (historically streaming systems) have been built to match. They never assume that all of the data will be available for reprocessing, only some known subset of it. To provide at-least-once or exactly-once semantics, any data that are no longer available for reprocessing must be accounted for in durable checkpoints. And if at-most-once is all you’re going for, you don’t need checkpointing.

At the end of the day, there’s nothing batch- or streaming-specific about persistent state. State can be useful in both circumstances. It just happens to be critical when processing unbounded data, so you’ll find that streaming systems typically provide more sophisticated support for persistent state.

Correctness and Efficiency
Given the inevitability of failures and the need to cope with them, persistent state can be seen as providing two things:

A basis for correctness in light of ephemeral inputs. When processing bounded data, it’s often safe to assume inputs stay around forever;1 with unbounded data, this assumption typically falls short of reality. Persistent state allows you to keep around the intermediate bits of information necessary to allow processing to continue when the inevitable happens, even after your input source has moved on and forgotten about records it gave you previously.

A way to minimize work duplicated and data persisted as part of coping with failures. Regardless of whether your inputs are ephemeral, when your pipeline experiences a machine failure, any work on the failed machine that wasn’t checkpointed somewhere must be redone. Depending upon the nature of the pipeline and its inputs, this can be costly in two dimensions: the amount of work performed during reprocessing, and the amount of input data stored to support reprocessing.

Minimizing duplicated work is relatively straightforward. By checkpointing partial progress within a pipeline (both the intermediate results computed as well as the current location within the input as of checkpointing time), it’s possible to greatly reduce the amount of work repeated when failures occur because none of the operations that came before the checkpoint need to be replayed from durable inputs. Most commonly, this involves data at rest (i.e., tables), which is why we typically refer to persistent state in the context of tables and grouping. But there are persistent forms of streams (e.g., Kafka and its relatives) that serve this function, as well.

Minimizing the amount of data persisted is a larger discussion, one that will consume a sizeable chunk of this chapter. For now, at least, suffice it to say that, for many real-world use cases, rather than remembering all of the raw inputs within a checkpoint for any given stage in the pipeline, it’s often practical to instead remember some partial, intermediate form of the ongoing calculation that consumes less space than all of the original inputs (for example, when computing a mean, the total sum and the count of values seen are much more compact than the complete list of values contributing to that sum and count). Not only can checkpointing these intermediate data drastically reduce the amount of data that you need to remember at any given point in the pipeline, it also commensurately reduces the amount of reprocessing needed for that specific stage to recover from a failure.

Furthermore, by intelligently garbage-collecting those bits of persistent state that are no longer needed (i.e., state for records which are known to have been processed completely by the pipeline already), the amount of data stored in persistent state for a given pipeline can be kept to a manageable size over time, even when the inputs are technically infinite. This is how pipelines processing unbounded data can continue to run effectively forever, while still providing strong consistency guarantees but without a need for complete recall of the original inputs to the pipeline.

At the end of the day, persistent state is really just a means of providing correctness and efficient fault tolerance in data processing pipelines. The amount of support needed in either of those dimensions depends greatly upon the natures of the inputs to the pipeline and the operations being performed. Unbounded inputs tend to require more correctness support than bounded inputs. Computationally expensive operations tend to demand more efficiency support than computationally cheap operations.

Implicit State
Let’s now begin to talk about the practicalities of persistent state. In most cases, this essentially boils down to finding the right balance between always persisting everything (good for consistency, bad for efficiency) and never persisting anything (bad for consistency, good for efficiency). We’ll begin at the always-persisting-everything end of the spectrum, and work our way in the other direction, looking at ways of trading off complexity of implementation for efficiency without compromising consistency (because compromising consistency by never persisting anything is the easy way out for cases in which consistency doesn’t matter, and a nonoption, otherwise). As before, we use the Apache Beam APIs to concretely ground our discussions, but the concepts we discuss are applicable across most systems in existence today.

Also, because there isn’t much you can do to reduce the size of raw inputs, short of perhaps compressing the data, our discussion centers around the ways data are persisted within the intermediate state tables created as part of grouping operations within a pipeline. The inherent nature of grouping multiple records together into some sort of composite will provide us with opportunities to eke out gains in efficiency at the cost of implementation complexity.

Raw Grouping
The first step in our exploration, at the always-persisting-everything end of the spectrum, is the most straightforward implementation of grouping within a pipeline: raw grouping of the inputs. The grouping operation in this case is typically akin to list appending: any time a new element arrives in the group, it’s appended to the list of elements seen for that group.

In Beam, this is exactly what you get when you apply a GroupByKey transform to a PCollection. The stream representing that PCollection in motion is grouped by key to yield a table at rest containing the records from the stream,2 grouped together as lists of values with identical keys. This shows up in the PTransform signature for GroupByKey, which declares the input as a PCollection of K/V pairs, and the output as a collection of K/Iterable<V> pairs:

class GroupByKey<K, V> extends PTransform<
    PCollection<KV<K, V>>, PCollection<KV<K, Iterable<V>>>>>
Every time a trigger fires for a key+window in that table, it will emit a new pane for that key+window, with the value being the Iterable<V> we see in the preceding signature.

Let’s look at an example in action in Example 7-1. We’ll take the summation pipeline from Example 6-5 (the one with fixed windowing and early/on-time/late triggers) and convert it to use raw grouping instead of incremental combination (which we discuss a little later in this chapter). We do this by first applying a GroupByKey transformation to the parsed user/score key/value pairs. The GroupByKey operation performs raw grouping, yielding a PCollection with key/value pairs of users and Iterable<Integer> groups of scores. We then sum up all of the Integers in each iterable by using a simple MapElements lambda that converts the Iterable<Integer> into an IntStream<Integer> and calls sum on it.

Example 7-1. Early, on-time, and late firings via the early/on-time/late API
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> groupedScores = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(
                 AfterWatermark()
                   .withEarlyFirings(AlignedDelay(ONE_MINUTE))
                   .withLateFirings(AfterCount(1))))
  .apply(GroupBy.<String, Integer>create());
PCollection<KV<Team, Integer>> totals = input
  .apply(MapElements.via((KV<String, Iterable<Integer>> kv) ->
    StreamSupport.intStream(
      kv.getValue().spliterator(), false).sum()));
Looking at this pipeline in action, we would see something like that depicted in Figure 7-1.

Figure 7-1. Summation via raw grouping of inputs with windowing and early/on-time/late triggering. The raw inputs are grouped together and stored in the table via the GroupByKey transformation. After being triggered, the MapElements lambda sums the raw inputs within a single pane together to yield per-team scores.
Comparing this to Figure 6-10 (which was using incremental combining, discussed shortly), it’s clear to see this is a lot worse. First, we’re storing a lot more data: instead of a single integer per window, we now store all the inputs for that window. Second, if we have multiple trigger firings, we’re duplicating effort by re-summing inputs we already added together for previous trigger firings. And finally, if the grouping operation is the point at which we checkpoint our state to persistent storage, upon machine failure we again must recompute the sums for any retriggerings of the table. That’s a lot of duplicated data and computation. Far better would be to incrementally compute and checkpoint the actual sums, which is an example of incremental combining.

Incremental Combining
The first step in our journey of trading implementation complexity for efficiency is incremental combining. This concept is manifested in the Beam API via the CombineFn class. In a nutshell, incremental combining is a form of automatic state built upon a user-defined associative and commutative combining operator (if you’re not sure what I mean by these two terms, I define them more precisely in a moment). Though not strictly necessary for the discussion that follows, the important parts of the CombineFn API look like Example 7-2.

Example 7-2. Abbreviated CombineFn API from Apache Beam
class CombineFn<InputT, AccumT, OutputT> {
    // Returns an accumulator representing the empty value.
    AccumT createAccumulator();

    // Adds the given input value into the given accumulator
    AccumT addInput(AccumT accumulator, InputT input);
    
    // Merges the given accumulators into a new, combined accumulator
    AccumT mergeAccumulators(Iterable<AccumT> accumulators);
    
    // Returns the output value for the given accumulator
    OutputT extractOutput(AccumT accumulator);
}
A CombineFn accepts inputs of type InputT, which can be combined together into partial aggregates called accumulators, of type AccumT. These accumulators themselves can also be combined together into new accumulators. And finally, an accumulator can be transformed into an output value of type OutputT. For something like an average, the inputs might be integers, the accumulators pairs of integers (i.e., Pair<sum of inputs, count of inputs>), and the output a single floating-point value representing the mean value of the combined inputs.

But what does all this structure buy us? Conceptually, the basic idea with incremental combining is that many types of aggregations (sum, mean, etc.) exhibit the following properties:

Incremental aggregations possess an intermediate form that captures the partial progress of combining a set of N inputs more compactly than the full list of those inputs themselves (i.e., the AccumT type in CombineFn). As discussed earlier, for mean, this is a sum/count pair. Basic summation is even simpler, with a single number as its accumulator. A histogram would have a relatively complex accumulator composed of buckets, where each bucket contains a count for the number of values seen within some specific range. In all three cases, however, the amount of space consumed by an accumulator that represents the aggregation of N elements remains significantly smaller than the amount of space consumed by the original N elements themselves, particularly as the size of N grows.

Incremental aggregations are indifferent to ordering across two dimensions:

Individual elements, meaning:

COMBINE(a, b) == COMBINE(b, a)

Groupings of elements, meaning:

COMBINE(COMBINE(a, b), c) == COMBINE(a, COMBINE(b, c))

These properties are known as commutativity and associativity, respectively. In concert,3 they effectively mean that we are free to combine elements and partial aggregates in any arbitrary order and with any arbitrary subgrouping. This allows us to optimize the aggregation in two ways:

Incrementalization
Because the order of individual inputs doesn’t matter, we don’t need to buffer all of the inputs ahead of time and then process them in some strict order (e.g., in order of event time; note, however, that this remains independent of shuffling elements by event time into proper event-time windows before aggregating); we can simply combine them one-by-one as they arrive. This not only greatly reduces the amount of data that must be buffered (thanks to the first property of our operation, which stated the intermediate form was a more compact representation of partial aggregation than the raw inputs themselves), but also spreads the computation load more evenly over time (versus aggregating a burst of inputs all at once after the full input set has been buffered).

Parallelization
Because the order in which partial subgroups of inputs are combined doesn’t matter, we’re free to arbitrarily distribute the computation of those subgroups. More specifically, we’re free to spread the computation of those subgroups across a plurality of machines. This optimization is at the heart of MapReduce’s Combiners (the genesis of Beam’s CombineFn).

MapReduce’s Combiner optimization is essential to solving the hot-key problem, where some sort of grouping computation is performed on an input stream that is too large to be reasonably processed by a single physical machine. A canonical example is breaking down high-volume analytics data (e.g., web traffic to a popular website) across a relatively low number of dimensions (e.g., by web browser family: Chrome, Firefox, Safari, etc.). For websites with a particularly high volume of traffic, it’s often intractable to calculate stats for any single web browser family on a single machine, even if that’s the only thing that machine is dedicated to doing; there’s simply too much traffic to keep up with. But with an associative and commutative operation like summation, it’s possible to spread the initial aggregation across multiple machines, each of which computes a partial aggregate. The set of partial aggregates generated by those machines (whose size is now many of orders magnitude smaller than the original inputs) might then be further combined together on a single machine to yield the final aggregate result.

As an aside, this ability to parallelize also yields one additional benefit: the aggregation operation is naturally compatible with merging windows. When two windows merge, their values must somehow be merged, as well. With raw grouping, this means merging the two full lists of buffered values together, which has a cost of O(N). But with a CombineFn, it’s a simple combination of two partial aggregates, typically an O(1) operation.

For the sake of completeness, consider again Example 6-5, shown in Example 7-3, which implements a summation pipeline using incremental combination.

Example 7-3. Grouping and summation via incremental combination, as in Example 6-5
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(
                 AfterWatermark()
                   .withEarlyFirings(AlignedDelay(ONE_MINUTE))
                   .withLateFirings(AfterCount(1))))
  .apply(Sum.integersPerKey());
When executed, we get what we saw Figure 6-10 (shown here in Figure 7-2). Compared to Figure 7-1, this is clearly a big improvement, with much greater efficiency in terms of amount of data stored and amount of computation performed.

Figure 7-2. Grouping and summation via incremental combination. In this version, incremental sums are computed and stored in the table rather than lists of inputs, which must later be summed together independently.
By providing a more compact intermediate representation for a grouping operation, and by relaxing requirements on ordering (both at the element and subgroup levels), Beam’s CombineFn trades off a certain amount of implementation complexity in exchange for increases in efficiency. In doing so, it provides a clean solution for the hot-key problem and also plays nicely with the concept of merging windows.

One shortcoming, however, is that your grouping operation must fit within a relatively restricted structure. This is all well and good for sums, means, and so on, but there are plenty of real-world use cases in which a more general approach, one which allows precise control over trade-offs of complexity and efficiency, is needed. We’ll look next at what such a general approach entails.

Generalized State
Though both of the implicit approaches we’ve looked at so far have their merits, they each fall short in one dimension: flexibility. The raw grouping method requires you to always buffer up the raw inputs to the grouping operation before processing the group in whole, so there’s no way to partially process some of the data along the way; it’s all or nothing. The incremental combining approach specifically allows for partial processing but with the restriction that the processing in question be commutative and associative and happen as records arrive one-by-one.

If we want to support a more generalized approach to streaming persistent state, we need something more flexible. Specifically, we need flexibility in three dimensions:

Flexibility in data structures; that is, an ability to structure the data we write and read in ways that are most appropriate and efficient for the task at hand. Raw grouping essentially provides an appendable list, and incremental combination essentially provides a single value that is always written and read in its entirety. But there are myriad other ways in which we might want to structure our persistent data, each with different types of access patterns and associated costs: maps, trees, graphs, sets, and so on. Supporting a variety of persistent data types is critical for efficiency.

Beam supports flexibility in data types by allowing a single DoFn to declare multiple state fields, each of a specific type. In this way, logically independent pieces of state (e.g., visits and impressions) can be stored separately, and semantically different types of state (e.g., maps and lists) can be accessed in ways that are natural given their types of access patterns.

Flexibility in write and read granularity; that is, an ability to tailor the amount and type of data written or read at any given time for optimal efficiency. What this boils down to is the ability to write and read precisely the necessary amount of data at any given point of time: no more, and no less (and in parallel as much as possible).

This goes hand in hand with the previous point, given that dedicated data types allow for focused types of access patterns (e.g., a set-membership operation that can use something like a Bloom filter under the covers to greatly minimize the amount of data read in certain circumstances). But it goes beyond it, as well; for example, allowing multiple large reads to be dispatched in parallel (e.g., via futures).

In Beam, flexibly granular writes and reads are enabled via datatype-specific APIs that provide fine-grained access capabilities, combined with an asynchronous I/O mechanism that allows for writes and reads to be batched together for efficiency.

Flexibility in scheduling of processing; that is, an ability to bind the time at which specific types of processing occur to the progress of time in either of the two time domains we care about: event-time completeness and processing time. Triggers provide a restricted set of flexibility here, with completeness triggers providing a way to bind processing to the watermark passing the end of the window, and repeated update triggers providing a way to bind processing to periodic progress in the processing-time domain. But for certain use cases (e.g., certain types of joins, for which you don’t necessarily care about input completeness of the entire window, just input completeness up to the event-time of a specific record in the join), triggers are insufficiently flexible. Hence, our need for a more general solution.

In Beam, flexible scheduling of processing is provided via timers.4 A timer is a special type of state that binds a specific point in time in either supported time domain (event time or processing time) with a method to be called when that point in time is reached. In this way, specific bits of processing can be delayed until a more appropriate time in the future.

The common thread among these three characteristics is flexibility. A specific subset of use cases are served very well by the relatively inflexible approaches of raw grouping or incremental combination. But when tackling anything outside their relatively narrow domain of expertise, those options often fall short. When that happens, you need the power and flexibility of a fully general-state API to let you tailor your utilization of persistent state optimally.

To think of it another way, raw grouping and incremental combination are relatively high-level abstractions that enable the pithy expression of pipelines with (in the case of combiners, at least) some good properties for automatic optimizations. But sometimes you need to go low level to get the behavior or performance you need. That’s what generalized state lets you do.

Case Study: Conversion Attribution
To see this in action, let’s now look at a use case that is poorly served by both raw grouping and incremental combination: conversion attribution. This is a technique that sees widespread use across the advertising world to provide concrete feedback on the effectiveness of advertisements. Though relatively easy to understand, its somewhat diverse set of requirements doesn’t fit nicely into either of the two types of implicit state we’ve considered so far.

Imagine that you have an analytics pipeline that monitors traffic to a website in conjunction with advertisement impressions that directed traffic to that site. The goal is to provide attribution of specific advertisements shown to a user toward the achievement of some goal on the site itself (which often might lie many steps beyond the initial advertisement landing page), such as signing up for a mailing list or purchasing an item.

Figure 7-3 shows an example set of website visits, goals, and ad impressions, with one attributed conversion highlighted in red. Building up conversion attributions over an unbounded, out-of-order stream of data requires keeping track of impressions, visits, and goals seen so far. That’s where persistent state comes in.


Figure 7-3. Example conversion attribution
In this diagram, a user’s traversal of various pages on a website is represented as a graph. Impressions are advertisements that were shown to the user and clicked, resulting in the user visiting a page on the site. Visits represent a single page viewed on the site. Goals are specific visited pages that have been identified as a desired destination for users (e.g., completing a purchase, or signing up for a mailing list). The goal of conversion attribution is to identify ad impressions that resulted in the user achieving some goal on the site. In this figure, there is one such conversion highlighted in red. Note that events might arrive out of order, hence the event-time axis in the diagram and the watermark reference point indicating the time up to which input is believed to be correct.

A lot goes into building a robust, large-scale attribution pipeline, but there are a few aspects worth calling out explicitly. Any such pipeline we attempt to build must do the following:

Handle out-of-order data
Because the website traffic and ad impression data come from separate systems, both of which are implemented as distributed collection services themselves, the data might arrive wildly out of order. Thus, our pipeline must be resilient to such disorder.

Handle high volumes of data
Not only must we assume that this pipeline will be processing data for a large number of independent users, but depending upon the volume of a given ad campaign and the popularity of a given website, we might need to store a large amount of impression and/or traffic data as we attempt to build evidence of attribution. For example, it would not be unheard of to store 90 days worth of visit, impression, and goal tree5 data per user to allow us to build up attributions that span multiple months’ worth of activity.

Protect against spam
Given that money is involved, correctness is paramount. Not only must we ensure that visits and impressions are accounted for exactly once (something we’ll get more or less for free by simply using an execution engine that supports effectively-once processing), but we must also guard our advertisers against spam attacks that attempt to charge advertisers unfairly. For example, a single ad that is clicked multiple times in a row by the same user will arrive as multiple impressions, but as long as those clicks occur within a certain amount of time of one another (e.g., within the same day), they must be attributed only once. In other words, even if the system guarantees we’ll see every individual impression once, we must also perform some manual deduplication across impressions that are technically different events but which our business logic dictates we interpret as duplicates.

Optimize for performance
Above all, because of the potential scale of this pipeline, we must always keep an eye toward optimizing the performance of our pipeline. Persistent state, because of the inherent costs of writing to persistent storage, can often be the performance bottleneck in such a pipeline. As such, the flexibility characteristics we discussed earlier will be critical in ensuring our design is as performant as possible.

Conversion Attribution with Apache Beam
Now that we understand the basic problem that we’re trying to solve and have some of the important requirements squarely in mind, let’s use Beam’s State and Timers API to build a basic conversion attribution transformation. We’ll write this just like we would any other DoFn in Beam, but we’ll make use of state and timer extensions that allow us to write and read persistent state and timer fields. Those of you that want to follow along in real code can find the full implementation on GitHub.

Note that, as with all grouping operations in Beam, usage of the State API is scoped to the current key and window, with window lifetimes dictated by the specified allowed lateness parameter; in this example, we’ll be operating within a single global window. Parallelism is linearized per key, as with most DoFns. Also note that, for simplicity, we’ll be eliding the manual garbage collection of visits and impressions falling outside of our 90-day horizon that would be necessary to keep the persisted state from growing forever.

To begin, let’s define a few POJO classes for visits, impressions, a visit/impression union (used for joining), and completed attributions, as shown in Example 7-4.

Example 7-4. POJO definitions of Visit, Impression, VisitOrImpression, and Attribution objects
@DefaultCoder(AvroCoder.class)
class Visit {
    @Nullable private String url;
    @Nullable private Instant timestamp;
    // The referring URL. Recall that we’ve constrained the problem in this
    // example to assume every page on our website has exactly one possible
    // referring URL, to allow us to solve the problem for simple trees
    // rather than more general DAGs.
    @Nullable private String referer;
    @Nullable private boolean isGoal;

    @SuppressWarnings("unused")
    public Visit() {
    }
    
    public Visit(String url, Instant timestamp, String referer,
                 boolean isGoal) {
    this.url = url;
    this.timestamp = timestamp;
    this.referer = referer;
    this.isGoal = isGoal;
    }
    
    public String url() { return url; }
    public Instant timestamp() { return timestamp; }
    public String referer() { return referer; }
    public boolean isGoal() { return isGoal; }
    
    @Override
    public String toString() {
        return String.format("{ %s %s from:%s%s }", url, timestamp, referer,
                             isGoal ? " isGoal" : "");
    }
}

@DefaultCoder(AvroCoder.class)
class Impression {
    @Nullable private Long id;
    @Nullable private String sourceUrl;
    @Nullable private String targetUrl;
    @Nullable private Instant timestamp;

    public static String sourceAndTarget(String source, String target) { 
        return source + ":" + target;
    }
    
    @SuppressWarnings("unused")
    public Impression() {
    }
    
    public Impression(Long id, String sourceUrl, String targetUrl,
                      Instant timestamp) {
        this.id = id;
    this.sourceUrl = sourceUrl;
    this.targetUrl = targetUrl;
    this.timestamp = timestamp;
    }
    
    public Long id() { return id; }
    public String sourceUrl() { return sourceUrl; }
    public String targetUrl() { return targetUrl; }
    public String sourceAndTarget() {
        return sourceAndTarget(sourceUrl, targetUrl);
    }
    public Instant timestamp() { return timestamp; }
    
    @Override
    public String toString() {
    return String.format("{ %s source:%s target:%s %s }",
                             id, sourceUrl, targetUrl, timestamp);
    }
}

@DefaultCoder(AvroCoder.class)
class VisitOrImpression {
    @Nullable private Visit visit;
    @Nullable private Impression impression;

    @SuppressWarnings("unused")
    public VisitOrImpression() {
    }
    
    public VisitOrImpression(Visit visit, Impression impression) {
    this.visit = visit;
    this.impression = impression;
    }
    
    public Visit visit() { return visit; }
    public Impression impression() { return impression; }
}

@DefaultCoder(AvroCoder.class)
class Attribution {
    @Nullable private Impression impression;
    @Nullable private List<Visit> trail;
    @Nullable private Visit goal;

    @SuppressWarnings("unused")
    public Attribution() {
    }
    
    public Attribution(Impression impression, List<Visit> trail, Visit goal) {
    this.impression = impression;
    this.trail = trail;
    this.goal = goal;
    }
    
    public Impression impression() { return impression; }
    public List<Visit> trail() { return trail; }
    public Visit goal() { return goal; }
    
    @Override
    public String toString() {
    StringBuilder builder = new StringBuilder();
    builder.append("imp=" + impression.id() + " " + impression.sourceUrl());
    for (Visit visit : trail) {
        builder.append(" → " + visit.url());
    }
    builder.append(" → " + goal.url());
    return builder.toString();
    }
}
We next define a Beam DoFn to consume a flattened collection of Visits and Impressions, keyed by the user. In turn, it will yield a collection of Attributions. Its signature looks like Example 7-5.

Example 7-5. DoFn signature for our conversion attribution transformation
class AttributionFn extends DoFn<KV<String, VisitOrImpression>, Attribution>
Within that DoFn, we need to implement the following logic:

Store all visits in a map keyed by their URL so that we can easily look them up when tracing visit trails backward from a goal.

Store all impressions in a map keyed by the URL they referred to, so we can identify impressions that initiated a trail to a goal.

Any time we see a visit that happens to be a goal, set an event-time timer for the timestamp of the goal. Associated with this timer will be a method that performs goal attribution for the pending goal. This will ensure that attribution only happens once the input leading up to the goal is complete.

Because Beam lacks support for a dynamic set of timers (currently all timers must be declared at pipeline definition time, though each individual timer can be set and reset for different points in time at runtime), we also need to keep track of the timestamps for all of the goals we still need to attribute. This will allow us to have a single attribution timer set for the minimum timestamp of all pending goals. After we attribute the goal with the earliest timestamp, we set the timer again with the timestamp of the next earliest goal.

Let’s now walk through the implementation in pieces. First up, we need to declare specifications for all of our state and timer fields within the DoFn. For state, the specification dictates the type of data structure for the field itself (e.g., map or list) as well as the type(s) of data contained therein, and their associated coder(s); for timers, it dictates the associated time domain. Each specification is then assigned a unique ID string (via the @StateID/@TimerId annotations), which will allow us to dynamically associate these specifications with parameters and methods later on. For our use case, we’ll define (in Example 7-6) the following:

Two MapState specifications for visits and impressions

A single SetState specification for goals

A ValueState specification for keeping track of the minimum pending goal timestamp

A Timer specification for our delayed attribution logic

Example 7-6. State field specifications
class AttributionFn extends DoFn<KV<String, VisitOrImpression>, Attribution> {
    @StateId("visits")
    private final StateSpec<MapState<String, Visit>> visitsSpec =
	StateSpecs.map(StringUtf8Coder.of(), AvroCoder.of(Visit.class));

    // Impressions are keyed by both sourceUrl (i.e., the query) and targetUrl
    // (i.e., the click), since a single query can result in multiple impressions.
    // The source and target are encoded together into a single string by the
    // Impression.sourceAndTarget method.
    @StateId("impressions")
    private final StateSpec<MapState<String, Impression>> impSpec =
    StateSpecs.map(StringUtf8Coder.of(), AvroCoder.of(Impression.class));
    
    @StateId("goals")
    private final StateSpec<SetState<Visit>> goalsSpec =
    StateSpecs.set(AvroCoder.of(Visit.class));
    
    @StateId("minGoal")
    private final StateSpec<ValueState<Instant>> minGoalSpec =
    StateSpecs.value(InstantCoder.of());
    
    @TimerId("attribution")
    private final TimerSpec timerSpec =
    TimerSpecs.timer(TimeDomain.EVENT_TIME);

... continued in Example 7-7 below ...
Next up, we implement our core @ProcessElement method. This is the processing logic that will run every time a new record arrives. As noted earlier, we need to record visits and impressions to persistent state as well as keep track of goals and manage the timer that will bind our attribution logic to the progress of event-time completeness as tracked by the watermark. Access to state and timers is provided via parameters passed to our @ProcessElement method, and the Beam runtime invokes our method with appropriate parameters indicated by @StateId and @TimerId annotations. The logic itself is then relatively straightforward, as demonstrated in Example 7-7.

Example 7-7. @ProcessElement implementation
... continued from Example 7-6 above ...

@ProcessElement
public void processElement(
        @Element KV<String, VisitOrImpression> kv,
	@StateId("visits") MapState<String, Visit> visitsState,
	@StateId("impressions") MapState<String, Impression> impressionsState,
	@StateId("goals") SetState<Visit> goalsState,
	@StateId("minGoal") ValueState<Instant> minGoalState,
	@TimerId("attribution") Timer attributionTimer) {
    Visit visit = kv.getValue().visit();
    Impression impression = kv.getValue().impression();

    if (visit != null) {
    if (!visit.isGoal()) {
        LOG.info("Adding visit: {}", visit);
        visitsState.put(visit.url(), visit);
    } else {
        LOG.info("Adding goal (if absent): {}", visit);
        goalsState.addIfAbsent(visit);
        Instant minTimestamp = minGoalState.read();
        if (minTimestamp == null || visit.timestamp().isBefore(minTimestamp)) {
                LOG.info("Setting timer from {} to {}",
                         Utils.formatTime(minTimestamp),
                         Utils.formatTime(visit.timestamp()));
                attributionTimer.set(visit.timestamp());
    	minGoalState.write(visit.timestamp());
        }
        LOG.info("Done with goal");
    }
    }
    if (impression != null) {
        // Dedup logical impression duplicates with the same source and target URL.
    // In this case, first one to arrive (in processing time) wins. A more
    // robust approach might be to pick the first one in event time, but that
        // would require an extra read before commit, so the processing-time
        // approach may be slightly more performant.
        LOG.info("Adding impression (if absent): {} → {}",
                 impression.sourceAndTarget(), impression);
    impressionsState.putIfAbsent(impression.sourceAndTarget(), impression);
    }
}

... continued in Example 7-8 below ...
Note how this ties back to our three desired capabilities in a general state API:

Flexibility in data structures
We have maps, a set, a value, and a timer. They allow us to efficiently manipulate our state in ways that are effective for our algorithm.

Flexibility in write and read granularity
Our @ProcessElement method is called for every single visit and impression we process. As such, we need it to be as efficient as possible. We take advantage of the ability to make fine-grained, blind writes only to the specific fields we need. We also only ever read from state within our @ProcessElement method in the uncommon case of encountering a new goal. And when we do, we read only a single integer value, without touching the (potentially much larger) maps and list.

Flexibility in scheduling of processing
Thanks to timers, we’re able to delay our complex goal attribution logic (defined next) until we’re confident we’ve received all the necessary input data, minimizing duplicated work and maximizing efficiency.

Having defined the core processing logic, let’s now look at our final piece of code, the goal attribution method. This method is annotated with an @TimerId annotation to identify it as the code to execute when the corresponding attribution timer fires. The logic here is significantly more complicated than the @ProcessElement method:

First, we need to load the entirety of our visit and impression maps, as well as our set of goals. We need the maps to piece our way backward through the attribution trail we’ll be building, and we need the goals to know which goals we’re attributing as a result of the current timer firing, as well as the next pending goal we want to schedule for attribution in the future (if any).

After we’ve loaded our state, we process goals for this timer one at a time in a loop, repeatedly:

Checking to see if any impressions referred the user to the current visit in the trail (beginning with the goal). If so, we’ve completed attribution of this goal and can break out of the loop and emit the attribution trail.

Checking next to see if any visits were the referrer for the current visit. If so, we’ve found a back pointer in our trail, so we traverse it and start the loop over.

If no matching impressions or visits are found, we have a goal that was reached organically, with no associated impression. In this case, we simply break out of the loop and move on to the next goal, if any.

After we’ve exhausted our list of goals ready for attribution, we set a timer for the next pending goal in the list (if any) and reset the corresponding ValueState tracking the minimum pending goal timestamp.

To keep things concise, we first look at the core goal attribution logic, shown in Example 7-8, which roughly corresponds to point 2 in the preceding list.

Example 7-8. Goal attribution logic
... continued from Example 7-7 above ...

private Impression attributeGoal(Visit goal,
				 Map<String, Visit> visits,
				 Map<String, Impression> impressions,
				 List<Visit> trail) {
    Impression impression = null;
    Visit visit = goal;
    while (true) {
        String sourceAndTarget = Impression.sourceAndTarget(
            visit.referer(), visit.url());
        LOG.info("attributeGoal: visit={} sourceAndTarget={}",
                 visit, sourceAndTarget);
	if (impressions.containsKey(sourceAndTarget)) {
	    LOG.info("attributeGoal: impression={}", impression);
	    // Walked entire path back to impression. Return success.
	    return impressions.get(sourceAndTarget);
	} else if (visits.containsKey(visit.referer())) {
	    // Found another visit in the path, continue searching.
	    visit = visits.get(visit.referer());
	    trail.add(0, visit);
	} else {
	    LOG.info("attributeGoal: not found");
	    // Referer not found, trail has gone cold. Return failure.
	    return null;
	}
    }
}

... continued in Example 7-9 below ...
The rest of the code (eliding a few simple helper methods), which handles initializing and fetching state, invoking the attribution logic, and handling cleanup to schedule any remaining pending goal attribution attempts, looks like Example 7-9.

Example 7-9. Overall @TimerId handling logic for goal attribution
... continued from Example 7-8 above ...

@OnTimer("attribution")
public void attributeGoal(
        @Timestamp Instant timestamp,
	@StateId("visits") MapState<String, Visit> visitsState,
	@StateId("impressions") MapState<String, Impression> impressionsState,
	@StateId("goals") SetState<Visit> goalsState,
	@StateId("minGoal") ValueState<Instant> minGoalState,
	@TimerId("attribution") Timer attributionTimer,
	OutputReceiver<Attribution> output) {
    LOG.info("Processing timer: {}", Utils.formatTime(timestamp));

    // Batch state reads together via futures.
    ReadableState<Iterable<Map.Entry<String, Visit> > > visitsFuture
        = visitsState.entries().readLater();
    ReadableState<Iterable<Map.Entry<String, Impression> > > impressionsFuture
        = impressionsState.entries().readLater();
    ReadableState<Iterable<Visit>> goalsFuture = goalsState.readLater();
    
    // Accessed the fetched state.
    Map<String, Visit> visits = buildMap(visitsFuture.read());
    Map<String, Impression> impressions = buildMap(impressionsFuture.read());
    Iterable<Visit> goals = goalsFuture.read();
    
    // Find the matching goal
    Visit goal = findGoal(timestamp, goals);
    
    // Attribute the goal
    List<Visit> trail = new ArrayList<>();
    Impression impression = attributeGoal(goal, visits, impressions, trail);
    if (impression != null) {
    output.output(new Attribution(impression, trail, goal));
    impressions.remove(impression.sourceAndTarget());
    }
    goalsState.remove(goal);
    
    // Set the next timer, if any.
    Instant minGoal = minTimestamp(goals, goal);
    if (minGoal != null) {
    LOG.info("Setting new timer at {}", Utils.formatTime(minGoal));
    minGoalState.write(minGoal);
    attributionTimer.set(minGoal);
    } else {
    minGoalState.clear();
    }
}
This code block ties back to the three desired capabilities of a general state API in very similar ways as the @ProcessElement method, with one noteworthy difference:

Flexibility in write and read granularity
We were able to make a single, coarse-grained read up front to load all of the data in the maps and set. This is typically much more efficient than loading each field separately, or even worse loading each field element by element. It also shows the importance of being able to traverse the spectrum of access granularities, from fine-grained to coarse-grained.

And that’s it! We’ve implemented a basic conversion attribution pipeline, in a way that’s efficient enough to be operated at respectable scales using a reasonable amount of resources. And importantly, it functions properly in the face of out-of-order data. If you look at the dataset used for the unit test in Example 7-10, you can see it presents a number of challenges, even at this small scale:

Tracking and attributing multiple distinct conversions across a shared set of URLs.

Data arriving out of order, and in particular, goals arriving (in processing time) before visits and impressions that lead to them, as well as other goals which occurred earlier.

Source URLs that generate multiple distinct impressions to different target URLs.

Physically distinct impressions (e.g., multiple clicks on the same advertisement) that must be deduplicated to a single logical impression.

Example 7-10. Example dataset for validating conversion attribution logic
private static TestStream<KV<String, VisitOrImpression>> createStream() {
    // Impressions and visits, in event-time order, for two (logical) attributable
    // impressions and one unattributable impression.
    Impression signupImpression = new Impression(
	123L, "http://search.com?q=xyz",
	"http://xyz.com/", Utils.parseTime("12:01:00"));
    Visit signupVisit = new Visit(
	"http://xyz.com/", Utils.parseTime("12:01:10"),
	"http://search.com?q=xyz", false/*isGoal*/);
    Visit signupGoal = new Visit(
	"http://xyz.com/join-mailing-list", Utils.parseTime("12:01:30"),
	"http://xyz.com/", true/*isGoal*/);

    Impression shoppingImpression = new Impression(
    456L, "http://search.com?q=thing",
    "http://xyz.com/thing", Utils.parseTime("12:02:00"));
    Impression shoppingImpressionDup = new Impression(
    789L, "http://search.com?q=thing",
    "http://xyz.com/thing", Utils.parseTime("12:02:10"));
    Visit shoppingVisit1 = new Visit(
    "http://xyz.com/thing", Utils.parseTime("12:02:30"),
    "http://search.com?q=thing", false/*isGoal*/);
    Visit shoppingVisit2 = new Visit(
    "http://xyz.com/thing/add-to-cart", Utils.parseTime("12:03:00"),
    "http://xyz.com/thing", false/*isGoal*/);
    Visit shoppingVisit3 = new Visit(
    "http://xyz.com/thing/purchase", Utils.parseTime("12:03:20"),
    "http://xyz.com/thing/add-to-cart", false/*isGoal*/);
    Visit shoppingGoal = new Visit(
    "http://xyz.com/thing/receipt", Utils.parseTime("12:03:45"),
    "http://xyz.com/thing/purchase", true/*isGoal*/);
    
    Impression unattributedImpression = new Impression(
    000L, "http://search.com?q=thing",
    "http://xyz.com/other-thing", Utils.parseTime("12:04:00"));
    Visit unattributedVisit = new Visit(
    "http://xyz.com/other-thing", Utils.parseTime("12:04:20"),
    "http://search.com?q=other thing", false/*isGoal*/);
    
    // Create a stream of visits and impressions, with data arriving out of order.
    return TestStream.create(
    KvCoder.of(StringUtf8Coder.of(), AvroCoder.of(VisitOrImpression.class)))
    .advanceWatermarkTo(Utils.parseTime("12:00:00"))
    .addElements(visitOrImpression(shoppingVisit2, null))
    .addElements(visitOrImpression(shoppingGoal, null))
    .addElements(visitOrImpression(shoppingVisit3, null))
    .addElements(visitOrImpression(signupGoal, null))
    .advanceWatermarkTo(Utils.parseTime("12:00:30"))
    .addElements(visitOrImpression(null, signupImpression))
    .advanceWatermarkTo(Utils.parseTime("12:01:00"))
    .addElements(visitOrImpression(null, shoppingImpression))
    .addElements(visitOrImpression(signupVisit, null))
    .advanceWatermarkTo(Utils.parseTime("12:01:30"))
    .addElements(visitOrImpression(null, shoppingImpressionDup))
    .addElements(visitOrImpression(shoppingVisit1, null))
    .advanceWatermarkTo(Utils.parseTime("12:03:45"))
    .addElements(visitOrImpression(null, unattributedImpression))
    .advanceWatermarkTo(Utils.parseTime("12:04:00"))
    .addElements(visitOrImpression(unattributedVisit, null))
    .advanceWatermarkToInfinity();
}
And remember, we’re working here on a relatively constrained version of conversion attribution. A full-blown impelementation would have additional challenges to deal with (e.g., garbage collection, DAGs of visits instead of trees). Regardless, this pipeline provides a nice contrast to the oftentimes insufficiently flexible approaches provided by raw grouping an incremental combination. By trading off some amount of implementation complexity, we were able to find the necessary balance of efficiency, without compromising on correctness. Additionally, this pipeline highlights the more imperative approach towards stream processing that state and timers afford (think C or Java), which is a nice complement to the more functional approach afforded by windowing and triggers (think Haskell).

Summary
In this chapter, we’ve looked closely at why persistent state is important, coming to the conclusion that it provides a basis for correctness and efficiency in long-lived pipelines. We then looked at the two most common types of implicit state encountered in data processing systems: raw grouping and incremental combination. We learned that raw grouping is straightforward but potentially inefficient and that incremental combination greatly improves efficiency for operations that are commutative and associative. Finally, we looked a relatively complex, but very practical use case (and implementation via Apache Beam Java) grounded in real-world experience, and used that to highlight the important characteristics needed in a general state abstraction:

Flexibility in data structures, allowing for the use of data types tailored to specific use cases at hand.

Flexibility in write and read granularity, allowing the amount of data written and read at any point to be tailored to the use case, minimizing or maximizing I/O as appropriate.

Flexibility in scheduling of processing, allowing certain portions of processing to be delayed until a more appropriate point in time, such as when the input is believed to be complete up to a specific point in event time.

1 For some definition of “forever,” typically at least “until we successfully complete execution of our batch pipeline and no longer require the inputs.”

2 Recall that Beam doesn’t currently expose these state tables directly; you must trigger them back into a stream to observe their contents as a new PCollection.

3 Or, as my colleague Kenn Knowles points out, if you take the definition as being commutativity across sets, the three-parameter version of commutativity is actually sufficient to also imply associativity: COMBINE(a, b, c) == COMBINE(a, c, b) == COMBINE(b, a, c) == COMBINE(b, c, a) == COMBINE(c, a, b) == COMBINE(c, b, a). Math is fun.

4 And indeed, timers are the underlying feature used to implement most of the completeness and repeated updated triggers we discussed in Chapter 2 as well as garbage collection based on allowed lateness.

5 Thanks to the nature of web browsing, the visit trails we’ll be analyzing are trees of URLs linked by HTTP referrer fields. In reality, they would end up being directed graphs, but for the sake of simplicity, we’ll assume each page on our website has incoming links from exactly one other referring page on the site, thus yielding a simpler tree structure. Generalizing to graphs is a natural extension of the tree-based implementation, and only further drives home the points being made.