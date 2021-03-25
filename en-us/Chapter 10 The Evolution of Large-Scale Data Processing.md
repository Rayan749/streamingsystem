# Chapter 10. The Evolution of Large-Scale Data Processing
You have now arrived at the final chapter in the book, you stoic literate, you. Your journey will soon be complete!

To wrap things up, I’d like you to join me on a brief stroll through history, starting back in the ancient days of large-scale data processing with MapReduce and touching upon some of the highlights over the ensuing decade and a half that have brought streaming systems to the point they’re at today. It’s a relatively lightweight chapter in which I make a few observations about important contributions from a number of well-known systems (and a couple maybe not-so-well known), refer you to a bunch of source material you can go read on your own should you want to learn more, all while attempting not to offend or inflame the folks responsible for systems whose truly impactful contributions I’m going to either oversimplify or ignore completely for the sake of space, focus, and a cohesive narrative. Should be a good time.

On that note, keep in mind as you read this chapter that we’re really just talking about specific pieces of the MapReduce/Hadoop family tree of large-scale data processing here. I’m not covering the SQL arena in any way shape or form1; we’re not talking HPC/supercomputers, and so on. So as broad and expansive as the title of this chapter might sound, I’m really focusing on a specific vertical swath of the grand universe of large-scale data processing. Caveat literatus, and all that.

Also note that I’m covering a disproportionate amount of Google technologies here. You would be right in thinking that this might have something to do with the fact that I’ve worked at Google for more than a decade. But there are two other reasons for it: 1) big data has always been important for Google, so there have been a number of worthwhile contributions created there that merit discussing in detail, and 2) my experience has been that folks outside of Google generally seem to enjoy learning more about the things we’ve done, because we as a company have historically been somewhat tight-lipped in that regard. So indulge me a bit while I prattle on excessively about the stuff we’ve been working on behind closed doors.

To ground our travels in concrete chronology, we’ll be following the timeline in Figure 10-1, which shows rough dates of existence for the various systems I discuss.


Figure 10-1. Approximate timeline of systems discussed in this chapter
At each stop, I give a brief history of the system as best I understand it and frame its contributions from the perspective of shaping streaming systems as we know them today. At the end, we recap all of the contributions to see how they’ve summed up to create the modern stream processing ecosystem of today.

MapReduce
We begin the journey with MapReduce (Figure 10-2).


Figure 10-2. Timeline: MapReduce
I think it’s safe to say that large-scale data processing as we all know it today got its start with MapReduce way back in 2003.2 At the time, engineers within Google were building all sorts of bespoke systems to tackle data processing challenges at the scale of the World Wide Web.3 As they did so, they noticed three things:

Data processing is hard
As the data scientists and engineers among us well know, you can build a career out of just focusing on the best ways to extract useful insights from raw data.

Scalability is hard
Extracting useful insights over massive-scale data is even more difficult yet.

Fault-tolerance is hard
Extracting useful insights from massive-scale data in a fault-tolerant, correct way on commodity hardware is brutal.

After solving all three of these challenges in tandem across a number of use cases, they began to notice some similarities between the custom systems they’d built. And they came to the conclusion that if they could build a framework that took care of the latter two issues (scalability and fault-tolerance), it would make focusing on the first issue a heck of a lot simpler. Thus was born MapReduce.4

The basic idea with MapReduce was to provide a simple data processing API centered around two well-understand operations from the functional programming realm: map and reduce (Figure 10-3). Pipelines built with that API would then be executed on a distributed systems framework that took care of all the nasty scalability and fault-tolerance stuff that quickens the hearts of hardcore distributed-systems engineers and crushes the souls of the rest of us mere mortals.


Figure 10-3. Visualization of a MapReduce job
We already discussed the semantics of MapReduce in great detail back in Chapter 6, so we won’t dwell on them here. Simply recall that we broke things down into six discrete phases (MapRead, Map, MapWrite, ReduceRead, Reduce, ReduceWrite) as part of our streams and tables analysis, and we came to the conclusion in the end that there really wasn’t all that much different between the overall Map and Reduce phases; at a high-level, they both do the following:

Convert a table to a stream

Apply a user transformation to that stream to yield another stream

Group that stream into a table

After it was placed into service within Google, MapReduce found such broad application across a variety of tasks that the team decided it was worth sharing its ideas with the rest of the world. The result was the MapReduce paper, published at OSDI 2004 (see Figure 10-4).


Figure 10-4. The MapReduce paper, published at OSDI 2004
In it, the team described in detail the history of the project, design of the API and implementation, and details about a number of different use cases to which MapReduce had been applied. Unfortunately, they provided no actual source code, so the best that folks outside of Google at the time could do was say, “Yes, that sounds very nice indeed,” and go back to building their bespoke systems.

Over the course of the decade that followed, MapReduce continued to undergo heavy development within Google, with large amounts of time invested in making the system scale to unprecedented levels. For a more detailed account of some of the highlights along that journey, I recommend the post “History of massive-scale sorting experiments at Google” (Figure 10-5) written by our official MapReduce historian/scalability and performance wizard, Marián Dvorský.


Figure 10-5. Marián Dvorský’s “History of massive-scale sorting experiments” blog post
But for our purposes here, suffice it to say that nothing else yet has touched the magnitude of scale achieved by MapReduce, not even within Google. Considering how long MapReduce has been around, that’s saying something; 14 years is an eternity in our industry.

From a streaming systems perspective, the main takeaways I want to leave you with for MapReduce are simplicity and scalability. MapReduce took the first brave steps toward taming the unruly beast that is massive-scale data processing, exposing a simple and straightforward API for crafting powerful data processing pipelines, its austerity belying the complex distributed systems magic happening under the covers to allow those pipelines to run at scale on large clusters of commodity hardware. 

Hadoop
Next in our list is Hadoop (Figure 10-6). Fair warning: this is one of those times where I will grossly oversimplify the impact of a system for the sake of a focused narrative. The impact Hadoop has had on our industry and the world at large cannot be overstated, and it extends well beyond the relatively specific scope I discuss here.


