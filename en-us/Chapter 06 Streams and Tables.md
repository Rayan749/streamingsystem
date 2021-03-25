# Chapter 6. Streams and Tables
You have reached the part of the book where we talk about streams and tables. If you recall, back in Chapter 1, we briefly discussed two important but orthogonal dimensions of data: cardinality and constitution. Until now, we’ve focused strictly on the cardinality aspects (bounded versus unbounded) and otherwise ignored the constitution aspects (stream versus table). This has allowed us to learn about the challenges brought to the table by the introduction of unbounded datasets, without worrying too much about the lower-level details that really drive the way things work. We’re now going to expand our horizons and see what the added dimension of constitution brings to the mix.

Though it’s a bit of a stretch, one way to think about this shift in approach is to compare the relationship of classical mechanics to quantum mechanics. You know how in physics class they teach you a bunch of classical mechanics stuff like Newtonian theory and so on, and then after you think you’ve more or less mastered that, they come along and tell you it was all bunk, and classical physics gives you only part of the picture, and there’s actually this other thing called quantum mechanics that really explains how things work at a lower level, but it didn’t make sense to complicate matters up front by trying to teach you both at once, and...oh wait...we also haven’t fully reconciled everything between the two yet, so just squint at it and trust us that it all makes sense somehow? Well this is a lot like that, except your brain will hurt less because physics is way harder than data processing, and you won’t have to squint at anything and pretend it makes sense because it actually does come together beautifully in the end, which is really cool.

So, with the stage appropriately set, the point of this chapter is twofold:

To try to describe the relationship between the Beam Model (as we’ve described it in the book up to this point) and the theory of “streams and tables” (as popularized by Martin Kleppmann and Jay Kreps, among others, but essentially originating out of the database world). It turns out that stream and table theory does an illuminating job of describing the low-level concepts that underlie the Beam Model. Additionally, a clear understanding of how they relate is particularly informative when considering how robust stream processing concepts might be cleanly integrated into SQL (something we consider in Chapter 8).

To bombard you with bad physics analogies for the sheer fun of it. Writing a book is a lot of work; you have to find little joys here and there to keep you going.

Stream-and-Table Basics Or: a Special Theory of Stream and Table Relativity
The basic idea of streams and tables derives from the database world. Anyone familiar with SQL is likely familiar with tables and their core properties, roughly summarized as: tables contain rows and columns of data, and each row is uniquely identified by some sort of key, either explicit or implicit.

If you think back to your database systems class in college,1 you’ll probably recall the data structure underlying most databases is an append-only log. As transactions are applied to a table in the database, those transactions are recorded in a log, the contents of which are then serially applied to the table to materialize those updates. In streams and tables nomenclature, that log is effectively the stream.

From that perspective, we now understand how to create a table from a stream: the table is just the result of applying the transaction log of updates found in the stream. But how to do we create a stream from a table? It’s essentially the inverse: a stream is a changelog for a table. The motivating example typically used for table-to-stream conversion is materialized views. Materialized views in SQL let you specify a query on a table, which itself is then manifested by the database system as another first-class table. This materialized view is essentially a cached version of that query, which the database system ensures is always up to date as the contents of the source table evolve over time. Perhaps unsurprisingly, materialized views are implemented via the changelog for the original table; any time the source table changes, that change is logged. The database then evaluates that change within the context of the materialized view’s query and applies any resulting change to the destination materialized view table.

Combining these two points together and employing yet another questionable physics analogy, we arrive at what one might call the Special Theory of Stream and Table Relativity:

Streams → tables
The aggregation of a stream of updates over time yields a table.

Tables → streams
The observation of changes to a table over time yields a stream.

This is a very powerful pair of concepts, and their careful application to the world of stream processing is a big reason for the massive success of Apache Kafka, the ecosystem that is built around these underlying principles. However, those statements themselves are not quite general enough to allow us to tie streams and tables to all of the concepts in the Beam Model. For that, we must go a little bit deeper.

Toward a General Theory of Stream and Table Relativity
If we want to reconcile stream/table theory with everything we know of the Beam Model, we’ll need to tie up some loose ends, specifically:

How does batch processing fit into all of this?

What is the relationship of streams to bounded and unbounded datasets?

How do the four what, where, when, how questions map onto a streams/tables world?

As we attempt to do so, it will be helpful to have the right mindset about streams and tables. In addition to understanding them in relation to each other, as captured by the previous definition, it can be illuminating to define them independent of each other. Here’s a simple way of looking at it that will underscore some of our future analyses:

Tables are data at rest.

This isn’t to say tables are static in any way; nearly all useful tables are continuously changing over time in some way. But at any given time, a snapshot of the table provides some sort of picture of the dataset contained together as a whole.2 In that way, tables act as a conceptual resting place for data to accumulate and be observed over time. Hence, data at rest.

Streams are data in motion.

Whereas tables capture a view of the dataset as a whole at a specific point in time, streams capture the evolution of that data over time. Julian Hyde is fond of saying streams are like the derivatives of tables, and tables the integrals of streams, which is a nice way of thinking about it for you math-minded individuals out there. Regardless, the important feature of streams is that they capture the inherent movement of data within a table as it changes. Hence, data in motion.

Though tables and streams are intimately related, it’s important to keep in mind that they are very much not the same thing, even if there are many cases in which one might be fully derived from the other. The differences are subtle but important, as we’ll see.

Batch Processing Versus Streams and Tables
With our proverbial knuckles now cracked, let’s start to tie up some loose ends. To begin, we tackle the first one, regarding batch processing. At the end, we’ll discover that the resolution to the second issue, regarding the relationship of streams to bounded and unbounded data, will fall out naturally from the answer for the first. Score one for serendipity.

