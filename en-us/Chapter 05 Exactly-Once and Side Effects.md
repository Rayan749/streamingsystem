# Chapter 5. Exactly-Once and Side Effects

We now shift from discussing programming models and APIs to the systems that implement them. A model and API allows users to describe what they want to compute. Actually running the computation accurately at scale requires a system—usually a distributed system.

In this chapter, we focus on how an implementing system can correctly implement the Beam Model to produce accurate results. Streaming systems often talk about exactly-once processing; that is, ensuring that every record is processed exactly one time. We will explain what we mean by this, and how it might be implemented.

As a motivating example, this chapter focuses on techniques used by Google Cloud Dataflow to efficiently guarantee exactly-once processing of records. Toward the end of the chapter, we also look at techniques used by some other popular streaming systems to guarantee exactly once.

Why Exactly Once Matters
It almost goes without saying that for many users, any risk of dropped records or data loss in their data processing pipelines is unacceptable. Even so, historically many general-purpose streaming systems made no guarantees about record processing—all processing was “best effort” only. Other systems provided at-least-once guarantees, ensuring that records were always processed at least once, but records might be duplicated (and thus result in inaccurate aggregations); in practice, many such at-least-once systems performed aggregations in memory, and thus their aggregations could still be lost when machines crashed. These systems were used for low-latency, speculative results but generally could guarantee nothing about the veracity of these results.

As Chapter 1 points out, this led to a strategy that was coined the Lambda Architecture—run a streaming system to get fast, but inaccurate results. Sometime later (often after end of day), a batch system runs to the correct answer. This works only if the data stream is replayable; however, this was true for enough data sources that this strategy proved viable. Nonetheless, many people who tried this experienced a number of issues with the Lambda Architecture:

Inaccuracy
Users tend to underestimate the impact of failures. They often assume that a small percentage of records will be lost or duplicated (often based on experiments they ran), and are shocked on that one bad day when 10% (or more!) of records are lost or are duplicated. In a sense, such systems provide only “half” a guarantee—and without a full one, anything is possible.

Inconsistency
The batch system used for the end-of-day calculation often has different data semantics than the streaming system. Getting the two pipelines to produce comparable results proved more difficult than initially thought.

Complexity
By definition, Lambda requires you to write and maintain two different codebases. You also must run and maintain two complex distributed systems, each with different failure modes. For anything but the simplest of pipelines, this quickly becomes overwhelming.

Unpredictability
In many use cases, end users will see streaming results that differ from the daily results by an uncertain amount, which can change randomly. In these cases, users will stop trusting the streaming data and wait for daily batch results instead, thus destroying the value of getting low-latency results in the first place.

Latency
Some business use cases require low-latency correct results, which the Lambda Architecture does not provide by design.

Fortunately, many Beam runners can do much better. In this chapter, we explain how exactly-once stream processing helps users count on accurate results and avoid the risk of data loss while relying on a single codebase and API. Because a variety of issues that can affect a pipeline’s output are often erroneously conflated with exactly-once guarantees, we first explain precisely which issues are in and out of scope when we refer to “exactly once” in the context of Beam and data processing.

Accuracy Versus Completeness
Whenever a Beam pipeline processes a record for a pipeline, we want to ensure that the record is never dropped or duplicated. However, the nature of streaming pipelines is such that records sometimes show up late, after aggregates for their time windows have already been processed. The Beam SDK allows the user to configure how long the system should wait for late data to arrive; any (and only) records arriving later than this deadline are dropped. This feature contributes to completeness, not to accuracy: all records that showed up in time for processing are accurately processed exactly once, whereas these late records are explicitly dropped.

Although late records are usually discussed in the context of streaming systems, it’s worth noting that batch pipelines have similar completeness issues. For example, a common batch paradigm is to run a job at 2 AM over all the previous day’s data. However, if some of yesterday’s data wasn’t collected until after 2 AM, it won’t be processed by the batch job! Thus, batch pipelines also provide accurate but not always complete results.

Side Effects
One characteristic of Beam and Dataflow is that users inject custom code that is executed as part of their pipeline graph. Dataflow does not guarantee that this code is run only once per record,1 whether by the streaming or batch runner. It might run a given record through a user transform multiple times, or it might even run the same record simultaneously on multiple workers; this is necessary to guarantee at-least-once processing in the face of worker failures. Only one of these invocations can “win” and produce output further down the pipeline.