Figure 10-6. Timeline: Hadoop
Hadoop came about in 2005, when Doug Cutting and Mike Cafarella decided that the ideas from the MapReduce paper were just the thing they needed as they built a distributed version of their Nutch webcrawler. They had already built their own version of Google’s distributed filesystem (originally called NDFS for Nutch Distributed File System, later renamed to HDFS, or Hadoop Distributed File System), so it was a natural next step to add a MapReduce layer on top after that paper was published. They called this layer Hadoop.

The key difference between Hadoop and MapReduce was that Cutting and Cafarella made sure the source code for Hadoop was shared with the rest of the world by open sourcing it (along with the source for HDFS) as part of what would eventually become the Apache Hadoop project. Yahoo’s hiring of Cutting to help transition the Yahoo webcrawler architecture onto Hadoop gave the project an additional boost of validity and engineering oomph, and from there, an entire ecosystem of open source data processing tools grew. As with MapReduce, others have told the history of Hadoop in other fora far better than I can; one particularly good reference is Marko Bonaci’s “The history of Hadoop,” itself originally slated for inclusion in a print book (Figure 10-7).


Figure 10-7. Marko Bonaci’s “The history of Hadoop”
The main point I want you to take away from this section is the massive impact the open source ecosystem that flowered around Hadoop had upon the industry as a whole. By creating an open community in which engineers could improve and extend the ideas from those early GFS and MapReduce papers, a thriving ecosystem was born, yielding dozens of useful tools like Pig, Hive, HBase, Crunch, and on and on. That openness was key to incubating the diversity of ideas that exist now across our industry, and it’s why I’m pigeonholing Hadoop’s open source ecosystem as its single most important contribution to the world of streaming systems as we know them today.

Flume
We now return to Google territory to talk about the official successor to MapReduce within Google: Flume ([Figure 10-8] sometimes also called FlumeJava in reference to the original Java version of the system, and not to be confused with Apache Flume, which is an entirely different beast that just so happens to share the same name).


Figure 10-8. Timeline: Flume
The Flume project was founded by Craig Chambers when the Google Seattle office opened in 2007. It was motivated by a desire to solve some of the inherent shortcomings of MapReduce, which had become apparent over the first few years of its success. Many of these shortcomings revolved around MapReduce’s rigid Map → Shuffle → Reduce structure; though refreshingly simple, it carried with it some downsides:

Because many use cases cannot be served by the application of a single MapReduce, a number of bespoke orchestration systems began popping up across Google for coordinating sequences of MapReduce jobs. These systems all served essentially the same purpose (gluing together multiple MapReduce jobs to create a coherent pipeline solving a complex problem). However, having been developed independently, they were naturally incompatible and a textbook example of unnecessary duplication of effort.

What’s worse, there were numerous cases in which a clearly written sequence of MapReduce jobs would introduce inefficiencies thanks to the rigid structure of the API. For example, one team might write a MapReduce that simply filtered out some number of elements; that is, a map-only job with an empty reducer. It might be followed up by another team’s map-only job doing some element-wise enrichment (with yet another empty reducer). The output from the second job might then finally be consumed by a final team’s MapReduce performing some grouping aggregation over the data. This pipeline, consisting of essentially a single chain of Map phases followed by a single Reduce phase, would require the orchestration of three completely independent jobs, each chained together by shuffle and output phases materializing the data. But that’s assuming you wanted to keep the codebase logical and clean, which leads to the final downside…

In an effort to optimize away these inefficiencies in their MapReductions, engineers began introducing manual optimizations that would obfuscate the simple logic of the pipeline, increasing maintenance and debugging costs.

Flume addressed these issues by providing a composable, high-level API for describing data processing pipelines, essentially based around the same PCollection and PTransform concepts found in Beam, as illustrated in Figure 10-9.


Figure 10-9. High-level pipelines in Flume (image credit: Frances Perry)
These pipelines, when launched, would be fed through an optimizer5 to generate a plan for an optimally efficient sequence of MapReduce jobs, the execution of which was then orchestrated by the framework, which you can see illustrated in Figure 10-10.


Figure 10-10. Optimization from a logical pipeline to a physical execution plan
Perhaps the most important example of an automatic optimization that Flume can perform is fusion (which Reuven discussed a bit back in Chapter 5), in which two logically independent stages can be run in the same job either sequentially (consumer-producer fusion) or in parallel (sibling fusion), as depicted in Figure 10-11.


Figure 10-11. Fusion optimizations combine successive or parallel operations together into the same physical operation
Fusing two stages together eliminates serialization/deserialization and network costs, which can be significant in pipelines processing large amounts of data.

Another type of automatic optimization is combiner lifting (see Figure 10-12), the mechanics of which we already touched upon in Chapter 7 when we talked about incremental combining. Combiner lifting is simply the automatic application of multilevel combine logic that we discussed in that chapter: a combining operation (e.g., summation) that logically happens after a grouping operation is partially lifted into the stage preceding the group-by-key (which by definition requires a trip across the network to shuffle the data) so that it can perform partial combining before the grouping happens. In cases of very hot keys, this can greatly reduce the amount of data shuffled over the network, and also spread the load of computing the final aggregate more smoothly across multiple machines.


Figure 10-12. Combiner lifting applies partial aggregation on the sender side of a group-by-key operation before completing aggregation on the consumer side
As a result of its cleaner API and automatic optimizations, Flume Java was an instant hit upon its introduction at Google in early 2009. Following on the heels of that success, the team published the paper titled “Flume Java: Easy, Efficient Data-Parallel Pipelines” (see Figure 10-13), itself an excellent resource for learning more about the system as it originally existed.


Figure 10-13. FlumeJava paper
Flume C++ followed not too much later in 2011, and in early 2012 Flume was introduced into Noogler6 training provided to all new engineers at Google. That was the beginning of the end for MapReduce.

Since then, Flume has been migrated to no longer use MapReduce as its execution engine; instead, it uses a custom execution engine, called Dax, built directly into the framework itself. By freeing Flume itself from the confines of the previously underlying Map → Shuffle → Reduce structure of MapReduce, Dax enabled new optimizations, such as the dynamic work rebalancing feature described in Eugene Kirpichov and Malo Denielou’s “No shard left behind” blog post (Figure 10-14).