A Streams and Tables Analysis of MapReduce
To keep our analysis relatively simple, but solidly concrete, as it were, let’s look at how a traditional MapReduce job fits into the streams/tables world. As alluded to by its name, a MapReduce job superficially consists of two phases: Map and Reduce. For our purposes, though, it’s useful to look a little deeper and treat it more like six:

MapRead
This consumes the input data and preprocesses them a bit into a standard key/value form for mapping.

Map
This repeatedly (and/or in parallel) consumes a single key/value pair3 from the preprocessed input and outputs zero or more key/value pairs.

MapWrite
This clusters together sets of Map-phase output values having identical keys and writes those key/value-list groups to (temporary) persistent storage. In this way, the MapWrite phase is essentially a group-by-key-and-checkpoint operation.

ReduceRead
This consumes the saved shuffle data and converts them into a standard key/value-list form for reduction.

Reduce
This repeatedly (and/or in parallel) consumes a single key and its associated value-list of records and outputs zero or more records, all of which may optionally remain associated with that same key.

ReduceWrite
This writes the outputs from the Reduce phase to the output datastore.

Note that the MapWrite and ReduceRead phases sometimes are referred to in aggregate as the Shuffle phase, but for our purposes, it’s better to consider them independently. It’s perhaps also worth noting that the functions served by the MapRead and ReduceWrite phases are more commonly referred to these days as sources and sinks. Digressions aside, however, let’s now see how this all relates to streams and tables.

Map as streams/tables
Because we start and end with static4 datasets, it should be clear that we begin with a table and end with a table. But what do we have in between? Naively, one might assume that it’s tables all the way down; after all, batch processing is (conceptually) known to consume and produce tables. And if you think of a batch processing job as a rough analog of executing a classic SQL query, that feels relatively natural. But let’s look a little more closely at what’s really happening, step by step.

First up, MapRead consumes a table and produces something. That something is consumed next by the Map phase, so if we want to understand its nature, a good place to start would be with the Map phase API, which looks something like this in Java:

void map(KI key, VI value, Emit<KO, VO> emitter);
The map call will be repeatedly invoked for each key/value pair in the input table. If you think this sounds suspiciously like the input table is being consumed as a stream of records, you’d be right. We look more closely at how the table is being converted into a stream later, but for now, suffice it to say that the MapRead phase is iterating over the data at rest in the input table and putting them into motion in the form of a stream that is then consumed by the Map phase.

Next up, the Map phase consumes that stream, and then does what? Because the map operation is an element-wise transformation, it’s not doing anything that will halt the moving elements and put them to rest. It might change the effective cardinality of the stream by either filtering some elements out or exploding some elements into multiple elements, but those elements all remain independent from one another after the Map phase concludes. So, it seems safe to say that the Map phase both consumes a stream as well as produces a stream.

After the Map phase is done, we enter the MapWrite phase. As I noted earlier, the MapWrite groups records by key and then writes them in that format to persistent storage. The persistent part of the write actually isn’t strictly necessary at this point as long as there’s persistence somewhere (i.e., if the upstream inputs are saved and one can recompute the intermediate results from them in cases of failure, similar to the approach Spark takes with Resilient Distributed Datasets [RDDs]). What is important is that the records are grouped together into some kind of datastore, be it in memory, on disk, or what have you. This is important because, as a result of this grouping operation, records that were previously flying past one-by-one in the stream are now brought to rest in a location dictated by their key, thus allowing per-key groups to accumulate as their like-keyed brethren and sistren arrive. Note how similar this is to the definition of stream-to-table conversion provided earlier: the aggregation of a stream of updates over time yields a table. The MapWrite phase, by virtue of grouping the stream of records by their keys, has put those data to rest and thus converted the stream back into a table.5 Cool!

We’re now halfway through the MapReduce, so, using Figure 6-1, let’s recap what we’ve seen so far.

We’ve gone from table to stream and back again across three operations. MapRead converted the table into a stream, which was then transformed into a new stream by Map (via the user’s code), which was then converted back into a table by MapWrite. We’re going to find that the next three operations in the MapReduce look very similar, so I’ll go through them more quickly, but I still want to point out one important detail along the way.


Figure 6-1. Map phases in a MapReduce. Data in a table are converted to a stream and back again.
Reduce as streams/tables
Picking up where we left off after the MapWrite phase, ReduceRead itself is relatively uninteresting. It’s basically identical to MapRead, except that the values being read are singleton lists of values instead of singleton values, because the data stored by MapWrite were key/value-list pairs. But it’s still just iterating over a snapshot of a table to convert it into a stream. Nothing new here.

And even though it sounds like it might be interesting, Reduce in this context is really just a glorified Map phase that happens to receive a list of values for each key instead of a single value. So it’s still just mapping single (composite) records into zero or more new records. Nothing particularly new here, either.

ReduceWrite is the one that’s a bit noteworthy. We know already that this phase must convert a stream to a table, given that Reduce produces a stream and the final output is a table. But how does that happen? If I told you it was a direct result of key-grouping the outputs from the previous phase into persistent storage, just like we saw with MapWrite, you might believe me, until you remembered that I noted earlier that key-association was an optional feature of the Reduce phase. With that feature enabled, ReduceWrite is essentially identical to MapWrite.6 But if that feature is disabled and the outputs from Reduce have no associated keys, what exactly is happening to bring those data to rest?