As a result, nonidempotent side effects are not guaranteed to execute exactly once; if you write code that has side effects external to the pipeline, such as contacting an outside service, these effects might be executed more than once for a given record. This situation is usually unavoidable because there is no way to atomically commit Dataflow’s processing with the side effect on the external service. Pipelines do need to eventually send results to the outside world, and such calls might not be idempotent. As you will see later in the chapter, often such sinks are able to add an extra stage to restructure the call into an idempotent operation first.

Problem Definition
So, we’ve given a couple of examples of what we’re not talking about. What do we mean then by exactly-once processing? To motivate this, let’s begin with a simple streaming pipeline,2 shown in Example 5-1.
Example 5-1. A simple streaming pipeline
Pipeline p = Pipeline.create(options);
// Calculate 1-minute counts of events per user.
PCollection<..> perUserCounts = 
      p.apply(ReadFromUnboundedSource.read())
       .apply(new KeyByUser())
       .Window.<..>into(FixedWindows.of(Duration.standardMinutes(1)))
       .apply(Count.perKey());
// Process these per-user counts, and write the output somewhere.
perUserCounts.apply(new ProcessPerUserCountsAndWriteToSink());
// Add up all these per-user counts to get 1-minute counts of all events.
perUserCounts.apply(Values.<..>create())
             .apply(Count.globally())
             .apply(new ProcessGlobalCountAndWriteToSink());
p.run();
This pipeline computes two different windowed aggregations. The first counts how many events came from each individual user over the course of a minute, and the second counts how many total events came in each minute. Both aggregations are written to unspecified streaming sinks.

Remember that Dataflow executes pipelines on many different workers in parallel. After each GroupByKey (the Count operations use GroupByKey under the covers), all records with the same key are processed on the same machine following a process called shuffle. The Dataflow workers shuffle data between themselves using Remote Procedure Calls (RPCs), ensuring that records for a given key all end up on the same machine.

Figure 5-1 shows the shuffles that Dataflow creates for the pipeline in Example 5-1.3 The Count.perKey shuffles all the data for each user onto a given worker, whereas the Count.globally shuffles all these partial counts to a single worker to calculate the global sum.


Figure 5-1. Shuffles in a pipeline
For Dataflow to accurately process data, this shuffle process must ensure that every record is shuffled exactly once. As you will see in a moment, the distributed nature of shuffle makes this a challenging problem.

This pipeline also both reads and writes data from and to the outside world, so Dataflow must ensure that this interaction does not introduce any inaccuracies. Dataflow has always supported this task—what Apache Spark and Apache Flink call end-to-end exactly once—for sources and sinks whenever technically feasible.

The focus of this chapter will be on three things:

Shuffle
How Dataflow guarantees that every record is shuffled exactly once.

Sources
How Dataflow guarantees that every source record is processed exactly once.

Sinks
How Dataflow guarantees that every sink produces accurate output.

Ensuring Exactly Once in Shuffle
As just explained, Dataflow’s streaming shuffle uses RPCs. Now, any time you have two machines communicating via RPC, you should think long and hard about data integrity. First of all, RPCs can fail for many reasons. The network might be interrupted, the RPC might time out before completing, or the receiving server might decide to fail the call. To guarantee that records are not lost in shuffle, Dataflow employs upstream backup. This simply means that the sender will retry RPCs until it receives positive acknowledgment of receipt. Dataflow also ensures that it will continue retrying these RPCs even if the sender crashes. This guarantees that every record is delivered at least once.

Now, the problem is that these retries might themselves create duplicates. Most RPC frameworks, including the one Dataflow uses, provide the sender with a status indicating success or failure. In a distributed system, you need to be aware that RPCs can sometimes succeed even when they have appeared to fail. There are many reasons for this: race conditions with the RPC timeout, positive acknowledgment from the server failing to transfer even though the RPC succeeded, and so on. The only status that a sender can really trust is a successful one.