Figure 10-14. “No shard left behind” post
Though discussed in that post in the context of Cloud Dataflow, dynamic work rebalancing (or liquid sharding, as it’s colloquially known at Google) automatically rebalances extra work from straggler shards to other idle workers in the system as they complete their work early. By dynamically rebalancing the work distribution over time, it’s possible to come much closer to an optimal work distribution than even the best educated initial splits could ever achieve. It also allows for adapting to variations across the pool of workers, where a slow machine that might have otherwise held up the completion of a job is simply compensated for by moving most of its tasks to other workers. When liquid sharding was rolled out at Google, it recouped significant amounts of resources across the fleet.

One last point on Flume is that it was also later extended to support streaming semantics. In addition to the batch Dax backend, Flume was extended to be able to execute pipelines on the MillWheel stream processing system (discussed in a moment). Most of the high-level streaming semantics concepts we’ve discussed in this book were first incorporated into Flume before later finding their way into Cloud Dataflow and eventually Apache Beam.

All that said, the primary thing to take away from Flume in this section is the introduction of a notion of high-level pipelines, which enabled the automatic optimization of clearly written, logical pipelines. This enabled the creation of much larger and complex pipelines, without the need for manual orchestration or optimization, and all while keeping the code for those pipelines logical and clear.

Storm
Next up is Apache Storm (Figure 10-15), the first real streaming system we cover. Storm most certainly wasn’t the first streaming system in existence, but I would argue it was the first streaming system to see truly broad adoption across the industry, and for that reason we give it a closer look here.


Figure 10-15. Timeline: Storm

Figure 10-16. “History of Apache Storm and lessons learned”
Storm was the brainchild of Nathan Marz, who later chronicled the history of its creation in a blog post titled “History of Apache Storm and lessons learned” (Figure 10-16). The TL;DR version of it is that Nathan’s team at the startup employing him then, BackType, had been attempting to process the Twitter firehose using a custom system of queues and workers. He came to essentially the same realization that the MapReduce folks had nearly a decade earlier: the actual data processing portion of their code was only a tiny amount of the system, and building those real-time data processing pipelines would be a lot easier if there were a framework doing all the distributed system’s dirty work under the covers. Out of that was born Storm.

The interesting thing about Storm, in comparison to the rest of the systems we’ve talked about so far, is that the team chose to loosen the strong consistency guarantees found in all of the other systems we’ve talked about so far as a way of providing lower latency. By combining at-most once or at-least once semantics with per-record processing and no integrated (i.e., no consistent) notion of persistent state, Storm was able provide much lower latency in providing results than systems that executed over batches of data and guaranteed exactly-once correctness. And for a certain type of use cases, this was a very reasonable trade-off to make.

Unfortunately, it quickly became clear that people really wanted to have their cake and eat it, too. They didn’t just want to get their answers quickly, they wanted to have both low-latency results and eventual correctness. But such a thing was impossible with Storm alone. Enter the Lambda Architecture.

Given the limitations of Storm, shrewd engineers began running a weakly consistent Storm streaming pipeline alongside a strongly consistent Hadoop batch pipeline. The former produced low-latency, inexact results, whereas the latter produced high-latency, exact results, both of which would then be somehow merged together in the end to provide a single low-latency, eventually consistent view of the outputs. We learned back in Chapter 1 that the Lambda Architecture was Marz’s other brainchild, as detailed in his post titled “How to beat the CAP theorem” (Figure 10-17).7


Figure 10-17. “How to beat the CAP theorem”
I’ve already spent a fair amount of time harping on the shortcomings of the Lambda Architecture, so I won’t belabor those points here. But I will reiterate this: the Lambda Architecture became quite popular, despite the costs and headaches associated with it, simply because it met a critical need that a great many businesses were otherwise having a difficult time fulfilling: that of getting low-latency, but eventually correct results out of their data processing pipelines.

From the perspective of the evolution of streaming systems, I argue that Storm was responsible for first bringing low-latency data processing to the masses. However, it did so at the cost of weak consistency, which in turn brought about the rise of the Lambda Architecture, and the years of dual-pipeline darkness that followed.


Figure 10-18. Heron paper
But hyperbolic dramaticism aside, Storm was the system that gave the industry its first taste of low-latency data processing, and the impact of that is reflected in the broad interest in and adoption of streaming systems today.

Before moving on, it’s also worth giving a shout out to Heron. In 2015, Twitter (the largest known user of Storm in the world, and the company that originally fostered the Storm project) surprised the industry by announcing it was abandoning the Storm execution engine in favor of a new system it had developed in house, called Heron. Heron aimed to address a number of performance and maintainability issues that had plagued Storm, while remaining API compatible, as detailed in the company’s paper titled “Twitter Heron: Stream Processing at Scale” (Figure 10-18). Heron itself was subsequently open sourced (with governance moved to its own independent foundation, not an existing one like Apache). Given the continued development on Storm, there are now two competing variants of the Storm lineage. Where things will end up is anyone’s guess, but it will be exciting to watch.

Spark
Moving on, we now come to Apache Spark (Figure 10-19). This is another section in which I’m going to greatly oversimplify the total impact that Spark has had on the industry by focusing on a specific portion of its contributions: those within the realm of stream processing. Apologies in advance.


Figure 10-19. Timeline: Spark
Spark got its start at the now famous AMPLab in UC Berkeley around 2009. The thing that initially fueled Spark’s fame was its ability to oftentimes perform the bulk of a pipeline’s calculations entirely in memory, without touching disk until the very end. Engineers achieved this via the Resilient Distributed Dataset (RDD) idea, which basically captured the full lineage of data at any given point in the pipeline, allowing intermediate results to be recalculated as needed on machine failure, under the assumptions that a) your inputs were always replayable, and b) your computations were deterministic. For many use cases, these preconditions were true, or at least true enough given the massive gains in performance users were able to realize over standard Hadoop jobs. From there, Spark gradually built up its eventual reputation as Hadoop’s de facto successor.

A few years after Spark was created, Tathagata Das, then a graduate student in the AMPLab, came to the realization that: hey, we’ve got this fast batch processing engine, what if we just wired things up so we ran multiple batches one after another, and used that to process streaming data? From that bit of insight, Spark Streaming was born. 

What was really fantastic about Spark Streaming was this: thanks to the strongly consistent batch engine powering things under the covers, the world now had a stream processing engine that could provide correct results all by itself without needing the help of an additional batch job. In other words, given the right use case, you could ditch your Lambda Architecture system and just use Spark Streaming. All hail Spark Streaming!