To understand what’s going on, it’s useful to think again of the semantics of a SQL table. Though often recommended, it’s not strictly required for a SQL table to have a primary key uniquely identifying each row. In the case of keyless tables, each row that is inserted is considered to be a new, independent row (even if the data therein are identical to one or more extant rows in the table), much as though there were an implicit AUTO_INCREMENT field being used as the key (which incidentally, is what’s effectively happening under the covers in most implementations, even though the “key” in this case might just be some physical block location that is never exposed or expected to be used as a logical identifier). This implicit unique key assignment is precisely what’s happening in ReduceWrite with unkeyed data. Conceptually, there’s still a group-by-key operation happening; that’s what brings the data to rest. But lacking a user-supplied key, the ReduceWrite is treating each record as though it has a new, never-before-seen key, and effectively grouping each record with itself, resulting again in data at rest.7

Take a look at Figure 6-2, which shows the entire pipeline from the perspective of stream/tables. You can see that it’s a sequence of TABLE → STREAM → STREAM → TABLE → STREAM → STREAM → TABLE. Even though we’re processing bounded data and even though we’re doing what we traditionally think of as batch processing, it’s really just streams and tables under the covers.


Figure 6-2. Map and Reduce phases in a MapReduce, viewed from the perspective of streams and tables
Reconciling with Batch Processing
So where does this leave us with respect to our first two questions?

Q: How does batch processing fit into stream/table theory?

A: Quite nicely. The basic pattern is as follows:

Tables are read in their entirety to become streams.

Streams are processed into new streams until a grouping operation is hit.

Grouping turns the stream into a table.

Steps a through c repeat until you run out of stages in the pipeline.

Q: How do streams relate to bounded/unbounded data?

A: As we can see from the MapReduce example, streams are simply the in-motion form of data, regardless of whether they’re bounded or unbounded.

Taken from this perspective, it’s easy to see that stream/table theory isn’t remotely at odds with batch processing of bounded data. In fact, it only further supports the idea I’ve been harping on that batch and streaming really aren’t that different: at the end of the of day, it’s streams and tables all the way down.

With that, we’re well on our way toward a general theory of streams and tables. But to wrap things up cleanly, we last need to revisit the four what/where/when/how questions within the streams/tables context, to see how they all relate.

What, Where, When, and How in a Streams and Tables World
In this section, we look at each of the four questions and see how they relate to streams and tables. We’ll also answer any questions that may be lingering from the previous section, one big one being: if grouping is the thing that brings data to rest, what precisely is the “ungrouping” inverse that puts them in motion? More on that later. But for now, on to transformations.

What: Transformations
In Chapter 3, we learned that transformations tell us what the pipeline is computing; that is, whether it’s building models, counting sums, filtering spam, and so on. We saw in the earlier MapReduce example that four of the six stages answered what questions:

Map and Reduce both applied the pipeline author’s element-wise transformation on each key/value or key/value-list pair in the input stream, respectively, yielding a new, transformed stream.

MapWrite and ReduceWrite both grouped the outputs from the previous stage according to the key assigned by that stage (possibly implicitly, in the optional Reduce case), and in doing so transformed the input stream into an output table.

Viewed in that light, you can see that there are essentially two types of what transforms from the perspective of stream/table theory:

Nongrouping
These operations (as we saw in Map and Reduce) simply accept a stream of records and produce a new, transformed stream of records on the other side. Examples of nongrouping transformations are filters (e.g., removing spam messages), exploders (i.e., splitting apart a larger composite record into its constituent parts), and mutators (e.g., divide by 100), and so on.

Grouping
These operations (as we saw in MapWrite and ReduceWrite) accept a stream of records and group them together in some way, thereby transforming the stream into a table. Examples of grouping transformations are joins, aggregations, list/set accumulation, changelog application, histogram creation, machine learning model training, and so forth.

To get a better sense for how all of this ties together, let’s look at an updated version of Figure 2-2, where we first began to look at transformations. To save you jumping back there to see what we were talking about, Example 6-1 contains the code snippet we were using.

Example 6-1. Summation pipeline
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals =
  input.apply(Sum.integersPerKey());
This pipeline is simply reading in input data, parsing individual team member scores, and then summing those scores per team. The event-time/processing-time visualization of it looks like the diagram presented in Figure 6-3.

Figure 6-3. Event-time/processing-time view of classic batch processing
Figure 6-4 depicts a more topological view of this pipeline over time, rendered from a streams-and-tables perspective.

Figure 6-4. Streams and tables view of classic batch processing
In the streams and tables version of this visualization, the passage of time is manifested by scrolling the graph area downward in the processing-time dimension (y-axis) as time advances. The nice thing about rendering things this way is that it very clearly calls out the difference between nongrouping and grouping operations. Unlike our previous diagrams, in which I elided all initial transformations in the pipeline other than the Sum.integersByKey, I’ve included the initial parsing operation here, as well, because the nongrouping aspect of the parsing operation provides a nice contrast to the grouping aspect of the summation. Viewed in this light, it’s very easy to see the difference between the two. The nongrouping operation does nothing to halt the motion of the elements in the stream, and as a result yields another stream on the other side. In contrast, the grouping operation brings all the elements in the stream to rest as it adds them together into the final sum. Because this example was running on a batch processing engine over bounded data, the final results are emitted only after the end of the input is reached. As we noted in Chapter 2 this example is sufficient for bounded data, but is too limiting in the context of unbounded data because the input will theoretically never end. But is it really insufficient?

Looking at the new streams/tables portion of the diagram, if all we’re doing is calculating sums as our final results (and not actually transforming those sums in any additional way further downstream within the pipeline), the table we created with our grouping operation has our answer sitting right there, evolving over time as new data arrive. Why don’t we just read our results from there?