An RPC returning a failure status generally indicates that the call might or might not have succeeded. Although specific error codes can communicate unambiguous failure, many common RPC failures, such as Deadline Exceeded, are ambiguous. In the case of streaming shuffle,4 retrying an RPC that really succeeded means delivering a record twice! Dataflow needs some way of detecting and removing these duplicates.

At a high level, the algorithm for this task is quite simple (see Figure 5-2): every message sent is tagged with a unique identifier. Each receiver stores a catalog of all identifiers that have already been seen and processed. Every time a record is received, its identifier is looked up in this catalog. If it is found, the record is dropped as a duplicate. Because Dataflow is built on top of a scalable key/value store, this store is used to hold the deduplication catalog.


Figure 5-2. Detecting duplicates in shuffle
Addressing Determinism
Making this strategy work in the real world requires a lot of care, however. One immediate wrinkle is that the Beam Model allows for user code to produce nondeterministic output. This means that a ParDo can execute twice on the same input record (due to a retry), yet produce different output on each retry. The desired behavior is that only one of those outputs will commit into the pipeline; however, the nondeterminism involved makes it difficult to guarantee that both outputs have the same deterministic ID. Even trickier, a ParDo can output multiple records, so each of these retries might produce a different number of outputs!

So, why don’t we simply require that all user processing be deterministic? Our experience is that in practice, many pipelines require nondeterministic transforms And all too often, pipeline authors do not realize that the code they wrote is nondeterministic. For example, consider a transform that looks up supplemental data in Cloud Bigtable in order to enrich its input data. This is a nondeterministic task, as the external value might change in between retries of the transform. Any code that relies on current time is likewise not deterministic. We have also seen transforms that need to rely on random number generators. And even if the user code is purely deterministic, any event-time aggregation that allows for late data might have nondeterministic inputs.

Dataflow addresses this issue by using checkpointing to make nondeterministic processing effectively deterministic. Each output from a transform is checkpointed, together with its unique ID, to stable storage before being delivered to the next stage.5 Any retries in the shuffle delivery simply replay the output that has been checkpointed—the user’s nondeterministic code is not run again on retry. To put it another way, the user’s code may be run multiple times but only one of those runs can “win.” Furthermore, Dataflow uses a consistent store that allows it to prevent duplicates from being written to stable storage.

Performance
To implement exactly-once shuffle delivery, a catalog of record IDs is stored in each receiver key. For every record that arrives, Dataflow looks up the catalog of IDs already seen to determine whether this record is a duplicate. Every output from step to step is checkpointed to storage to ensure that the generated record IDs are stable.

However, unless implemented carefully, this process would significantly degrade pipeline performance for customers by creating a huge increase in reads and writes. Thus, for exactly-once processing to be viable for Dataflow users, that I/O has to be reduced, in particular by preventing I/O on every record.

Dataflow achieves this goal via two key techniques: graph optimization and Bloom filters.

Graph Optimization
The Dataflow service runs a series of optimizations on the pipeline graph before executing it. One such optimization is fusion, in which the service fuses many logical steps into a single execution stage. Figure 5-3 shows some simple examples.


Figure 5-3. Example optimizations: fusion
All fused steps are run as an in-process unit, so there’s no need to store exactly-once data for each of them. In many cases, fusion reduces the entire graph down to a few physical steps, greatly reducing the amount of data transfer needed (and saving on state usage, as well).

Dataflow also optimizes associative and commutative Combine operations (such as Count and Sum) by performing partial combining  locally before sending the data to the main grouping operation, as illustrated in Figure 5-4. This approach can greatly reduce the number of messages for delivery, consequently also reducing the number of reads and writes.


Figure 5-4. Example optimizations: combiner lifting
Bloom Filters
The aforementioned optimizations are general techniques that improve exactly-once performance as a byproduct. For an optimization aimed strictly at improving exactly-once processing, we turn to Bloom filters.

In a healthy pipeline, most arriving records will not be duplicates. We can use that fact to greatly improve performance via Bloom filters, which are compact data structures that allow for quick set-membership checks. Bloom filters have a very interesting property: they can return false positives but never false negatives. If the filter says “Yes, the element is in the set,” we know that the element is probably in the set (with a probability that can be calculated). However, if the filter says an element is not in the set, it definitely isn’t. This function is a perfect fit for the task at hand.