The one major caveat here was the “right use case” part. The big downside to the original version of Spark Streaming (the 1.x variants) was that it provided support for only a specific flavor of stream processing: processing-time windowing. So any use case that cared about event time, needed to deal with late data, and so on, couldn’t be handled out of the box without a bunch of extra code being written by the user to implement some form of event-time handling on top of Spark’s processing-time windowing architecture. This meant that Spark Streaming was best suited for in-order data or event-time-agnostic computations. And, as I’ve reiterated throughout this book, those conditions are not as prevalent as you would hope when dealing with the large-scale, user-centric datasets common today.

Another interesting controversy that surrounds Spark Streaming is the age-old “microbatch versus true streaming” debate. Because Spark Streaming is built upon the idea of small, repeated runs of a batch processing engine, detractors claim that Spark Streaming is not a true streaming engine in the sense that progress in the system is gated by the global barriers of each batch. There’s some amount of truth there. Even though true streaming engines almost always utilize some sort of batching or bundling for the sake of throughput, they have the flexibility to do so at much finer-grained levels, down to individual keys. The fact that microbatch architectures process bundles at a global level means that it’s virtually impossible to have both low per-key latency and high overall throughput, and there are a number of benchmarks that have shown this to be more or less true. But at the same time, latency on the order of minutes or multiple seconds is still quite good. And there are very few use cases that demand exact correctness and such stringent latency capabilities. So in some sense, Spark was absolutely right to target the audience it did originally; most people fall in that category. But that hasn’t stopped its competitors from slamming this as a massive disadvantage for the platform. Personally, I see it as a minor complaint at best in most cases.

Shortcomings aside, Spark Streaming was a watershed moment for stream processing: the first publicly available, large-scale stream processing engine that could also provide the correctness guarantees of a batch system. And of course, as previously noted, streaming is only a very small part of Spark’s overall success story, with important contributions made in the space of iterative processing and machine learning, its native SQL integration, and the aforementioned lightning-fast in-memory performance, to name a few.

If you’re curious to learn more about the details of the original Spark 1.x architecture, I highly recommend Matei Zaharia’s dissertation on the subject, “An Architecture for Fast and General Data Processing on Large Clusters” (Figure 10-20). It’s 113 pages of Sparky goodness that’s well worth the investment.


Figure 10-20. Spark dissertation
As of today, the 2.x variants of Spark are greatly expanding upon the semantic capabilities of Spark Streaming, incorporating many parts of the model described in this book, while attempting to simplify some of the more complex pieces. And Spark is even pushing a new true streaming architecture, to try to shut down the microbatch naysayer arguments. But when it first came on the scene, the important contribution that Spark brought to the table was the fact that it was the first publicly available stream processing engine with strong consistency semantics, albeit only in the case of in-order data or event-time-agnostic computation.

MillWheel
Next we discuss MillWheel, a project that I first dabbled with in my 20% time after joining Google in 2008, later joining the team full time in 2010 (Figure 10-21).


Figure 10-21. Timeline: MillWheel
MillWheel is Google’s original, general-purpose stream processing architecture, and the project was founded by Paul Nordstrom around the time Google’s Seattle office opened. MillWheel’s success within Google has long centered on an ability to provide low-latency, strongly consistent processing of unbounded, out-of-order data. Over the course of this book, we’ve looked at most of the bits and pieces that came together in MillWheel to make this possible:

Reuven discussed exactly-once guarantees in Chapter 5. Exactly-once guarantees are essential for correctness.

In Chapter 7 we looked at persistent state, the strongly consistent variations of which provide the foundation for maintaining that correctness in long-running pipelines executing on unreliable hardware.

Slava talked about watermarks in Chapter 3. Watermarks provide a foundation for reasoning about disorder in input data.

Also in Chapter 7, we looked at persistent timers, which provide the necessary link between watermarks and the pipeline’s business logic.

It’s perhaps somewhat surprising then to note that the MillWheel project was not initially focused on correctness. Paul’s original vision more closely targeted the niche that Storm later espoused: low-latency data processing with weak consistency. It was the initial MillWheel customers, one building sessions over search data and another performing anomaly detection on search queries (the Zeitgeist example from the MillWheel paper), who drove the project in the direction of correctness. Both had a strong need for consistent results: sessions were used to infer user behavior, and anomaly detection was used to infer trends in search queries; the utility of both decreased significantly if the data they provided were not reliable. As a result, MillWheel’s direction was steered toward one of strong consistency.

Support for out-of-order processing, which is the other core aspect of robust streaming often attributed to MillWheel, was also motivated by customers. The Zeitgeist pipeline, as a true streaming use case, wanted to generate an output stream that identified anomalies in search query traffic, and only anomalies (i.e., it was not practical for consumers of its analyses to poll all the keys in a materialized view output table waiting for an anomaly to be flagged; consumers needed a direct signal only when anomalies happened for specific keys). For anomalous spikes (i.e., increases in query traffic), this is relatively straightforward: when the count for a given query exceeds the expected value in your model for that query by some statistically significant amount, you can signal an anomaly. But for anomalous dips (i.e., decreases in query traffic), the problem is a bit trickier. It’s not enough to simply see that the number of queries for a given search term has decreased, because for any period of time, the observed number always starts out at zero. What you really need to do in these cases is wait until you have reason to believe that you’ve seen a sufficiently representative portion of the input for a given time period, and only then compare the count against your model.

TRUE STREAMING
“True streaming use case” bears a bit of explanation. One recent trend in streaming systems is to try to simplify the programming models to make them more accessible by limiting the types of use cases one can address. For example, at the time of writing, both Spark’s Structured Streaming and Apache Kafka’s Kafka Streams systems limit themselves to what I refer to in Chapter 8 as “materialized view semantics,” essentially repeated updates to an eventually consistent output table. Materialized view semantics are great when you want to consume your output as a lookup table: any time you can just lookup a value in that table and be okay with the latest result as of query time, materialized views are a good fit. They are not, however, particularly well suited for use cases in which you want to consume your output as a bonafide stream. I refer to these as true streaming use cases, with anomaly detection being one of the better examples.