This is exactly the point being made by the folks championing stream processors as a database8 (primarily the Kafka and Flink crews): anywhere you have a grouping operation in your pipeline, you’re creating a table that includes what is effectively the output values of that portion of the stage. If those output values happen to be the final thing your pipeline is calculating, you don’t need to rematerialize them somewhere else if you can read them directly out of that table. Besides providing quick and easy access to results as they evolve over time, this approach saves on compute resources by not requiring an additional sink stage in the pipeline to materialize the outputs, yields disk savings by eliminating redundant data storage, and obviates the need for any engineering work building the aforementioned sink stages.9 The only major caveat is that you need to take care to ensure that only the data processing pipeline has the ability to make modifications to the table. If the values in the table can change out from under the pipeline due to external modification, all bets are off regarding consistency guarantees.

A number of folks in the industry have been recommending this approach for a while now, and it’s being put to great use in a variety of scenarios. We’ve seen MillWheel customers within Google do the same thing by serving data directly out of their Bigtable-based state tables, and we’re in the process of adding first-class support for accessing state from outside of your pipeline in the C++–based Apache Beam equivalent we use internally at Google (Google Flume); hopefully those concepts will make their way to Apache Beam proper someday soon, as well.

Now, reading from the state tables is great if the values therein are your final results. But, if you have more processing to perform downstream in the pipeline (e.g., imagine our pipeline was actually computing the top scoring team), we still need some better way to cope with unbounded data, allowing us to transform the table back into a stream in a more incremental fashion. For that, we’ll want to journey back through the remaining three questions, beginning with windowing, expanding into triggering, and finally tying it all together with accumulation.

Where: Windowing
As we know from Chapter 3, windowing tells us where in event time grouping occurs. Combined with our earlier experiences, we can thus also infer it must play a role in stream-to-table conversion because grouping is what drives table creation. There are really two aspects of windowing that interact with stream/table theory:

Window assignment
This effectively just means placing a record into one or more windows.

Window merging
This is the logic that makes dynamic, data-driven types of windows, such as sessions, possible.

The effect of window assignment is quite straightforward. When a record is conceptually placed into a window, the definition of the window is essentially combined with the user-assigned key for that record to create an implicit composite key used at grouping time.10 Simple.

For completeness, let’s take another look at the original windowing example from Chapter 3, but from a streams and tables perspective. If you recall, the code snippet looked something like Example 6-2 (with parsing not elided this time).

Example 6-2. Summation pipeline
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES)))
  .apply(Sum.integersPerKey());
And the original visualization looked like that shown in Figure 6-5.

Figure 6-5. Event-time/processing-time view of windowed summation on a batch engine
And now, Figure 6-6 shows the streams and tables version.

Figure 6-6. Streams and tables view of windowed summation on a batch engine
As you might expect, this looks remarkably similar to Figure 6-4, but with four groupings in the table (corresponding to the four windows occupied by the data) instead of just one. But as before, we must wait until the end of our bounded input is reached before emitting results. We look at how to address this for unbounded data in the next section, but first let’s touch briefly on merging windows.

Window merging
Moving on to merging, we’ll find that the effect of window merging is more complicated than window assignment, but still straightforward when you think about the logical operations that would need to happen. When grouping a stream into windows that can merge, that grouping operation has to take into account all of the windows that could possibly merge together. Typically, this is limited to windows whose data all have the same key (because we’ve already established that windowing modifies grouping to not be just by key, but also key and window). For this reason, the system doesn’t really treat the key/window pair as a flat composite key, but rather as a hierarchical key, with the user-assigned key as the root, and the window a child component of that root. When it comes time to actually group data together, the system first groups by the root of the hierarchy (the key assigned by the user). After the data have been grouped by key, the system can then proceed with grouping by window within that key (using the child components of the hierarchical composite keys). This act of grouping by window is where window merging happens.

What’s interesting from a streams and tables perspective is how this window merging changes the mutations that are ultimately applied to a table; that is, how it modifies the changelog that dictates the contents of the table over time. With nonmerging windows, each new element being grouped results in a single mutation to the table (to add that element to the group for the element’s key+window). With merging windows, the act of grouping a new element can result in one or more existing windows being merged with the new window. So, the merging operation must inspect all of the existing windows for the current key, figure out which windows can merge with this new window, and then atomically commit deletes for the old unmerged windows in conjunction with an insert for the new merged window into the table. This is why systems that support merging windows typically define the unit of atomicity/parallelization as key, rather than key+window. Otherwise, it would be impossible (or at least much more expensive) to provide the strong consistency needed for correctness guarantees. When you begin to look at it in this level of detail, you can see why it’s so nice to have the system taking care of the nasty business of dealing with window merges. For an even closer view of window merging semantics, I refer you to section 2.2.2 of “The Dataflow Model”.

At the end of the day, windowing is really just a minor alteration to the semantics of grouping, which means it’s a minor alteration to the semantics of stream-to-table conversion. For window assignment, it’s as simple as incorporating the window into an implicit composite key used at grouping time. When window merging becomes involved, that composite key is treated more like a hierarchical key, allowing the system to handle the nasty business of grouping by key, figuring out window merges within that key, and then atomically applying all the necessary mutations to the corresponding table for us. Hooray for layers of abstraction!

All that said, we still haven’t actually addressed the problem of converting a table to a stream in a more incremental fashion in the case of unbounded data. For that, we need to revisit triggers.