The implementation in Dataflow works like this: each worker keeps a Bloom filter of every ID it has seen. Whenever a new record ID shows up, it looks it up in the filter. If the filter returns false, this record is not a duplicate and the worker can skip the more expensive lookup from stable storage. It needs to do that second lookup only if the Bloom filter returns true, but as long as the filter’s false-positive rate is low, that step is rarely needed.

Bloom filters tend to fill up over time, however, and as that happens, the false-positive rate increases. We also need to construct this Bloom filter anew any time a worker restarts by scanning the ID catalog stored in state. Helpfully, Dataflow attaches a system timestamp to each record.6 Thus, instead of creating a single Bloom filter, the service creates a separate one for every 10-minute range. When a record arrives, Dataflow queries the appropriate filter based on the system timestamp.7 This step prevents the Bloom filters from saturating because filters are garbage-collected over time, and it also bounds the amount of data that needs to be scanned at startup.8

Figure 5-5 illustrates this process: records arrive in the system and are delegated to a Bloom filter based on their arrival time. None of the records hitting the first filter are duplicates, and all of their catalog lookups are filtered. Record r1 is delivered a second time, so a catalog lookup is needed to verify that it is indeed a duplicate; the same is true for records r4 and r6. Record r8 is not a duplicate; however, due to a false positive in its Bloom filter, a catalog lookup is generated (which will determine that r8 is not a duplicate and should be processed).


Figure 5-5. Exactly-once Bloom filters
Garbage Collection
Every Dataflow worker persistently stores a catalog of unique record IDs it has seen. As Dataflow’s state and consistency model is per-key, in reality each key stores a catalog of records that have been delivered to that key. We can’t store these identifiers forever, or all available storage will eventually fill up. To avoid that issue, you need garbage collection of acknowledged record IDs.

One strategy for accomplishing this goal would be for senders to tag each record with a strictly increasing sequence number in order to track the earliest sequence number still in flight (corresponding to an unacknowledged record delivery). Any identifier in the catalog with an earlier sequence number could then be garbage-collected because all earlier records have already been acknowledged.

There is a better alternative, however. As previously mentioned, Dataflow already tags each record with a system timestamp that is used for bucketing exactly-once Bloom filters. Consequently, instead of using sequence numbers to garbage-collect the exactly-once catalog, Dataflow calculates a garbage-collection watermark based on these system timestamps (this is the processing-time watermark discussed in Chapter 3). A nice side benefit of this approach is that because this watermark is based on the amount of physical time spent waiting in a given stage (unlike the data watermark, which is based on custom event times), it provides intuition on what parts of the pipeline are slow. This metadata is the basis for the System Lag metric shown in the Dataflow WebUI.

What happens if a record arrives with an old timestamp and we’ve already garbage-collected identifiers for this point in time? This can happen due to an effect we call network remnants, in which an old message becomes stuck for an indefinite period of time inside the network and then suddenly shows up. Well, the low watermark that triggers garbage collection won’t advance until record deliveries have been acknowledged, so we know that this record has already been successfully processed. Such network remnants are clearly duplicates and are ignored.

Exactly Once in Sources
Beam provides a source API for reading data into a Dataflow pipeline.9 Dataflow might retry reads from a source if processing fails and needs to ensure that every unique record produced by a source is processed exactly once.

For most sources Dataflow handles this process transparently; such sources are deterministic. For example, consider a source that reads data out of files. The records in a file will always be in a deterministic order and at deterministic byte locations, no matter how many times the file is read.10 The filename and byte location uniquely identify each record, so the service can automatically generate unique IDs for each record. Another source that provides similar determinism guarantees is Apache Kafka; each Kafka topic is divided into a static set of partitions, and records in a partition always have a deterministic order. Such deterministic sources will work seamlessly in Dataflow with no duplicates.

However, not all sources are so simple. For example, one common source for Dataflow pipelines is Google Cloud Pub/Sub. Pub/Sub is a nondeterministic source: multiple subscribers can pull from a Pub/Sub topic, but which subscribers receive a given message is unpredictable. If processing fails Pub/Sub will redeliver messages but the messages might be delivered to different workers than those that processed them originally, and in a different order. This nondeterministic behavior means that Dataflow needs assistance for detecting duplicates because there is no way for the service to deterministically assign record IDs that will be stable upon retry. (We dive into a more detailed case study of Pub/Sub later in this chapter.)