As we’ll discuss shortly, there are certain aspects of anomaly detection that make it unsuitable for pure materialized view semantics (i.e., record-by-record processing only), specifically the fact that it relies on reasoning about the completeness of the input data to accurately identify anomalies that are the result of an absence of data (in addition to the fact that polling an output table to see if an anomaly signal has arrived is not an approach that scales particularly well). True streaming use cases are thus the motivation for features like watermarks (Preferably low watermarks that pessimistically track input completeness, as described in Chapter 3, not high watermarks that track the event time of the newest record the system is aware of, as used by Spark Structured Streaming for garbage collecting windows, since high watermarks are more prone to incorrectly throwing away data as event time skew varies within the pipeline) and triggers. Systems that omit these features do so for the sake of simplicity but at the cost of decreased ability. There can be great value in that, most certainly, but don’t be fooled if you hear such systems claim these simplifications yield equivalent or even greater generality; you can’t address fewer use cases and be equally or more general.

The Zeitgeist pipeline first attempted to do this by inserting processing-time delays before the analysis logic that looked for dips. This would work reasonably decently when data arrived in order, but the pipeline’s authors discovered that data could, at times, be greatly delayed and thus arrive wildly out of order. In these cases, the processing-time delays they were using weren’t sufficient, because the pipeline would erroneously report a flurry of dip anomalies that didn’t actually exist. What they really needed was a way to wait until the input became complete.

Watermarks were thus born out of this need for reasoning about input completeness in out-of-order data. As Slava described in Chapter 3, the basic idea was to track the known progress of the inputs being provided to the system, using as much or as little data available for the given type of data source, to construct a progress metric that could be used to quantify input completeness. For simpler input sources like a statically partitioned Kafka topic with each partition being written to in increasing event-time order (such as by web frontends logging events in real time), you can compute a perfect watermark. For more complex input sources like a dynamic set of input logs, a heuristic might be the best you can do. But either way, watermarks provide a distinct advantage over the alternative of using processing time to reason about event-time completeness, which experience has shown serves about as well as a map of London while trying to navigate the streets of Cairo.

So thanks to the needs of its customers, MillWheel ended up as a system with the right set of features for supporting robust stream processing on out-of-order data. As a result, the paper titled “MillWheel: Fault-Tolerant Stream Processing at Internet Scale”8 (Figure 10-22) spends most of its time discussing the difficulties of providing correctness in a system like this, with consistency guarantees and watermarks being the main areas of focus. It’s well worth your time if you’re interested in the subject.


Figure 10-22. MillWheel paper
Not long after the MillWheel paper was published, MillWheel was integrated as an alternative, streaming backend for Flume, together often referred to as Streaming Flume. Within Google today, MillWheel is in the process of being replaced by its successor, Windmill (the execution engine that also powers Cloud Dataflow, discussed in a moment), a ground-up rewrite that incorporates all the best ideas from MillWheel, along with a few new ones like better scheduling and dispatch, and a cleaner separation of user and system code.

However, the big takeaway for MillWheel is that the four concepts listed earlier (exactly-once, persistent state, watermarks, persistent timers) together provided the basis for a system that was finally able to deliver on the true promise of stream processing: robust, low-latency processing of out-of-order data, even on unreliable commodity hardware.

Kafka
We now come to Kafka (Figure 10-23). Kafka is unique among the systems discussed in this chapter in that it’s not a data processing framework,9 but instead a transport layer. Make no mistake, however: Kafka has played one of the most influential roles in advancing stream processing out of all the system’s we’re discussing here.


Figure 10-23. Timeline: Kafka
If you’re not familiar with it, Kafka is essentially a persistent streaming transport, implemented as a set of partitioned logs. It was developed originally at LinkedIn by such industry luminaries as Neha Narkhede and Jay Kreps, and its accolades include the following:

Providing a clean model of persistence that packaged that warm fuzzy feeling of durable, replayable input sources from the batch world in a streaming friendly interface.

Providing an elastic isolation layer between producers and consumers.

Embodying the relationship between streams and tables that we discussed in Chapter 6, revealing a foundational way of thinking about data processing in general while also providing a conceptual link to the rich and storied world of databases.

As of side of effect of all of the above, not only becoming the cornerstone of a majority of stream processing installations across the industry, but also fostering the stream-processing-as-databases and microservices movements.

They must get up very early in the morning.

Of those accolades, there are two that stand out most to me. The first is the application of durability and replayability to stream data. Prior to Kafka, most stream processing systems used some sort of ephemeral queuing system like Rabbit MQ or even plain-old TCP sockets to send data around. Durability might be provided to some degree via upstream backup in the producers (i.e., the ability for upstream producers of data to resend if the downstream workers crashed), but oftentimes the upstream data was stored ephemerally, as well. And most approaches entirely ignored the idea of being able to replay input data later in cases of backfills or for prototyping, development, and regression testing.

Kafka changed all that. By taking the battle-hardened concept of a durable log from the database world and applying it to the realm of stream processing, Kafka gave us all back that sense of safety and security we’d lost when moving from the durable input sources common in the Hadoop/batch world to the ephemeral sources prevalent at the time in the streaming world. With durability and replayability, stream processing took yet another step toward being a robust, reliable replacement for the ad hoc, continuous batch processing systems of yore that were still being applied to streaming use cases.

As a streaming system developer, one of the more interesting visible artifacts of the impact that Kafka’s durability and replayability features have had on the industry is how many of the stream processing engines today have grown to fundamentally rely on that replayability to provide end-to-end exactly-once guarantees. Replayability is the foundation upon which end-to-end exactly-once guarantees in Apex, Flink, Kafka Streams, Spark, and Storm are all built. When executing in exactly-once mode, each of those systems assumes/requires that the input data source be able to rewind and replay all of the data up until the most recent checkpoint. When used with an input source that does not provide such ability (even if the source can guarantee reliable delivery via upstream backup), end-to-end exactly-once semantics fall apart. That sort of broad reliance on replayability (and the related aspect of durability) is a huge testament to the amount of impact those features have had across the industry.