When: Triggers
We learned in Chapter 3 that we use triggers to dictate when the contents of a window will be materialized (with watermarks providing a useful signal of input completeness for certain types of triggers). After data have been grouped together into a window, we use triggers to dictate when that data should be sent downstream. In streams/tables terminology, we understand that grouping means stream-to-table conversion. From there, it’s a relatively small leap to see that triggers are the complement to grouping; in other words, that “ungrouping” operation we were grasping for earlier. Triggers are what drive table-to-stream conversion.

In streams/tables terminology, triggers are special procedures applied to a table that allow for data within that table to be materialized in response to relevant events. Stated that way, they actually sound suspiciously similar to classic database triggers. And indeed, the choice of name here was no coincidence; they are essentially the same thing. When you specify a trigger, you are in effect writing code that then is evaluated for every row in the state table as time progresses. When that trigger fires, it takes the corresponding data that are currently at rest in the table and puts them into motion, yielding a new stream.

Let’s return to our examples. We’ll begin with the simple per-record trigger from Chapter 2, which simply emits a new result every time a new record arrives. The code and event-time/processing-time visualization for that example is shown in Example 6-3. Figure 6-7 presents the results.

Example 6-3. Triggering repeatedly with every record
PCollection<String>> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());	
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(Repeatedly(AfterCount(1))));
  .apply(Sum.integersPerKey());
Figure 6-7. Streams and tables view of windowed summation on a batch engine
As before, new results are materialized every time a new record is encountered. Rendered in a streams and tables type of view, this diagram would look like Figure 6-8.

Figure 6-8. Streams and tables view of windowed summation with per-record triggering on a streaming engine
An interesting side effect of using per-record triggers is how it somewhat masks the effect of data being brought to rest, given that they are then immediately put back into motion again by the trigger. Even so, the aggregate artifact from the grouping remains at rest in the table, as the ungrouped stream of values flows away from it.

To get a better sense of the at-rest/in-motion relationship, let’s skip forward in our triggering examples to the basic watermark completeness streaming example from Chapter 2, which simply emitted results when complete (due to the watermark passing the end of the window). The code and event-time/processing-time visualization for that example are presented in Example 6-4 (note that I’m only showing the heuristic watermark version here, for brevity and ease of comparison) and Figure 6-9 illustrates the results.

Example 6-4. Watermark completeness trigger
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(AfterWatermark()))
  .apply(Sum.integersPerKey());
Figure 6-9. Event-time/processing-time view of windowed summation with a heuristic watermark on a streaming engine
Thanks to the trigger specified in Example 6-4, which declares that windows should be materialized when the watermark passes them, the system is able to emit results in a progressive fashion as the otherwise unbounded input to the pipeline becomes more and more complete. Looking at the streams and tables version in Figure 6-10, it looks as you might expect.

Figure 6-10. Streams and tables view of windowed summation with a heuristic watermark on a streaming engine
In this version, you can see very clearly the ungrouping effect triggers have on the state table. As the watermark passes the end of each window, it pulls the result for that window out of the table and sets it in motion downstream, separate from all the other values in the table. We of course still have the late data issue from before, which we can solve again with the more comprehensive trigger shown in Example 6-5.

Example 6-5. Early, on-time, and late firings via the early/on-time/late API
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(
                 AfterWatermark()
                   .withEarlyFirings(AlignedDelay(ONE_MINUTE))
                   .withLateFirings(AfterCount(1))))
  .apply(Sum.integersPerKey());
The event-time/processing-time diagram looks like Figure 6-11.

Figure 6-11. Event-time/processing-time view of windowed summation on a streaming engine with early/on-time/late trigger
Whereas the streams and tables version looks like that shown in Figure 6-12.

Figure 6-12. Streams and tables view of windowed summation on a streaming engine with early/on-time/late trigger
This version makes even more clear the ungrouping effect triggers have, rendering an evolving view of the various independent pieces of the table into a stream, as dictated by the triggers specified in Example 6-6.

The semantics of all the concrete triggers we’ve talked about so far (event-time, processing-time, count, composites like early/on-time/late, etc.) are just as you would expect when viewed from the streams/tables perspective, so they aren’t worth further discussion. However, we haven’t yet spent much time talking about what triggers look like in a classic batch processing scenario. Now that we understand what the underlying streams/tables topology of a batch pipeline looks like, this is worth touching upon briefly.

At the end of the day, there’s really only one type of trigger used in classic batch scenarios: one that fires when the input is complete. For the initial MapRead stage of the MapReduce job we looked at earlier, that trigger would conceptually fire for all of the data in the input table as soon as the pipeline launched, given that the input for a batch job is assumed to be complete from the get go.11 That input source table would thus be converted into a stream of individual elements, after which the Map stage could begin processing them.

For table-to-stream conversions in the middle of the pipeline, such as the ReduceRead stage in our example, the same type of trigger is used. In this case, however, the trigger must actually wait for all of the data in the table to be complete (i.e., what is more commonly referred to as all of the data being written to the shuffle), much as our example batch pipelines in Figures 6-4 and 6-6 waited for the end of the input before emitting their final results.

Given that classic batch processing effectively always makes use of the input-data-complete trigger, you might ask what any custom triggers specified by the author of the pipeline might mean in a batch scenario. The answer here really is: it depends. There are two aspects worth discussing:

Trigger guarantees (or lack thereof)
Most existing batch processing systems have been designed with this lock-step read-process-group-write-repeat sequence in mind. In such circumstances, it’s difficult to provide any sort of finer-grained trigger abilities, because the only place they would manifest any sort of change would be at the final shuffle stage of the pipeline. This doesn’t mean that the triggers specified by the user aren’t honored, however; the semantics of triggers are such that it’s possible to resort to lower common denominators when appropriate.