Because Dataflow cannot automatically assign record IDs, nondeterministic sources are required to inform the system what the record IDs should be. Beam’s Source API provides the UnboundedReader.getCurrentRecordId11 method. If a source provides unique IDs per record and notifies Dataflow that it requires deduplication,12 records with the same ID will be filtered out.

Exactly Once in Sinks
At some point, every pipeline needs to output data to the outside world, and a sink is simply a transform that does exactly that. Keep in mind that delivering data externally is a side effect, and we have already mentioned that Dataflow does not guarantee exactly-once application of side effects. So, how can a sink guarantee that outputs are delivered exactly once?

The simplest answer is that a number of built-in sinks are provided as part of the Beam SDK. These sinks are carefully designed to ensure that they do not produce duplicates, even if executed multiple times. Whenever possible, pipeline authors are encouraged to use one of these built-in sinks.

However, sometimes the built-ins are insufficient and you need to write your own. The best approach is to ensure that your side-effect operation is idempotent and therefore robust in the face of replay. However, often some component of a side-effect DoFn is nondeterministic and thus might change on replay. For example, in a windowed aggregation, the set of records in the window can also be nondeterministic!

Specifically, the window might attempt to fire with elements e0, e1, e2, but the worker crashes before committing the window processing (but not before those elements are sent as a side effect). When the worker restarts, the window will fire again, but now a late element e3 shows up. Because this element shows up before the window is committed, it’s not counted as late data, so the DoFn is called again with elements e0, e1, e2, e3. These are then sent to the side-effect operation. Idempotency does not help here, because different logical record sets were sent each time.

There are other ways nondeterminism can be introduced. The standard way to address this risk is to rely on the fact that Dataflow currently guarantees that only one version of a DoFn’s output can make it past a shuffle boundary.13

A simple way of using this guarantee is via the built-in Reshuffle transform. The pattern presented in Example 5-2 ensures that the side-effect operation always receives a deterministic record to output.

Example 5-2. Reshuffle example
c.apply(Window.<..>into(FixedWindows.of(Duration.standardMinutes(1))))
 .apply(GroupByKey.<..>.create())
 .apply(new PrepareOutputData())
 .apply(Reshuffle.<..>of())
 .apply(WriteToSideEffect());
The preceding pipeline splits the sink into two steps: PrepareOutputData and WriteToSideEffect. PrepareOutputData outputs records corresponding to idempotent writes. If we simply ran one after the other, the entire process might be replayed on failure, PrepareOutputData might produce a different result, and both would be written as side effects. When we add the Reshuffle in between the two, Dataflow guarantees this can’t happen.

Of course, Dataflow might still run the WriteToSideEffect operation multiple times. The side effects themselves still need to be idempotent, or the sink will receive duplicates. For example, an operation that sets or overwrites a value in a data store is idempotent, and will generate correct output even if it’s run several times. An operation that appends to a list is not idempotent; if the operation is run multiple times, the same value will be appended each time.

While Reshuffle provides a simple way of achieving stable input to a DoFn, a GroupByKey works just as well. However, there is currently a proposal that removes the need to add a GroupByKey to achieve stable input into a DoFn. Instead, the user could annotate WriteToSideEffect with a special annotation, @RequiresStableInput, and the system would then ensure stable input to that transform.

Use Cases
To illustrate, let’s examine some built-in sources and sinks to see how they implement the aforementioned patterns.

Example Source: Cloud Pub/Sub
Cloud Pub/Sub is a fully managed, scalable, reliable, and low-latency system for delivering messages from publishers to subscribers. Publishers publish data on named topics, and subscribers create named subscriptions to pull data from these topics. Multiple subscriptions can be created for a single topic, in which case each subscription receives a full copy of all data published on the topic from the time of the subscription’s creation. Pub/Sub guarantees that records will continue to be delivered until they are acknowledged; however, a record might be delivered multiple times.