The second noteworthy bullet from Kafka’s resume is the popularization of stream and table theory. We spent the entirety of Chapter 6 discussing streams and tables as well as much of Chapters 8 and 9. And for good reason. Streams and tables form the foundation of data processing, be it the MapReduce family tree of systems, the enormous legacy of SQL database systems, or what have you. Not all data processing approaches need speak directly in terms of streams and tables but conceptually speaking, that’s how they all operate. And as both users and developers of these systems, there’s great value in understanding the core underlying concepts that all of our systems build upon. We all owe a collective thanks to the folks in the Kafka community who helped shine a broader light on the streams-and-tables way of thinking.


Figure 10-24. I ❤ Logs
If you’d like to learn more about Kafka and the foundations it’s built on, I ❤ Logs by Jay Kreps (O’Reilly; Figure 10-24) is an excellent resource.10 Additionally, as cited originally in Chapter 6, Kreps and Martin Kleppmann have a pair of articles (Figure 10-25) that I highly recommend for reading up on the origins of streams and table theory.

Kafka has made huge contributions to the world of stream processing, arguably more than any other single system out there. In particular, the application of durability and replayability to input and output streams played a big part in helping move stream processing out of the niche realm of approximation tools and into the big leagues of general data processing. Additionally, the theory of streams and tables, popularized by the Kafka community, provides deep insight into the underlying mechanics of data processing in general. 


Figure 10-25. Martin’s post (left) and Jay’s post (right)
Cloud Dataflow
Cloud Dataflow (Figure 10-26) is Google’s fully managed, cloud-based data processing service. Dataflow launched to the world in August 2015. It was built with the intent to take the decade-plus of experiences that had gone into building MapReduce, Flume, and MillWheel, and package them up into a serverless cloud experience.


Figure 10-26. Timeline: Cloud Dataflow
Although the serverless aspect of Cloud Dataflow is perhaps its most technically challenging and distinguishing factor from a systems perspective, the primary contribution to streaming systems that I want to discuss here is its unified batch plus streaming programming model. That’s all the transformations, windowing, watermarks, triggers, and accumulation goodness we’ve spent most of the book talking about. And all of them, of course, wrapped up the what/where/when/how way of thinking about things.

The model first arrived back in Flume, as we looked to incorporate the robust out-of-order processing support in MillWheel into the higher-level programming model Flume afforded. The combined batch and streaming approach available to Googlers internally with Flume was then the basis for the fully unified model included in Dataflow.

The key insight in the unified model—the full extent of which none of us at the time even truly appreciated—is that under the covers, batch and streaming are really not that different: they’re both just minor variations on the streams and tables theme. As we learned in Chapter 6, the main difference really boils down to the ability to incrementally trigger tables into streams; everything else is conceptually the same.11 By taking advantage of the underlying commonalities of the two approaches, it was possible to provide a single, nearly seamless experience that applied to both worlds. This was a big step forward in making stream processing more accessible.

In addition to taking advantage of the commonalities between batch and streaming, we took a long, hard look at the variety of use cases we’d encountered over the years at Google and used those to inform the pieces that went into the unified model. Key aspects we targeted included the following:

Unaligned, event-time windows such as sessions, providing the ability to concisely express powerful analytic constructs and apply them to out-of-order data.

Custom windowing support, because one (or even three or four) sizes rarely fit all.

Flexible triggering and accumulation modes, providing the ability to shape the way data flow through the pipeline to match the correctness, latency, and cost needs of the given use case.

The use of watermarks for reasoning about input completeness, which is critical for use cases like anomalous dip detection where the analysis depends upon an absence of data.

Logical abstraction of the underlying execution environment, be it batch, microbatch, or streaming, providing flexibility of choice in execution engine and avoiding system-level constructs (such as micro-batch size) from creeping into the logical API.

Taken together, these aspects provided the flexibility to balance the tensions between correctness, latency, and cost, allowing the model to be applied across a wide breadth of use cases.


Figure 10-27. Dataflow Model paper
Given that you’ve just read an entire book covering the finer points of the Dataflow/Beam Model, there’s little point in trying to retread any those concepts here. However, if you’re looking for a slightly more academic take on things as well as a nice overview of some of the motivating use cases alluded to earlier, you might find our 2015 Dataflow Model paper worthwhile (Figure 10-27).

Though there are many other compelling aspects to Cloud Dataflow, the important contribution from the perspective of this chapter is its unified batch plus streaming programming model. It brought the world a comprehensive approach to tackling unbounded, out-of-order datasets, and in a way that provided the flexibility to make the trade-offs necessary to balance the tensions between correctness, latency, and cost to match the requirements for a given use case.

Flink
Flink (Figure 10-28) burst onto the scene in 2015, rapidly transforming itself from a system that almost no one had heard of into one of the powerhouses of the streaming world, seemingly overnight.


Figure 10-28. Timeline: Flink
There were two main reasons for Flink’s rise to prominence:

Its rapid adoption of the Dataflow/Beam programming model, which put it in the position of being the most semantically capable fully open source streaming system on the planet at the time.

Followed shortly thereafter by its highly efficient snapshotting implementation (derived from research in Chandy and Lamport’s original paper “Distributed Snapshots: Determining Global States of Distributed Systems” [Figure 10-29]), which gave it the strong consistency guarantees needed for correctness.


Figure 10-29. Chandy-Lamport snapshots
Reuven covered Flink’s consistency mechanism briefly in Chapter 5, but to reiterate, the basic idea is that periodic barriers are propagated along the communication paths between workers in the system. The barriers act as an alignment mechanism between the various distributed workers producing data upstream from a consumer. When the consumer receives a given barrier on all of its input channels (i.e., from all of its upstream producers), it checkpoints its current progress for all active keys, at which point it is then safe to acknowledge processing of all data that came before the barrier. By tuning how frequently barriers are sent through the system, it’s possible to tune the frequency of checkpointing and thus trade off increased latency (due to the need for side effects to be materialized only at checkpoint times) in exchange for higher throughput.


Figure 10-30. “Extending the Yahoo! Streaming Benchmark”
The simple fact that Flink now had the capability to provide exactly-once semantics along with native support for event-time processing was huge at the time. But it wasn’t until Jamie Grier published his article titled “Extending the Yahoo! Streaming Benchmark” (Figure 10-30) that it became clear just how performant Flink was. In that article, Jamie described two impressive achievements:

Building a prototype Flink pipeline that achieved greater accuracy than one of Twitter’s existing Storm pipelines (thanks to Flink’s exactly-once semantics) at 1% of the cost of the original.