For example, an AfterWatermark trigger is meant to trigger after the watermark passes the end of a window. It makes no guarantees how far beyond the end of the window the watermark may be when it fires. Similarly, an AfterCount(N) trigger only guarantees that at least N elements have been processed before triggering; N might very well be all of the elements in the input set.

Note that this clever wording of trigger names wasn’t chosen simply to accommodate classic batch systems within the model; it’s a very necessary part of the model itself, given the natural asynchronicity and nondeterminism of triggering. Even in a finely tuned, low-latency, true-streaming system, it’s essentially impossible to guarantee that an AfterWatermark trigger will fire while the watermark is precisely at the end of any given window, except perhaps under the most extremely limited circumstances (e.g., a single machine processing all of the data for the pipeline with a relatively modest load). And even if you could guarantee it, what really would be the point? Triggers provide a means of controlling the flow of data from a table into a stream, nothing more.

The blending of batch and streaming
Given what we’ve learned in this writeup, it should be clear that the main semantic difference between batch and streaming systems is the ability to trigger tables incrementally. But even that isn’t really a semantic difference, but more of a latency/throughput trade-off (because batch systems typically give you higher throughput at the cost of higher latency of results).

This goes back to something I said in “Batch and Streaming Efficiency Differences”: there’s really not that much difference between batch and streaming systems today except for an efficiency delta (in favor of batch) and a natural ability to deal with unbounded data (in favor of streaming). I argued then that much of that efficiency delta comes from the combination of larger bundle sizes (an explicit compromise of latency in favor of throughput) and more efficient shuffle implementations (i.e., stream → table → stream conversions). From that perspective, it should be possible to provide a system that seamlessly integrates the best of both worlds: one which provides the ability to handle unbounded data naturally but can also balance the tensions between latency, throughput, and cost across a broad spectrum of use cases by transparently tuning the bundle sizes, shuffle implementations, and other such implementation details under the covers.

This is precisely what Apache Beam already does at the API level.12 The argument being made here is that there’s room for unification at the execution-engine level, as well. In a world like that, batch and streaming will no longer be a thing, and we’ll be able to say goodbye to both batch and streaming as independent concepts once and for all. We’ll just have general data processing systems that combine the best ideas from both branches in the family tree to provide an optimal experience for the specific use case at hand. Some day.

At this point, we can stick a fork in the trigger section. It’s done. We have only one more brief stop on our way to having a holistic view of the relationship between the Beam Model and streams-and-tables theory: accumulation.

How: Accumulation
In Chapter 2, we learned that the three accumulation modes (discarding, accumulating, accumulating and retracting13) tell us how refinements of results relate when a window is triggered multiple times over the course of its life. Fortunately, the relationship to streams and tables here is pretty straightforward:

Discarding mode requires the system to either throw away the previous value for the window when triggering or keep around a copy of the previous value and compute the delta the next time the window triggers.14 (This mode might have better been called Delta mode.)

Accumulating mode requires no additional work; the current value for the window in the table at triggering time is what is emitted. (This mode might have better been called Value mode.)

Accumulating and retracting mode requires keeping around copies of all previously triggered (but not yet retracted) values for the window. This list of previous values can grow quite large in the case of merging windows like sessions, but is vital to cleanly reverting the effects of those previous trigger firings in cases where the new value cannot simply be used to overwrite a previous value. (This mode might have better been called Value and Retractions mode.)

The streams-and-tables visualizations of accumulation modes add little additional insight into their semantics, so we won’t investigate them here.

A Holistic View of Streams and Tables in the Beam Model
Having addressed the four questions, we can now take a holistic view of streams and tables in a Beam Model pipeline. Let’s take our running example (the team scores calculation pipeline) and see what its structure looks like at the streams-and-table level. The full code for the pipeline might look something like Example 6-6 (repeating Example 6-4).

Example 6-6. Our full score-parsing pipeline
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(
                 AfterWatermark()
                   .withEarlyFirings(AlignedDelay(ONE_MINUTE))
                   .withLateFirings(AfterCount(1))))
  .apply(Sum.integersPerKey());
Breaking that apart into stages separated by the intermediate PCollection types (where I’ve used more semantic “type” names like Team and User Score than real types for clarity of what is happening at each stage), you would arrive at something like that depicted in Figure 6-13.


Figure 6-13. Logical phases of a team score summation pipeline, with intermediate PCollection types
When you actually run this pipeline, it first goes through an optimizer, whose job is to convert this logical execution plan into an optimized, physical execution plan. Each execution engine is different, so actual physical execution plans will vary between runners. But a believable strawperson plan might look something like Figure 6-14.


Figure 6-14. Theoretical physical phases of a team score summation pipeline, with intermediate PCollection types
There’s a lot going on here, so let’s walk through all of it. There are three main differences between Figures 6-13 and 6-14 that we’ll be discussing:

Logical versus physical operations
As part of building a physical execution plan, the underlying engine must convert the logical operations provided by the user into a sequence of primitive operations supported by the engine. In some cases, those physical equivalents look essentially the same (e.g., Parse), and in others, they’re very different.

Physical stages and fusion
It’s often inefficient to execute each logical phase as a fully independent physical stage in the pipeline (with attendant serialization, network communication, and deserialization overhead between each). As a result, the optimizer will typically try to fuse as many physical operations as possible into a single physical stage.

Keys, values, windows, and partitioning
To make it more evident what each physical operation is doing, I’ve annotated the intermediate PCollections with the type of key, value, window, and data partitioning in effect at each point.

Let’s now walk through each logical operation in detail and see what it translated to in the physical plan and how they all relate to streams and tables:

ReadFromSource
Other than being fused with the physical operation immediately following it (Parse), not much interesting happens in translation for ReadFromSource. As far as the characteristics of our data at this point, because the read is essentially consuming raw input bytes, we basically have raw strings with no keys, no windows, and no (or random) partitioning. The original data source can be either a table (e.g., a Cassandra table) or a stream (e.g., RabbitMQ) or something a little like both (e.g., Kafka in log compaction mode). But regardless, the end result of reading from the input source is a stream.

Parse
The logical Parse operation also translates in a relatively straightforward manner to the physical version. Parse takes the raw strings and extracts a key (team ID) and value (user score) from them. It’s a nongrouping operation, and thus the stream it consumed remains a stream on the other side.

Window+Trigger
This logical operation is spread out across a number of distinct physical operations. The first is window assignment, in which each element is assigned to a set of windows. That happens immediately in the AssignWindows operation, which is a nongrouping operation that simply annotates each element in the stream with the window(s) it now belongs to, yielding another stream on the other side.

The second is window merging, which we learned earlier in the chapter happens as part of the grouping operation. As such, it gets sunk down into the GroupMergeAndCombine operation later in the pipeline. We discuss that operation when we talk about the logical Sum operation next.

And finally, there’s triggering. Triggering happens after grouping and is the way that we’ll convert the table created by grouping back into a stream. As such, it gets sunk into its own operation, which follows GroupMergeAndCombine.

Sum
Summation is really a composite operation, consisting of a couple pieces: partitioning and aggregation. Partitioning is a nongrouping operation that redirects the elements in the stream in such a way that elements with the same keys end up going to the same physical machine. Another word for partitioning is shuffling, though that term is a bit overloaded because “Shuffle” in the MapReduce sense is often used to mean both partitioning and grouping (and sorting, for that matter). Regardless, partitioning physically alters the stream in way that makes it groupable but doesn’t do anything to actually bring the data to rest. As a result, it’s a nongrouping operation that yields another stream on the other side.

After partitioning comes grouping. Grouping itself is a composite operation. First comes grouping by key (enabled by the previous partition-by-key operation). Next comes window merging and grouping by window, as we described earlier. And finally, because summation is implemented as a CombineFn in Beam (essentially an incremental aggregation operation), there’s combining, where individual elements are summed together as they arrive. The specific details are not terribly important for our purposes here. What is important is the fact that, since this is (obviously) a grouping operation, our chain of streams now comes to rest in a table containing the summed team totals as they evolve over time.

WriteToSink
Lastly, we have the write operation, which takes the stream yielded by triggering (which was sunk below the GroupMergeAndCombine operation, as you might recall) and writes it out to our output data sink. That data itself can be either a table or stream. If it’s a table, WriteToSink will need to perform some sort of grouping operation as part of writing the data into the table. If it’s a stream, no grouping will be necessary (though partitioning might still be desired; for example, when writing into something like Kafka).

The big takeaway here is not so much the precise details of everything that’s going on in the physical plan, but more the overall relationship of the Beam Model to the world of streams and tables. We saw three types of operations: nongrouping (e.g., Parse), grouping (e.g., GroupMergeAndCombine), and ungrouping (e.g., Trigger). The nongrouping operations always consumed streams and produced streams on the other side. The grouping operations always consumed streams and yielded tables. And the ungrouping operations consumed tables and yielded streams. These insights, along with everything else we’ve learned along the way, are enough for us to formulate a more general theory about the relationship of the Beam Model to streams and tables.

A General Theory of Stream and Table Relativity
Having surveyed how stream processing, batch processing, the four what/where/when/how questions, and the Beam Model as a whole relate to stream and table theory, let’s now attempt to articulate a more general definition of stream and table relativity.

A general theory of stream and table relativity:

Data processing pipelines (both batch and streaming) consist of tables, streams, and operations upon those tables and streams.

Tables are data at rest, and act as a container for data to accumulate and be observed over time.

Streams are data in motion, and encode a discretized view of the evolution of a table over time.

Operations act upon a stream or table and yield a new stream or table. They are categorized as follows:

stream → stream: Nongrouping (element-wise) operations

Applying nongrouping operations to a stream alters the data in the stream while leaving them in motion, yielding a new stream with possibly different cardinality.

stream → table: Grouping operations

Grouping data within a stream brings those data to rest, yielding a table that evolves over time.

Windowing incorporates the dimension of event time into such groupings.

Merging windows dynamically combine over time, allowing them to reshape themselves in response to the data observed and dictating that key remain the unit of atomicity/parallelization, with window being a child component of grouping within that key.

table → stream: Ungrouping (triggering) operations

Triggering data within a table ungroups them into motion, yielding a stream that captures a view of the table’s evolution over time.

Watermarks provide a notion of input completeness relative to event time, which is a useful reference point when triggering event-timestamped data, particularly data grouped into event-time windows from unbounded streams.

The accumulation mode for the trigger determines the nature of the stream, dictating whether it contains deltas or values, and whether retractions for previous deltas/values are provided.

table → table: (none)

There are no operations that consume a table and yield a table, because it’s not possible for data to go from rest and back to rest without being put into motion. As a result, all modifications to a table are via conversion to a stream and back again.

What I love about these rules is that they just make sense. They have a very natural and intuitive feeling about them, and as a result they make it so much easier to understand how data flow (or don’t) through a sequence of operations. They codify the fact that data exist in one of two constitutions at any given time (streams or tables), and they provide simple rules for reasoning about the transitions between those states. They demystify windowing by showing how it’s just a slight modification of a thing everyone already innately understands: grouping. They highlight why grouping operations in general are always such a sticking point for streaming (because they bring data in streams to rest as tables) but also make it very clear what sorts of operations are needed to get things unstuck (triggers; i.e., ungrouping operations). And they underscore just how unified batch and stream processing really are, at a conceptual level.