Pub/Sub is intended for distributed use, so many publishing processes can publish to the same topic and many subscribing processes can pull from the same subscription. After a record has been pulled, the subscriber must acknowledge it within a certain amount of time, or that pull expires and Pub/Sub will redeliver that record to another of the subscribing processes.

Although these characteristics make Pub/Sub highly scalable, they also make it a challenging source for a system like Dataflow. It’s impossible to know which record will be delivered to which worker, and in which order. What’s more, in the case of failure, redelivery might send the records to different workers in different orders!

Pub/Sub provides a stable message ID with each message, and this ID will be the same upon redelivery. The Dataflow Pub/Sub source will default to using this ID for removing duplicates from Pub/Sub. (The records are shuffled based on a hash of the ID, so that repeated deliveries are always processed on the same worker.) In some cases, however, this is not quite enough. The user’s publishing process might retry publishes, and as a result introduce duplicates into Pub/Sub. From that service’s perspective these are unique records, so they will get unique record IDs. Dataflow’s Pub/Sub source allows the user to provide their own record IDs as a custom attribute. As long as the publisher sends the same ID when retrying, Dataflow will be able to detect these duplicates.

Beam (and therefore Dataflow) provides a reference source implementation for Pub/Sub. However, keep in mind that this is not what Dataflow uses but rather an implementation used only by non-Dataflow runners (such as Apache Spark, Apache Flink, and the DirectRunner). For a variety of reasons, Dataflow handles Pub/Sub internally and does not use the public Pub/Sub source.

Example Sink: Files
The streaming runner can use Beam’s file sinks (TextIO, AvroIO, and any other sink that implements FileBasedSink) to continuously output records to files. Example 5-3 provides an example use case.

Example 5-3. Windowed file writes
c.apply(Window.<..>into(FixedWindows.of(Duration.standardMinutes(1))))
 .apply(TextIO.writeStrings().to(new MyNamePolicy()).withWindowedWrites());
The snippet in Example 5-3 writes 10 new files each minute, containing data from that window. MyNamePolicy is a user-written function that determines output filenames based on the shard and the window. You can also use triggers, in which case each trigger pane will be output as a new file.

This process is implemented using a variant on the pattern in Example 5-3. Files are written out to temporary locations, and these temporary filenames are sent to a subsequent transform through a GroupByKey. After the GroupByKey is a finalize transform that atomically moves the temporary files into their final location. The pseudocode in Example 5-4 provides a sketch of how a consistent streaming file sink is implemented in Beam. (For more details, see FileBasedSink and WriteFiles in the Beam codebase.)

Example 5-4. File sink
c
  // Tag each record with a random shard id.
  .apply("AttachShard", WithKeys.of(new RandomShardingKey(getNumShards())))
  // Group all records with the same shard.
  .apply("GroupByShard", GroupByKey.<..>())
  // For each window, write per-shard elements to a temporary file. This is the 
  // non-deterministic side effect. If this DoFn is executed multiple times, it will
  // simply write multiple temporary files; only one of these will pass on through 
  // to the Finalize stage.
  .apply("WriteTempFile", ParDo.of(new DoFn<..> {
    @ProcessElement
     public void processElement(ProcessContext c, BoundedWindow window) {
       // Write the contents of c.element() to a temporary file.
       // User-provided name policy used to generate a final filename.
      c.output(new FileResult()).
    }
  }))
  // Group the list of files onto a singleton key.
  .apply("AttachSingletonKey", WithKeys.<..>of((Void)null))
  .apply("FinalizeGroupByKey", GroupByKey.<..>create())
  // Finalize the files by atomically renaming them. This operation is idempotent. 
  // Once this DoFn has executed once for a given FileResult, the temporary file  
  // is gone, so any further executions will have no effect. 
  .apply("Finalize", ParDo.of(new DoFn<..>, Void> {
    @ProcessElement
     public void processElement(ProcessContext c)  {
       for (FileResult result : c.element()) { 
         rename(result.getTemporaryFileName(), result.getFinalFilename());
       }
}}));
You can see how the nonidempotent work is done in WriteTempFile. After the GroupByKey completes, the Finalize step will always see the same bundles across retries. Because file rename is idempotent,14 this give us an exactly-once sink.