Updating the Yahoo! Streaming Benchmark to show Flink (with exactly-once) achieving 7.5 times the throughput of Storm (without exactly-once). Furthermore, Flink’s performance was shown to be limited due to network saturation; removing the network bottleneck allowed Flink to achieve almost 40 times the throughput of Storm.

Since then, numerous other projects (notably, Storm and Apex) have all adopted the same type of consistency mechanism.


Figure 10-31. “Savepoints: Turning Back Time”
With the addition of a snapshotting mechanism, Flink gained the strong consistency needed for end-to-end exactly-once. But to its credit, Flink went one step further, and used the global nature of its snapshots to provide the ability to restart an entire pipeline from any point in the past, a feature known as savepoints (described in the “Savepoints: Turning Back Time” post by Fabian Hueske and Michael Winters [Figure 10-31]). The savepoints feature took the warm fuzziness of durable replay that Kafka had applied to the streaming transport layer and extended it to cover the breadth of an entire pipeline. Graceful evolution of a long-running streaming pipeline over time remains an important open problem in the field, with lots of room for improvement. But Flink’s savepoints feature stands as one of the first huge steps in the right direction, and one that remains unique across the industry as of this writing.

If you’re interested in learning more about the system constructs underlying Flink’s snapshots and savepoints, the paper “State Management in Apache Flink” (Figure 10-32) discusses the implementation in good detail.


Figure 10-32. “State Management in Apache Flink”
Beyond savepoints, the Flink community has continued to innovate, including bringing the first practical streaming SQL API to market for a large-scale, distributed stream processing engine, as we discussed in Chapter 8.

In summary, Flink’s rapid rise to stream processing juggernaut can be attributed primarily to three characteristics of its approach: 1) incorporating the best existing ideas from across the industry (e.g., being the first open source adopter of the Dataflow/Beam Model), 2) bringing its own innovations to the table to push forward the state of the art (e.g., strong consistency via snapshots and savepoints, streaming SQL), and 3) doing both of those things quickly and repeatedly. Add in the fact that all of this is done in open source, and you can see why Flink has consistently continued to raise the bar for streaming processing across the industry.

Beam
The last system we talk about is Apache Beam (Figure 10-33). Beam differs from most of the other systems in this chapter in that it’s primarily a programming model, API, and portability layer, not a full stack with an execution engine underneath. But that’s exactly the point: just as SQL acts as a lingua franca for declarative data processing, Beam aims to be the lingua franca for programmatic data processing. Let’s explore how.


Figure 10-33. Timeline: Beam
Concretely, Beam is composed a number of components:

A unified batch plus streaming programming model, inherited from Cloud Dataflow where it originated, and the finer points of which we’ve spent the majority of this book discussing. The model is independent of any language implementations or runtime systems. You can think of this as Beam’s equivalent to SQL’s relational algebra.

A set of SDKs (software development kits) that implement that model, allowing pipelines to be expressed in terms of the model in idiomatic ways for a given language. Beam currently provides SDKs in Java, Python, and Go. You can think of these as Beam’s programmatic equivalents to the SQL language itself.

A set of DSLs (domain specific languages) that build upon the SDKs, providing specialized interfaces that capture pieces of the model in unique ways. Whereas SDKs are required to surface all aspects of the model, DSLs can expose only those pieces that make sense for the specific domain a DSL is targeting. Beam currently provides a Scala DSL called Scio and an SQL DSL, both of which layer on top of the existing Java SDK.

A set of runners that can execute Beam pipelines. Runners take the logical pipeline described in Beam SDK terms, and translate them as efficiently as possible into a physical pipeline that they can then execute. Beam runners exist currently for Apex, Flink, Spark, and Google Cloud Dataflow. In SQL terms, you can think of these runners as Beam’s equivalent to the various SQL database implementations, such as Postgres, MySQL, Oracle, and so on.

The core vision for Beam is built around its value as a portability layer, and one of the more compelling features in that realm is its planned support for full cross-language portability. Though not yet fully complete (but landing imminently), the plan is for Beam to provide sufficiently performant abstraction layers between SDKs and runners that will allow for a full cross-product of SDK × runner matchups. In such a world, a pipeline written in a JavaScript SDK could seamlessly execute on a runner written in Haskell, even if the Haskell runner itself had no native ability to execute JavaScript code.

As an abstraction layer, the way that Beam positions itself relative to its runners is critical to ensure that Beam actually brings value to the community, rather than introducing just an unnecessary layer of abstraction. The key point here is that Beam aims to never be just the intersection (lowest common denominator) or union (kitchen sink) of the features found in its runners. Instead, it aims to include only the best ideas across the data processing community at large. This allows for innovation in two dimensions:

Innovation in Beam

Figure 10-34. Powerful and modular I/O
Beam might include API support for runtime features that not all runners initially support. This is okay. Over time, we expect many runners will incorporate such features into future versions; those that don’t will be a less-attractive runner choice for use cases that need such features.

An example here is Beam’s SplittableDoFn API for writing composable, scalable sources (described by Eugene Kirpichov in his post “Powerful and modular I/O connectors with Splittable DoFn in Apache Beam” [Figure 10-34]). It’s both unique and extremely powerful but also does not yet see broad support across all runners for some of the more innovative parts like dynamic work rebalancing. Given the value such features bring, however, we expect that will change over time.

Innovation in runners
Runners might introduce runtime features for which Beam does not initially provide API support. This is okay. Over time, runtime features that have proven their usefulness will have API support incorporated into Beam.

An example here is the state snapshotting mechanism in Flink, or savepoints, which we discussed earlier. Flink is still the only publicly available streaming system to support snapshots in this way, but there’s a proposal in Beam to provide an API around snapshots because we believe graceful evolution of pipelines over time is an important feature that will be valuable across the industry. If we were to magically push out such an API today, Flink would be the only runtime system to support it. But again, that’s okay. The point here is that the industry as a whole will begin to catch up over time as the value of these features becomes clear.12 And that’s better for everyone.