When I set out to write this chapter, I wasn’t entirely sure what I was going to end up with, but the end result was much more satisfying than I’d imagined it might be. In the chapters to come, we use this theory of stream and table relativity again and again to help guide our analyses. And every time, its application will bring clarity and insight that would otherwise have been much harder to gain. Streams and tables are the best.

Summary
In this chapter, we first established the basics of stream and table theory. We first defined streams and tables relatively:

streams → tables
The aggregation of a stream of updates over time yields a table.

tables → streams
The observation of changes to a table over time yields a stream.

We next defined them independently:

Tables are data at rest.

Streams are data in motion.

We then assessed the classic MapReduce model of batch computation from a streams and tables perspective and came to the conclusion that the following four steps describe batch processing from that perspective:

Tables are read in their entirety to become streams.

Streams are processed into new streams until a grouping operation is hit.

Grouping turns the stream into a table.

Steps 1 through 3 repeat until you run out of operations in the pipeline.

From this analysis, we were able to see that streams are just as much a part of batch processing as they are stream processing, and also that the idea of data being a stream is an orthogonal one from whether the data in question are bounded or unbounded.

Next, we spent a good deal of time considering the relationship between streams and tables and the robust, out-of-order stream processing semantics afforded by the Beam Model, ultimately arriving at the general theory of stream and table relativity we enumerated in the previous section. In addition to the basic definitions of streams and tables, the key insight in that theory is that there are four (really, just three) types of operations in a data processing pipeline:

stream → stream
Nongrouping (element-wise) operations

stream → table
Grouping operations

table → stream
Ungrouping (triggering) operations

table → table
(nonexistent)

By classifying operations in this way, it becomes trivial to understand how data flow through (and linger within) a given pipeline over time.

Finally, and perhaps most important of all, we learned this: when you look at things from the streams-and-tables point of view, it becomes abundantly clear how batch and streaming really are just the same thing conceptually. Bounded or unbounded, it doesn’t matter. It’s streams and tables from top to bottom.

</bad-physics-jokes>

1 If you didn’t go to college for computer science and you’ve made it this far in the book, you are likely either 1) my parents, 2) masochistic, or 3) very smart (and for the record, I’m not implying these groups are necessarily mutually exclusive; figure that one out if you can, Mom and Dad! <winky-smiley/>).

2 And note that in some cases, the tables themselves can accept time as a query parameter, allowing you to peer backward in time to snapshots of the table as it existed in the past.

3 Note that no guarantees are made about the keys of two successive records observed by a single mapper, because no key-grouping has occurred yet. The existence of the key here is really just to allow keyed datasets to be consumed in a natural way, and if there are no obvious keys for the input data, they’ll all just share what is effectively a global null key.

4 Calling the inputs to a batch job “static” might be a bit strong. In reality, the dataset being consumed can be constantly changing as it’s processed; that is, if you’re reading directly from an HBase/Bigtable table within a timestamp range in which the data aren’t guaranteed to be immutable. But in most cases, the recommended approach is to ensure that you’re somehow processing a static snapshot of the input data, and any deviation from that assumption is at your own peril.

5 Note that grouping a stream by key is importantly distinct from simply partitioning that stream by key, which ensures that all records with the same key end up being processed by the same machine but doesn’t do anything to put the records to rest. They instead remain in motion and thus continue on as a stream. A grouping operation is more like a partition-by-key followed by a write to the appropriate group for that partition, which is what puts them to rest and turns the stream into a table.

6 One giant difference, from an implementation perspective at least, being that ReduceWrite, knowing that keys have already been grouped together by MapWrite, and further knowing that Reduce is unable to alter keys for the case in which its outputs remain keyed, can simply accumulate the outputs generated by reducing the values for a single key in order to group them together, which is much simpler than the full-blown shuffle implementation required for a MapWrite phase.

7 Another way of looking at it is that there are two types of tables: updateable and appendable; this is the way the Flink folks have framed it for their Table API. But even though that’s a great intuitive way of capturing the observed semantics of the two situations, I think it obscures the underlying nature of what’s actually happening that causes a stream to come to rest as a table; that is, grouping.

8 Though as we can clearly see from this example, it’s not just a streaming thing; you can get the same effect with a batch system if its state tables are world readable.

9 This is particularly painful if a sink for your storage system of choice doesn’t exist yet; building proper sinks that can uphold consistency guarantees is a surprisingly subtle and difficult task.

10 This also means that if you place a value into multiple windows—for example, sliding windows—the value must conceptually be duplicated into multiple, independent records, one per window. Even so, it’s possible in some cases for the underlying system to be smart about how it treats certain types of overlapping windows, thus optimize away the need for actually duplicating the value. Spark, for example, does this for sliding windows.

11 Note that this high-level conceptual view of how things work in batch pipelines belies the complexity of efficiently triggering an entire table of data at once, particularly when that table is sizeable enough to require a plurality of machines to process. The SplittableDoFn API recently added to Beam provides some insight into the mechanics involved.

12 And yes, if you blend batch and streaming together you get Beam, which is where that name came from originally. For reals.

13 This is why you should always use an Oxford comma.

14 Note that in the case of merging windows, in addition to merging the current values for the two windows to yield a merged current value, the previous values for those two windows would need to be merged, as well, to allow for the later calculation of a merged delta come triggering time.