Example Sink: Google BigQuery
Google BigQuery is a fully managed, cloud-native data warehouse. Beam provides a BigQuery sink, and BigQuery provides a streaming insert API that supports extremely low-latency inserts. This streaming insert API allows allows you to tag inserts with a unique ID, and BigQuery will attempt to filter duplicate inserts with the same ID.15 To use this capability, the BigQuery sink must generate statistically unique IDs for each record. It does this by using the java.util.UUID package, which generates statistically unique 128-bit IDs.

Generating a random universally unique identifier (UUID) is a nondeterministic operation, so we must add a Reshuffle before we insert into BigQuery. After we do this, any retries by Dataflow will always use the same UUID that was shuffled. Duplicate attempts to insert into BigQuery will always have the same insert ID, so BigQuery is able to filter them. The pseudocode shown in Example 5-5 illustrates how the BigQuery sink is implemented.

Example 5-5. BigQuery sink
// Apply a unique identifier to each record
c
 .apply(new DoFn<> {
  @ProcessElement
  public void processElement(ProcessContext context) {
   String uniqueId = UUID.randomUUID().toString();
   context.output(KV.of(ThreadLocalRandom.current().nextInt(0, 50),
                                     new RecordWithId(context.element(), uniqueId)));
 }
})
// Reshuffle the data so that the applied identifiers are stable and will not change.
.apply(Reshuffle.<Integer, RecordWithId>of())
// Stream records into BigQuery with unique ids for deduplication.
.apply(ParDo.of(new DoFn<..> {
   @ProcessElement
   public void processElement(ProcessContext context) {
     insertIntoBigQuery(context.element().record(), context.element.id());
   }
 });
Again we split the sink into a nonidempotent step (generating a random number), followed by a step that is idempotent. 

Other Systems
Now that we have explained Dataflow’s exactly once in detail, let us contrast this with some brief overviews of other popular streaming systems. Each implements exactly-once guarantees in a different way and makes different trade-offs as a result.

Apache Spark Streaming
Spark Streaming uses a microbatch architecture for continuous data processing. Users logically deal with a stream object; however, under the covers, Spark represents this stream as a continuous series of RDDs.16 Each RDD is processed as a batch, and Spark relies on the exactly-once nature of batch processing to ensure correctness; as mentioned previously, techniques for correct batch shuffles have been known for some time. This approach can cause increased latency to output—especially for deep pipelines and high input volumes—and often careful tuning is required to achieve desired latency.

Spark does assume that operations are all idempotent and might replay the chain of operations up the current point in the graph. A checkpoint primitive is provided, however, that causes an RDD to be materialized, guaranteeing that history prior to that RDD will not be replayed. This checkpoint feature is intended for performance reasons (e.g., to prevent replaying an expensive operation); however, you can also use it to implement nonidempotent side effects.

Apache Flink
Apache Flink also provides exactly-once processing for streaming pipelines but does so in a manner different than either Dataflow or Spark. Flink streaming pipelines periodically compute consistent snapshots, each representing the consistent point-in-time state of an entire pipeline. Flink snapshots are computed progressively, so there is no need to halt all processing while computing a snapshot. This allows records to continue flowing through the system while taking a snapshot, alleviating some of the latency issues with the Spark Streaming approach.

Flink implements these snapshots by inserting special numbered snapshot markers into the data streams flowing from sources. As each operator receives a snapshot marker, it executes a specific algorithm allowing it to copy its state to an external location and propagate the snapshot marker to downstream operators. After all operators have executed this snapshot algorithm, a complete snapshot is made available. Any worker failures will cause the entire pipeline to roll back its state from the last complete snapshot. In-flight messages do not need to be included in the snapshot. All message delivery in Flink is done via an ordered TCP-based channel. Any connection failures can be handled by resuming the connection from the last good sequence number;17 unlike Dataflow, Flink tasks are statically allocated to workers, so it can assume that the connection will resume from the same sender and replay the same payloads.

Because Flink might roll back to the previous snapshot at any time, any state modifications not yet in a snapshot must be considered tentative. A sink that sends data to the world outside the Flink pipeline must wait until a snapshot has completed, and then send only the data that is included in that snapshot. Flink provides a notifySnapshotComplete callback that allows sinks to know when each snapshot is completed, and send the data onward. Even though this does affect the output latency of Flink pipelines,18 this latency is introduced only at sinks. In practice, this allows Flink to have lower end-to-end latency than Spark for deep pipelines because Spark introduces batch latency at each stage in the pipeline.

Flink’s distributed snapshots are an elegant way of dealing with consistency in a streaming pipeline; however, a number of assumptions are made about the pipeline. Failures are assumed to be rare,19 as the impact of a failure (rolling back to the previous snapshot) is substantial. To maintain low-latency output, it is also assumed that snapshots can complete quickly. It remains to be seen whether this causes issues on very large clusters where the failure rate will likely increase, as will the time needed to complete a snapshot.

Implementation is also simplified by assuming that tasks are statically allocated to workers (at least within a single snapshot epoch). This assumption allows Flink to provide a simple exactly-once transport between workers because it knows that if a connection fails, the same data can be pulled in order from the same worker. In contrast, tasks in Dataflow are constantly load balanced between workers (and the set of workers is constantly growing and shrinking), so Dataflow is unable to make this assumption. This forces Dataflow to implement a much more complex transport layer in order to provide exactly-once processing.

Summary
In summary, exactly-once data processing, which was once thought to be incompatible with low-latency results, is quite possible—Dataflow does it efficiently without sacrificing latency. This enables far richer uses for stream processing.

Although this chapter has focused on Dataflow-specific techniques, other streaming systems also provide exactly-once guarantees. Apache Spark Streaming runs streaming pipelines as a series of small batch jobs, relying on exactly-once guarantees in the Spark batch runner. Apache Flink uses a variation on Chandy Lamport distributed snapshots to get a running consistent state and can use these snapshots to ensure exactly-once processing. We encourage you to learn about these other systems, as well, for a broad understanding of how different stream-processing systems work! 

1 In fact, no system we are aware of that provides at-least once (or better) is able to guarantee this, including all other Beam runners.

2 Dataflow also provides an accurate batch runner; however, in this context we are focused on the streaming runner.

3 The Dataflow optimizer groups many steps together and adds shuffles only where they are needed.

4 Batch pipelines also need to guard against duplicates in shuffle. However the problem is much easier to solve in batch, which is why historical batch systems did do this and streaming systems did not. Streaming runtimes that use a microbatch architecture, such as Spark Streaming, delegate duplicate detection to a batch shuffler.

5 A lot of care is taken to make sure this checkpointing is efficient; for example, schema and access pattern optimizations that are intimately tied to the characteristics of the underlying key/value store.

6 This is not the custom user-supplied timestamp used for windowing. Rather this is a deterministic processing-time timestamp that is assigned by the sending worker.

7 Some care needs to be taken to ensure that this algorithm works. Each sender must guarantee that the system timestamps it generates are strictly increasing, and this guarantee must be maintained across worker restarts.

8 In theory, we could dispense with startup scans entirely by lazily building the Bloom filter for a bucket only when a threshold number of records show up with timestamps in that bucket.

9 At the time of this writing, a new, more-flexible API called SplittableDoFn is available for Apache Beam.

10 We assume that nobody is maliciously modifying the bytes in the file while we are reading it.

11 Again note that the SplittableDoFn API has different methods for this.

12 Using the requiresDedupping override.

13 Note that these determinism boundaries might become more explicit in the Beam Model at some point. Other Beam runners vary in their ability to handle nondeterministic user code.

14 As long as you properly handle the failure when the source file no longer exists.

15 Due to the global nature of the service, BigQuery does not guarantee that all duplicates are removed. Users can periodically run a query over their tables to remove any duplicates that were not caught by the streaming insert API. See the BigQuery documentation for more information.

16 Resilient Distributed Datasets; Spark’s abstraction of a distributed dataset, similar to PCollection in Beam.

17 These sequence numbers are per connection and are unrelated to the snapshot epoch number.

18 Only for nonidempotent sinks. Completely idempotent sinks do not need to wait for the snapshot to complete.

19 Specifically, Flink assumes that the mean time to worker failure is less than the time to snapshot; otherwise, the pipeline would be unable to make progress.