By encouraging innovation within both Beam itself as well as runners, we hope to push forward the capabilities of the entire industry at a greater pace over time, without accepting compromises along the way. And by delivering on the promise of portability across runtime execution engines, we hope to establish Beam as the common language for expressing programmatic data processing pipelines, similar to how SQL exists today as the common currency of declarative data processing. It’s an ambitious goal, and as of writing, we’re still a ways off from seeing it fully realized, but we’ve also come a long way so far.

Summary
We just took a whirlwind tour through a decade and a half of advances in data processing technology, with a focus on the contributions that made streaming systems what they are today. To summarize one last time, the main takeaways for each system were:

MapReduce—scalability and simplicity
By providing a simple set of abstractions for data processing on top of a robust and scalable execution engine, MapReduce allowed data engineers to focus on the business logic of their data processing needs rather than the gnarly details of building distributed systems resilient to the failure modes of commodity hardware.

Hadoop—open source ecosystem
By building an open source platform on the ideas of MapReduce, Hadoop created a thriving ecosystem that expanded well beyond the scope of its progenitor and allowed a multitude of new ideas to flourish.

Flume—pipelines, optimization
By coupling a high-level notion of logical pipeline operations with an intelligent optimizer, Flume made it possible to write clean and maintainable pipelines whose capabilities extended beyond the Map → Shuffle → Reduce confines of MapReduce, without sacrificing any of the performance theretofore gained by contorting the logical pipeline via hand-tuned manual optimizations.

Storm—low latency with weak consistency
By sacrificing correctness of results in favor of decreased latency, Storm brought stream processing to the masses and also ushered in the era of the Lambda Architecture, where weakly consistent stream processing engines were run alongside strongly consistent batch systems to realize the true business goal of low-latency, eventually consistent results.

Spark—strong consistency
By utilizing repeated runs of a strongly consistent batch engine to provide continuous processing of unbounded datasets, Spark Streaming proved it possible to have both correctness and low-latency results, at least for in-order datasets.

MillWheel—out-of-order processing
By coupling strong consistency and exactly-once processing with tools for reasoning about time like watermarks and timers, MillWheel conquered the challenge of robust stream processing over out-of-order data.

Kafka—durable streams, streams and tables
By applying the concept of a durable log to the problem of streaming transports, Kafka brought back the warm, fuzzy feeling of replayability that had been lost by ephemeral streaming transports like RabbitMQ and TCP sockets. And by popularizing the ideas of stream and table theory, it helped shed light on the conceptual underpinnings of data processing in general.

Cloud Dataflow—unified batch plus streaming
By melding the out-of-order stream processing concepts from MillWheel with the logical, automatically optimizable pipelines of Flume, Cloud Dataflow provided a unified model for batch plus streaming data processing that provided the flexibility to balance the tensions between correctness, latency, and cost to match any given use case.

Flink—open source stream processing innovator
By rapidly bringing the power of out-of-order processing to the world of open source and combining it with innovations of their own like distributed snapshots and its related savepoints features, Flink raised the bar for open source stream processing and helped lead the current charge of stream processing innovation across the industry.

Beam—portability
By providing a robust abstraction layer that incorporates the best ideas from across the industry, Beam provides a portability layer positioned as the programmatic equivalent to the declarative lingua franca provided by SQL, while also encouraging the adoption of innovative new ideas throughout the industry.

To be certain, these 10 projects and the sampling of their achievements that I’ve highlighted here do not remotely encompass the full breadth of the history that has led the industry to where it exists today. But they stand out to me as important and noteworthy milestones along the way, which taken together paint an informative picture of the evolution of stream processing over the past decade and a half. We’ve come a long way since the early days of MapReduce, with a number of ups, downs, twists, and turns along the way. Even so, there remains a long road of open problems ahead of us in the realm of streaming systems. I’m excited to see what the future holds.

1 Which means I’m skipping a ton of the academic literature around stream processing, because that’s where much of it started. If you’re really into hardcore academic papers on the topic, start from the references in “The Dataflow Model” paper and work backward. You should be able to find your way pretty easily.

2 Certainly, MapReduce itself was built upon many ideas that had been well known before, as is even explicitly stated in the MapReduce paper. That doesn’t change the fact that MapReduce was the system that tied those ideas together (along with some of its own) to create something practical that solved an important and emerging problem better than anyone else before ever had, and in a way that inspired generations of data-processing systems that followed.

3 To be clear, Google was most certainly not the only company tackling data processing problems at this scale at the time. Google was just one among a number of companies involved in that first generation of attempts at taming massive-scale data processing.

4 And to be clear, MapReduce actually built upon the Google File System, GFS, which itself solved the scalability and fault-tolerance issues for a specific subset of the overall problem.

5 Not unlike the query optimizers long used in the database world.

6 Noogler == New + Googler == New hires at Google

7 As an aside, I also highly recommend reading Martin Kleppmann’s “A Critique of the CAP Theorem” for very nice analysis of the shortcomings of the CAP theorem itself, as well as a more principled alternative way of looking at the same problem.

8 For the record, written primarily by Sam McVeety with help from Reuven and bits of input from the rest of us on the author list; we shouldn’t have alphabetized that author list, because everyone always assumes I’m the primary author on it, even though I wasn’t.

9 Kafka Streams and now KSQL are of course changing that, but those are relatively recent developments, and I’ll be focusing primarily on the Kafka of yore.

10 While I recommend the book as the most comprehensive and cohesive resource, you can find much of the content from it scattered across O’Reilly’s website if you just search around for Kreps’ articles. Sorry, Jay...

11 As with many broad generalizations, this one is true in a specific context, but belies the underlying complexity of reality. As I alluded to in Chapter 1, batch systems go to great lengths to optimize the cost and runtime of data processing pipelines over bounded datasets in ways that stream processing engines have yet to attempt to duplicate. To imply that modern batch and streaming systems only differ in one small way is a sizeable oversimplification in any realm beyond the purely conceptual.

12 There’s an additional subtlety here that’s worth calling out: even as runners adopt new semantics and tick off feature checkboxes, it’s not the case that you can blindly choose any runner and have an identical experience. This is because the runners themselves can still vary greatly in their runtime and operational characteristics. Even for cases in which two given runners implement the same set of semantic features within the Beam Model, the way they go about executing those features at runtime is typically very different. As a result, when building a Beam pipeline, it’s important to do your homework regarding various runners, to ensure that you choose a runtime platform that serves your use case best.