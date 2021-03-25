# Chapter 8. Streaming SQL
Let’s talk SQL. In this chapter, we’re going to start somewhere in the middle with the punchline, jump back in time a bit to establish additional context, and finally jump back to the future to wrap everything up with a nice bow. Imagine Quentin Tarantino held a degree in computer science and was super pumped to tell the world about the finer points of streaming SQL, and so he offered to ghostwrite this chapter with me; it’s sorta like that. Minus the violence.

What Is Streaming SQL?
I would argue that the answer to this question has eluded our industry for decades. In all fairness, the database community has understood maybe 99% of the answer for quite a while now. But I have yet to see a truly cogent and comprehensive definition of streaming SQL that encompasses the full breadth of robust streaming semantics. That’s what we’ll try to come up with here, although it would be hubris to assume we’re 100% of the way there now. Maybe 99.1%? Baby steps.

Regardless, I want to point out up front that most of what we’ll discuss in this chapter is still purely hypothetical as of the time of writing. This chapter and the one that follows (covering streaming joins) both describe an idealistic vision for what streaming SQL could be. Some pieces are already implemented in systems like Apache Calcite, Apache Flink, and Apache Beam. Many others aren’t implemented anywhere. Along the way, I’ll try to call out a few of the things that do exist in concrete form, but given what a moving target that is, your best bet is to simply consult the documentation for your specific system of interest.

On that note, it’s also worth highlighting that the vision for streaming SQL presented here is the result of a collaborative discussion between the Calcite, Flink, and Beam communities. Julian Hyde, the lead developer on Calcite, has long pitched his vision for what streaming SQL might look like. In 2016, members of the Flink community integrated Calcite SQL support into Flink itself, and began adding streaming-specific features such as windowing constructs to the Calcite SQL dialect. Then, in 2017, all three communities began a discussion to try to come to agreement on what language extensions and semantics for robust stream processing in Calcite SQL should look like. This chapter attempts to distill the ideas from that discussion down into a clear and cohesive narrative about integrating streaming concepts into SQL, regardless of whether it’s Calcite or some other dialect.

Relational Algebra
When talking about what streaming means for SQL, it’s important to keep in mind the theoretical foundation of SQL: relational algebra. Relational algebra is simply a mathematical way of describing relationships between data that consist of named, typed tuples. At the heart of relational algebra is the relation itself, which is a set of these tuples. In classic database terms, a relation is something akin to a table, be it a physical database table, the result of a SQL query, a view (materialized or otherwise), and so on; it’s a set of rows containing named and typed columns of data.

One of the more critical aspects of relational algebra is its closure property: applying any operator from the relational algebra to any valid relation1 always yields another relation. In other words, relations are the common currency of relational algebra, and all operators consume them as input and produce them as output.

Historically, many attempts to support streaming in SQL have fallen short of satisfying the closure property. They treat streams separately from classic relations, providing new operators to convert between the two, and restricting the operations that can be applied to one or the other. This significantly raises the bar of adoption for any such streaming SQL system: would-be users must learn the new operators and understand the places where they’re applicable, where they aren’t, and similarly relearn the rules of applicability in this new world for any old operators. What’s worse, most of these systems still fall short of providing the full suite of streaming semantics that we would want, such as support for robust out-of-order processing and strong temporal join support (the latter of which we cover in Chapter 9). As a result, I would argue that it’s basically impossible to name any existing streaming SQL implementation that has achieved truly broad adoption. The additional cognitive overhead and restricted capabilities of such streaming SQL systems have ensured that they remain a niche enterprise.

To change that, to truly bring streaming SQL to the forefront, what we need is a way for streaming to become a first-class citizen within the relational algebra itself, such that the entire standard relational algebra can apply naturally in both streaming and nonstreaming use cases. That isn’t to say that streams and tables should be treated as exactly the same thing; they most definitely are not the same, and recognizing that fact lends clarity to understanding and power to navigating the stream/table relationship, as we’ll see shortly. But the core algebra should apply cleanly and naturally to both worlds, with minimal extensions beyond the standard relational algebra only in the cases where absolutely necessary.

Time-Varying Relations
To cut to the chase, the punchline I referred to at the beginning of the chapter is this: the key to naturally integrating streaming into SQL is to extend relations, the core data objects of relational algebra, to represent a set of data over time rather than a set of data at a specific point in time. More succinctly, instead of point-in-time relations, we need time-varying relations.2

But what are time-varying relations? Let’s first define them in terms of classic relational algebra, after which we’ll also consider their relationship to stream and table theory.

In terms of relational algebra, a time-varying relation is really just the evolution of a classic relation over time. To understand what I mean by that, imagine a raw dataset consisting of user events. Over time, as users generate new events, the dataset continues to grow and evolve. If you observe that set at a specific point in time, that’s a classic relation. But if you observe the holistic evolution of the set over time, that’s a time-varying relation.

Put differently, if classic relations are like two-dimensional tables consisting of named, typed columns in the x-axis and rows of records in the y-axis, time-varying relations are like three-dimensional tables with x- and y-axes as before, but an additional z-axis capturing different versions of the two-dimensional table over time. As the relation changes, new snapshots of the relation are added in the z dimension.

Let’s look at an example. Imagine our raw dataset is users and scores; for example, per-user scores from a mobile game as in most of the other examples throughout the book. And suppose that our example dataset here ultimately ends up looking like this when observed at a specific point in time, in this case 12:07:

12:07> SELECT * FROM UserScores;
-------------------------
| Name  | Score | Time  |
-------------------------
| Julie | 7     | 12:01 |
| Frank | 3     | 12:03 |
| Julie | 1     | 12:03 |
| Julie | 4     | 12:07 |
-------------------------
In other words, it recorded the arrivals of four scores over time: Julie’s score of 7 at 12:01, both Frank’s score of 3 and Julie’s second score of 1 at 12:03, and, finally, Julie’s third score of 4 at 12:07 (note that the Time column here contains processing-time timestamps representing the arrival time of the records within the system; we get into event-time timestamps a little later on). Assuming these were the only data to ever arrive for this relation, it would look like the preceding table any time we observed it after 12:07. But if instead we had observed the relation at 12:01, it would have looked like the following, because only Julie’s first score would have arrived by that point:

12:01> SELECT * FROM UserScores;
-------------------------
| Name  | Score | Time  |
-------------------------
| Julie | 7     | 12:01 |
-------------------------
If we had then observed it again at 12:03, Frank’s score and Julie’s second score would have also arrived, so the relation would have evolved to look like this:

12:03> SELECT * FROM UserScores;
-------------------------
| Name  | Score | Time  |
-------------------------
| Julie | 7     | 12:01 |
| Frank | 3     | 12:03 |
| Julie | 1     | 12:03 |
-------------------------
From this example we can begin to get a sense for what the time-varying relation for this dataset must look like: it would capture the entire evolution of the relation over time. Thus, if we observed the time-varying relation (or TVR) at or after 12:07, it would thus look like the following (note the use of a hypothetical TVR keyword to signal that we want the query to return the full time-varying relation, not the standard point-in-time snapshot of a classic relation):

12:07> SELECT TVR * FROM UserScores;
---------------------------------------------------------
| [-inf, 12:01)             | [12:01, 12:03)            |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           |                           |
|                           |                           |
|                           |                           |
|                           |                           |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07)            | [12:07, now)              |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           | Julie                     |
|                           | Frank                     |
|                           | Julie                     |
|                           |                           |
| ------------------------- | ------------------------- |
---------------------------------------------------------
Because the printed/digital page remains constrained to two dimensions, I’ve taken the liberty of flattening the third dimension into a grid of two-dimensional relations. But you can see how the time-varying relation essentially consists of a sequence of classic relations (ordered left to right, top to bottom), each capturing the full state of the relation for a specific range of time (all of which, by definition, are contiguous).

What’s important about defining time-varying relations this way is that they really are, for all intents and purposes, just a sequence of classic relations that each exist independently within their own disjointed (but adjacent) time ranges, with each range capturing a period of time during which the relation did not change. This is important, because it means that the application of a relational operator to a time-varying relation is equivalent to individually applying that operator to each classic relation in the corresponding sequence. And taken one step further, the result of individually applying a relational operator to a sequence of relations, each associated with a time interval, will always yield a corresponding sequence of relations with the same time intervals. In other words, the result is a corresponding time-varying relation. This definition gives us two very important properties:

The full set of operators from classic relational algebra remain valid when applied to time-varying relations, and furthermore continue to behave exactly as you’d expect.

The closure property of relational algebra remains intact when applied to time-varying relations.

Or more succinctly, all the rules of classic relational algebra continue to hold when applied to time-varying relations. This is huge, because it means that our substitution of time-varying relations for classic relations hasn’t altered the parameters of the game in any way. Everything continues to work the way it did back in classic relational land, just on sequences of classic relations instead of singletons. Going back to our examples, consider two more time-varying relations over our raw dataset, both observed at some time after 12:07. First a simple filtering relation using a WHERE clause:

12:07> SELECT TVR * FROM UserScores WHERE Name = "Julie";
---------------------------------------------------------
| [-inf, 12:01)             | [12:01, 12:03)            |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           |                           |
|                           |                           |
|                           |                           |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07)            | [12:07, now)              |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           | Julie                     |
|                           | Julie                     |
|                           |                           |
| ------------------------- | ------------------------- |
---------------------------------------------------------
As you would expect, this relation looks a lot like the preceding one, but with Frank’s scores filtered out. Even though the time-varying relation captures the added dimension of time necessary to record the evolution of this dataset over time, the query behaves exactly as you would expect, given your understanding of SQL.

For something a little more complex, let’s consider a grouping relation in which we’re summing up all the per-user scores to generate a total overall score for each user:

12:07> SELECT TVR Name, SUM(Score) as Total, MAX(Time) as Time 
       FROM UserScores GROUP BY Name;
---------------------------------------------------------
| [-inf, 12:01)             | [12:01, 12:03)            |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           |                           |
|                           |                           |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07)            | [12:07, now)              |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           | Julie                     |
|                           | Frank                     |
| ------------------------- | ------------------------- |
---------------------------------------------------------
Again, the time-varying version of this query behaves exactly as you would expect, with each classic relation in the sequence simply containing the sum of the scores for each user. And indeed, no matter how complicated a query we might choose, the results are always identical to applying that query independently to the commensurate classic relations composing the input time-varying relation. I cannot stress enough how important this is!

All right, that’s all well and good, but time-varying relations themselves are more of a theoretical construct than a practical, physical manifestation of data; it’s pretty easy to see how they could grow to be quite huge and unwieldy for large datasets that change frequently. To see how they actually tie into real-world stream processing, let’s now explore the relationship between time-varying relations and stream and table theory.

Streams and Tables
For this comparison, let’s consider again our grouped time-varying relation that we looked at earlier:

12:07> SELECT TVR Name, SUM(Score) as Total, MAX(Time) as Time
       FROM UserScores GROUP BY Name;
---------------------------------------------------------
| [-inf, 12:01)             | [12:01, 12:03)            |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           |                           |
|                           |                           |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07)            | [12:07, now)              |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           | Julie                     |
|                           | Frank                     |
| ------------------------- | ------------------------- |
---------------------------------------------------------
We understand that this sequence captures the full history of the relation over time. Given our understanding of tables and streams from Chapter 6, it’s not too difficult to understand how time-varying relations relate to stream and table theory.

Tables are quite straightforward: because a time-varying relation is essentially a sequence of classic relations (each capturing a snapshot of the relation at a specific point in time), and classic relations are analogous to tables, observing a time-varying relation as a table simply yields the point-in-time relation snapshot for the time of observation.

For example, if we were to observe the previous grouped time-varying relation as a table at 12:01, we’d get the following (note the use of another hypothetical keyword, TABLE, to explicitly call out our desire for the query to return a table):

12:01> SELECT TABLE Name, SUM(Score) as Total, MAX(Time) as Time
       FROM UserScores GROUP BY Name;
-------------------------
| Name  | Total | Time  |
-------------------------
| Julie | 7     | 12:01 |
-------------------------
And observing at 12:07 would yield the expected:

12:07> SELECT TABLE Name, SUM(Score) as Total, MAX(Time) as Time
       FROM UserScores GROUP BY Name;
-------------------------
| Name  | Total | Time  |
-------------------------
| Julie | 12    | 12:07 |
| Frank | 3     | 12:03 |
-------------------------
What’s particularly interesting here is that there’s actually support for the idea of time-varying relations within SQL, even as it exists today. The SQL 2011 standard provides “temporal tables,” which store a versioned history of the table over time (in essence, time-varying relations) as well an AS OF SYSTEM TIME construct that allows you to explicitly query and receive a snapshot of the temporal table/time-varying relation at whatever point in time you specified. For example, even if we performed our query at 12:07, we could still see what the relation looked like back at 12:03:

12:07> SELECT TABLE Name, SUM(Score) as Total, MAX(Time) as Time
       FROM UserScores GROUP BY Name AS OF SYSTEM TIME ‘12:03’;
-------------------------
| Name  | Total | Time  |
-------------------------
| Julie | 8     | 12:03 |
| Frank | 3     | 12:03 |
-------------------------
So there’s some amount of precedent for time-varying relations in SQL already. But I digress. The main point here is that tables capture a snapshot of the time-varying relation at a specific point in time. Most real-world table implementations simply track real time as we observe it; others maintain some additional historical information, which in the limit is equivalent to a full-fidelity time-varying relation capturing the entire history of a relation over time.

Streams are slightly different beasts. We learned in Chapter 6 that they too capture the evolution of a table over time. But they do so somewhat differently than the time-varying relations we’ve looked at so far. Instead of holistically capturing snapshots of the entire relation each time it changes, they capture the sequence of changes that result in those snapshots within a time-varying relation. The subtle difference here becomes more evident with an example.

As a refresher, recall again our baseline example TVR query:

12:07> SELECT TVR Name, SUM(Score) as Total, MAX(Time) as Time
       FROM UserScores GROUP BY Name;
---------------------------------------------------------
| [-inf, 12:01)             | [12:01, 12:03)            |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           |                           |
|                           |                           |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07)            | [12:07, now)              |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           | Julie                     |
|                           | Frank                     |
| ------------------------- | ------------------------- |
---------------------------------------------------------
Let’s now observe our time-varying relation as a stream as it exists at a few distinct points in time. At each step of the way, we’ll compare the original table rendering of the TVR at that point in time with the evolution of the stream up to that point. To see what stream renderings of our time-varying relation look like, we’ll need to introduce two new hypothetical keywords:

A STREAM keyword, similar to the TABLE keyword I’ve already introduced, that indicates we want our query to return an event-by-event stream capturing the evolution of the time-varying relation over time. You can think of this as applying a per-record trigger to the relation over time.

A special Sys.Undo3 column that can be referenced from a STREAM query, for the sake of identifying rows that are retractions. More on this in a moment.

Thus, starting out from 12:01, we’d have the following:

                                          12:01> SELECT STREAM Name, 
12:01> SELECT TABLE Name,                          SUM(Score) as Total,
         SUM(Score) as Total,                      MAX(Time) as Time,
         MAX(Time) as Time                         Sys.Undo as Undo
       FROM UserScores GROUP BY Name;            FROM UserScores GROUP BY Name;
-------------------------                 --------------------------------
| Name  | Total | Time  |                 | Name  | Total | Time  | Undo |
-------------------------                 --------------------------------
| Julie | 7     | 12:01 |                 | Julie | 7     | 12:01 |      |
-------------------------                 ........ [12:01, 12:01] ........
The table and stream renderings look almost identical at this point. Mod the Undo column (discussed in more detail in the next example), there’s only one difference: whereas the table version is complete as of 12:01 (signified by the final line of dashes closing off the bottom end of the relation), the stream version remains incomplete, as signified by the final ellipsis-like line of periods marking both the open tail of the relation (where additional data might be forthcoming in the future) as well as the processing-time range of data observed so far. And indeed, if executed on a real implementation, the STREAM query would wait indefinitely for additional data to arrive. Thus, if we waited until 12:03, three new rows would show up for the STREAM query. Compare that to a fresh TABLE rendering of the TVR at 12:03:

                                          12:01> SELECT STREAM Name, 
12:03> SELECT TABLE Name,                          SUM(Score) as Total,
         SUM(Score) as Total,                      MAX(Time) as Time,
         MAX(Time) as Time                         Sys.Undo as Undo
       FROM UserScores GROUP BY Name;            FROM UserScores GROUP BY Name;
-------------------------                 --------------------------------
| Name  | Total | Time  |                 | Name  | Total | Time  | Undo |
-------------------------                 --------------------------------
| Julie | 8     | 12:03 |                 | Julie | 7     | 12:01 |      |
| Frank | 3     | 12:03 |                 | Frank | 3     | 12:03 |      |
-------------------------                 | Julie | 7     | 12:03 | undo |
                                          | Julie | 8     | 12:03 |      |
                                          ........ [12:01, 12:03] ........
Here’s an interesting point worth addressing: why are there three new rows in the stream (Frank’s 3 and Julie’s undo-7 and 8) when our original dataset contained only two rows (Frank’s 3 and Julie’s 1) for that time period? The answer lies in the fact that here we are observing the stream of changes to an aggregation of the original inputs; in particular, for the time period from 12:01 to 12:03, the stream needs to capture two important pieces of information regarding the change in Julie’s aggregate score due to the arrival of the new 1 value:

The previously reported total of 7 was incorrect.

The new total is 8.

That’s what the special Sys.Undo column allows us to do: distinguish between normal rows and rows that are a retraction of a previously reported value.4

A particularly nice feature of STREAM queries is that you can begin to see how all of this relates to the world of classic Online Transaction Processing (OLTP) tables: the STREAM rendering of this query is essentially capturing a sequence of INSERT and DELETE operations that you could use to materialize this relation over time in an OLTP world (and really, when you think about it, OLTP tables themselves are essentially time-varying relations mutated over time via a stream of INSERTs, UPDATEs, and DELETEs).

Now, if we don’t care about the retractions in the stream, it’s also perfectly fine not to ask for them. In that case, our STREAM query would look like this:

12:01> SELECT STREAM Name,
         SUM(Score) as Total,
         MAX(Time) as Time
       FROM UserScores GROUP BY Name;
------------------------- 
| Name  | Total | Time  |
------------------------- 
| Julie | 7     | 12:01 | 
| Frank | 3     | 12:03 |
| Julie | 8     | 12:03 |
.... [12:01, 12:03] .....
But there’s clearly value in understanding what the full stream looks like, so we’ll go back to including the Sys.Undo column for our final example. Speaking of which, if we waited another four minutes until 12:07, we’d be greeted by two additional rows in the STREAM query, whereas the TABLE query would continue to evolve as before:

                                          12:01> SELECT STREAM Name, 
12:07> SELECT TABLE Name,                          SUM(Score) as Total,
         SUM(Score) as Total,                      MAX(Time) as Time,
         MAX(Time) as Time                         Sys.Undo as Undo
       FROM UserScores GROUP BY Name;            FROM UserScores GROUP BY Name;
-------------------------                 --------------------------------
| Name  | Total | Time  |                 | Name  | Total | Time  | Undo |
-------------------------                 --------------------------------
| Julie | 12    | 12:07 |                 | Julie | 7     | 12:01 |      |
| Frank | 3     | 12:03 |                 | Frank | 3     | 12:03 |      |
-------------------------                 | Julie | 7     | 12:03 | undo |
                                          | Julie | 8     | 12:03 |      |
                                          | Julie | 8     | 12:07 | undo |
                                          | Julie | 12    | 12:07 |      |
                                          ........ [12:01, 12:07] ........
And by this time, it’s quite clear that the STREAM version of our time-varying relation is a very different beast from the table version: the table captures a snapshot of the entire relation at a specific point in time, whereas the stream captures a view of the individual changes to the relation over time.5 Interestingly though, that means that the STREAM rendering has more in common with our original, table-based TVR rendering:

12:07> SELECT TVR Name, SUM(Score) as Total, MAX(Time) as Time
       FROM UserScores GROUP BY Name;
---------------------------------------------------------
| [-inf, 12:01)             | [12:01, 12:03)            |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           |                           |
|                           |                           |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07)            | [12:07, now)              |
| ------------------------- | ------------------------- |
|                           | Name                      |
| ------------------------- | ------------------------- |
|                           | Julie                     |
|                           | Frank                     |
| ------------------------- | ------------------------- |
---------------------------------------------------------
Indeed, it’s safe to say that the STREAM query simply provides an alternate rendering of the entire history of data that exists in the corresponding table-based TVR query. The value of the STREAM rendering is its conciseness: it captures only the delta of changes between each of the point-in-time relation snapshots in the TVR. The value of the sequence-of-tables TVR rendering is the clarity it provides: it captures the evolution of the relation over time in a format that highlights its natural relationship to classic relations, and in doing so provides for a simple and clear definition of relational semantics within the context of streaming as well as the additional dimension of time that streaming brings.

Another important aspect of the similarities between the STREAM and table-based TVR renderings is the fact that they are essentially equivalent in the overall data they encode. This gets to the core of the stream/table duality that its proponents have long preached: streams and tables6 are really just two different sides of the same coin. Or to resurrect the bad physics analogy from Chapter 6, streams and tables are to time-varying relations as waves and particles are to light:7 a complete time-varying relation is both a table and a stream at the same time; tables and streams are simply different physical manifestations of the same concept, depending upon the context.

Now, it’s important to keep in mind that this stream/table duality is true only as long as both versions encode the same information; that is, when you have full-fidelity tables or streams. In many cases, however, full fidelity is impractical. As I alluded to earlier, encoding the full history of a time-varying relation, no matter whether it’s in stream or table form, can be rather expensive for a large data source. It’s quite common for stream and table manifestations of a TVR to be lossy in some way. Tables typically encode only the most recent version of a TVR; those that support temporal or versioned access often compress the encoded history to specific point-in-time snapshots, and/or garbage-collect versions that are older than some threshold. Similarly, streams typically encode only a limited duration of the evolution of a TVR, often a relatively recent portion of that history. Persistent streams like Kafka afford the ability to encode the entirety of a TVR, but again this is relatively uncommon, with data older than some threshold typically thrown away via a garbage-collection process.

The main point here is that streams and tables are absolutely duals of one another, each a valid way of encoding a time-varying relation. But in practice, it’s common for the physical stream/table manifestations of a TVR to be lossy in some way. These partial-fidelity streams and tables trade off a decrease in total encoded information for some benefit, usually decreased resource costs. And these types of trade-offs are important because they’re often what allow us to build pipelines that operate over data sources of truly massive scale. But they also complicate matters, and require a deeper understanding to use correctly. We discuss this topic in more detail later on when we get to SQL language extensions. But before we try to reason about SQL extensions, it will be useful to understand a little more concretely the biases present in both the SQL and non-SQL data processing approaches common today.

Looking Backward: Stream and Table Biases
In many ways, the act of adding robust streaming support to SQL is largely an exercise in attempting to merge the where, when, and how semantics of the Beam Model with the what semantics of the classic SQL model. But to do so cleanly, and in a way that remains true to the look and feel of classic SQL, requires an understanding of how the two models relate to each other. Thus, much as we explored the relationship of the Beam Model to stream and table theory in Chapter 6, we’ll now explore the relationship of the Beam Model to the classic SQL model, using stream and table theory as the underlying framework for our comparison. In doing so, we’ll uncover the inherent biases present in each model, which will provide us some insights in how to best marry the two in a clean, natural way.

The Beam Model: A Stream-Biased Approach
Let’s begin with the Beam Model, building upon the discussion in Chapter 6. To begin, I want to discuss the inherent stream bias in the Beam Model as it exists today relative to streams and tables.

If you think back to Figures 6-11 and 6-12, they showed two different views of the same score-summation pipeline that we’ve used as an example throughout the book: in Figure 6-11 a logical, Beam-Model view, and in Figure 6-12 a physical, streams and tables–oriented view. Comparing the two helped highlight the relationship of the Beam Model to streams and tables. But by overlaying one on top of the other, as I’ve done in Figure 8-1, we can see an additional interesting aspect of the relationship: the Beam Model’s inherent stream bias.


Figure 8-1. Stream bias in the Beam Model approach
In this figure, I’ve drawn dashed red lines connecting the transforms in the logical view to their corresponding components in the physical view. The thing that stands out when observed this way is that all of the logical transformations are connected by streams, even the operations that involve grouping (which we know from Chapter 6 results in a table being created somewhere). In Beam parlance, these transformations are PTransforms, and they are always applied to PCollections to yield new PCollections. The important takeaway here is that PCollections in Beam are always streams. As a result, the Beam Model is an inherently stream-biased approach to data processing: streams are the common currency in a Beam pipeline (even batch pipelines), and tables are always treated specially, either abstracted behind sources and sinks at the edges of the pipeline or hidden away beneath a grouping and triggering operation somewhere in the pipeline.

Because Beam operates in terms of streams, anywhere a table is involved (sources, sinks, and any intermediate groupings/ungroupings), some sort of conversion is necessary to keep the underlying table hidden. Those conversions in Beam look something like this:

Sources that consume tables typically hardcode the manner in which those tables are triggered; there is no way for a user to specify custom triggering of the table they want to consume. The source may be written to trigger every new update to the table as a record, it might batch groups of updates together, or it might provide a single, bounded snapshot of the data in the table at some point in time. It really just depends on what’s practical for a given source, and what use case the author of the source is trying to address.

Sinks that write tables typically hardcode the manner in which they group their input streams. Sometimes, this is done in a way that gives the user a certain amount of control; for example, by simply grouping on a user-assigned key. In other cases, the grouping might be implicitly defined; for example, by grouping on a random physical partition number when writing input data with no natural key to a sharded output source. As with sources, it really just depends on what’s practical for the given sink and what use case the author of the sink is trying to address.

For grouping/ungrouping operations, in contrast to sources and sinks, Beam provides users complete flexibility in how they group data into tables and ungroup them back into streams. This is by design. Flexibility in grouping operations is necessary because the way data are grouped is a key ingredient of the algorithms that define a pipeline. And flexibility in ungrouping is important so that the application can shape the generated streams in ways that are appropriate for the use case at hand.8

However, there’s a wrinkle here. Remember from Figure 8-1 that the Beam Model is inherently biased toward streams. As result, although it’s possible to cleanly apply a grouping operation directly to a stream (this is Beam’s GroupByKey operation), the model never provides first-class table objects to which a trigger can be directly applied. As a result, triggers must be applied somewhere else. There are basically two options here:

Predeclaration of triggers
This is where triggers are specified at a point in the pipeline before the table to which they are actually applied. In this case, you’re essentially prespecifying behavior you’d like to see later on in the pipeline after a grouping operation is encountered. When declared this way, triggers are forward-propagating.

Post-declaration of triggers
This is where triggers are specified at a point in the pipeline following the table to which they are applied. In this case, you’re specifying the behavior you’d like to see at the point where the trigger is declared. When declared this way, triggers are backward-propagating.

Because post-declaration of triggers allows you to specify the behavior you want at the actual place you want to observe it, it’s much more intuitive. Unfortunately, Beam as it exists today (2.x and earlier) uses predeclaration of triggers (similar to how windowing is also predeclared).

Even though Beam provides a number of ways to cope with the fact that tables are hidden, we’re still left with the fact that tables must always be triggered before they can be observed, even if the contents of that table are really the final data that you want to consume. This is a shortcoming of the Beam Model as it exists today, one which could be addressed by moving away from a stream-centric model and toward one that treats both streams and tables as first-class entities.

Let’s now look at the Beam Model’s conceptual converse: classic SQL.

The SQL Model: A Table-Biased Approach
In contrast to the Beam Model’s stream-biased approach, SQL has historically taken a table-biased approach: queries are applied to tables, and always result in new tables. This is similar to the batch processing model we looked at in Chapter 6 with MapReduce,9 but it will be useful to consider a concrete example like the one we just looked at for the Beam Model.

Consider the following denormalized SQL table:

UserScores (user, team, score, timestamp)
It contains user scores, each annotated with the IDs of the corresponding user and their corresponding team. There is no primary key, so you can assume that this is an append-only table, with each row being identified implicitly by its unique physical offset. If we want to compute team scores from this table, we could use a query that looks something like this:

    SELECT team, SUM(score) as total
    FROM UserScores
    GROUP BY team;
When executed by a query engine, the optimizer will probably break this query down into roughly three steps:

Scanning the input table (i.e., triggering a snapshot of it)

Projecting the fields in that table down to team and score

Grouping rows by team and summing the scores

If we look at this using a diagram similar to Figure 8-1, it would look like Figure 8-2.

The SCAN operation takes the input table and triggers it into a bounded stream that contains a snapshot of the contents of that table at query execution time. That stream is consumed by the SELECT operation, which projects the four-column input rows down to two-column output rows. Being a nongrouping operation, it yields another stream. Finally, that two-column stream of teams and user scores enters the GROUP BY and is grouped by team into a table, with scores for the same team SUM’d together, yielding our output table of teams and their corresponding team score totals.


Figure 8-2. Table bias in a simple SQL query
This is a relatively simple example that naturally ends in a table, so it really isn’t sufficient to highlight the table-bias in classic SQL. But we can tease out some more evidence by simply splitting the main pieces of this query (projection and grouping) into two separate queries:

    SELECT team, score
    INTO TeamAndScore
    FROM UserScores;
    
    SELECT team, SUM(score) as total
    INTO TeamTotals
    FROM TeamAndScore
    GROUP BY team;
In these queries, we first project the UserScores table down to just the two columns we care about, storing the results in a temporary TeamAndScore table. We then group that table by team, summing up the scores as we do so. After breaking things out into a pipeline of two queries, our diagram looks like that shown in Figure 8-3.


Figure 8-3. Breaking the query into two to reveal more evidence of table bias
If classic SQL exposed streams as first-class objects, you would expect the result from the first query, TeamAndScore, to be a stream because the SELECT operation consumes a stream and produces a stream. But because SQL’s common currency is tables, it must first convert the projected stream into a table. And because the user hasn’t specified any explicit key for grouping, it must simply group keys by their identity (i.e., append semantics, typically implemented by grouping by the physical storage offset for each row).

Because TeamAndScore is now a table, the second query must then prepend an additional SCAN operation to scan the table back into a stream to allow the GROUP BY to then group it back into a table again, this time with rows grouped by team and with their individual scores summed together. Thus, we see the two implicit conversions (from a stream and back again) that are inserted due to the explicit materialization of the intermediate table.

That said, tables in SQL are not always explicit; implicit tables can exist, as well. For example, if we were to add a HAVING clause to the end of the query with the GROUP BY statement, to filter out teams with scores less than a certain threshold, the diagram would change to look something like Figure 8-4.


Figure 8-4. Table bias with a final HAVING clause
With the addition of the HAVING clause, what used to be the user-visible TeamTotals table is now an implicit, intermediate table. To filter the results of the table according to the rules in the HAVING clause, that table must be triggered into a stream that can be filtered and then that stream must be implicitly grouped back into a table to yield the new output table, LargeTeamTotals.

The important takeaway here is the clear table bias in classic SQL. Streams are always implicit, and thus for any materialized stream a conversion from/to a table is required. The rules for such conversions can be categorized roughly as follows:

Input tables (i.e., sources, in Beam Model terms)
These are always implicitly triggered in their entirety at a specific point in time10 (generally query execution time) to yield a bounded stream containing a snapshot of the table at that time. This is identical to what you get with classic batch processing, as well; for example, the MapReduce case we looked at in Chapter 6.

Output tables (i.e., sinks, in Beam Model terms)
These tables are either direct manifestations of a table created by a final grouping operation in the query, or are the result of an implicit grouping (by some unique identifier for the row) applied to a query’s terminal stream, for queries that do not end in a grouping operation (e.g., the projection query in the previous examples, or a GROUP BY followed by a HAVING clause). As with inputs, this matches the behavior seen in classic batch processing.

Grouping/ungrouping operations
Unlike Beam, these operations provide complete flexibility in one dimension only: grouping. Whereas classic SQL queries provide a full suite of grouping operations (GROUP BY, JOIN, CUBE, etc.), they provide only a single type of implicit ungrouping operation: trigger an intermediate table in its entirety after all of the upstream data contributing to it have been incorporated (again, the exact same implicit trigger provided in MapReduce as part of the shuffle operation). As a result, SQL offers great flexibility in shaping algorithms via grouping but essentially zero flexibility in shaping the implicit streams that exist under the covers during query execution.

Materialized views
Given how analogous classic SQL queries are to classic batch processing, it might be tempting to write off SQL’s inherent table bias as nothing more than an artifact of SQL not supporting stream processing in any way. But to do so would be to ignore the fact that databases have supported a specific type of stream processing for quite some time: materialized views. A materialized view is a view that is physically materialized as a table and kept up to date over time by the database as the source table(s) evolve. Note how this sounds remarkably similar to our definition of a time-varying relation. What’s fascinating about materialized views is that they add a very useful form of stream processing to SQL without significantly altering the way it operates, including its inherent table bias.

For example, let’s consider the queries we looked at in Figure 8-4. We can alter those queries to instead be CREATE MATERIALIZED VIEW11 statements:

    CREATE MATERIALIZED VIEW TeamAndScoreView AS
    SELECT team, score
    FROM UserScores;
    
    CREATE MATERIALIZED VIEW LargeTeamTotalsView AS
    SELECT team, SUM(score) as total
    FROM TeamAndScoreView
    GROUP BY team
    HAVING SUM(score) > 100;
In doing so, we transform them into continuous, standing queries that process the updates to the UserScores table continuously, in a streaming manner. Even so, the resulting physical execution diagram for the views looks almost exactly the same as it did for the one-off queries; nowhere are streams made into explicit first-class objects in order to support this idea of streaming materialized views. The only noteworthy change in the physical execution plan is the substitution of a different trigger: SCAN-AND-STREAM instead of SCAN, as illustrated in Figure 8-5.


Figure 8-5. Table bias in materialized views
What is this SCAN-AND-STREAM trigger? SCAN-AND-STREAM starts out like a SCAN trigger, emitting the full contents of the table at a point in time into a stream. But instead of stopping there and declaring the stream to be done (i.e., bounded), it continues to also trigger all subsequent modifications to the input table, yielding an unbounded stream that captures the evolution of the table over time. In the general case, these modifications include not only INSERTs of new values, but also DELETEs of previous values and UPDATEs to existing values (which, practically speaking, are treated as a simultaneous DELETE/INSERT pair, or undo/redo values as they are called in Flink).

Furthermore, if we consider the table/stream conversion rules for materialized views, the only real difference is the trigger used:

Input tables are implicitly triggered via a SCAN-AND-STREAM trigger instead of a SCAN trigger. Everything else is the same as classic batch queries.

Output tables are treated the same as classic batch queries.

Grouping/ungrouping operations function the same as classic batch queries, with the only difference being the use of a SCAN-AND-STREAM trigger instead of a SNAPSHOT trigger for implicit ungrouping operations.

Given this example, it’s clear to see that SQL’s inherent table bias is not just an artifact of SQL being limited to batch processing:12 materialized views lend SQL the ability to perform a specific type of stream processing without any significant changes in approach, including the inherent bias toward tables. Classic SQL is just a table-biased model, regardless of whether you’re using it for batch or stream processing.

Looking Forward: Toward Robust Streaming SQL
We’ve now looked at time-varying relations, the ways in which tables and streams provide different renderings of a time-varying relation, and what the inherent biases of the Beam and SQL models are with respect to stream and table theory. So where does all of this leave us? And perhaps more to the point, what do we need to change or add within SQL to support robust stream processing? The surprising answer is: not much if we have good defaults.

We know that the key conceptual change is to replace classic, point-in-time relations with time-varying relations. We saw earlier that this is a very seamless substitution, one which applies across the full breadth of relational operators already in existence, thanks to maintaining the critical closure property of relational algebra. But we also saw that dealing in time-varying relations directly is often impractical; we need the ability to operate in terms of our two more-common physical manifestations: tables and streams. This is where some simple extensions with good defaults come in.

We also need some tools for robustly reasoning about time, specifically event time. This is where things like timestamps, windowing, and triggering come into play. But again, judicious choice of defaults will be important to minimize how often these extensions are necessary in practice.

What’s great is that we don’t really need anything more than that. So let’s now finally spend some time looking in detail at these two categories of extensions: stream/table selection and temporal operators.

Stream and Table Selection
As we worked through time-varying relation examples, we already encountered the two key extensions related to stream and table selection. They were those TABLE and STREAM keywords we placed after the SELECT keyword to dictate our desired physical view of a given time-varying relation:

12:07> SELECT TABLE Name,                 12:01> SELECT STREAM Name
         SUM(Score) as Total,                      SUM(Score) as Total,                      
         MAX(Time)                                 MAX(Time) 
       FROM UserScores                           FROM UserScores
       GROUP BY Name;                            GROUP BY Name;
-------------------------                 -------------------------
| Name  | Total | Time  |                 | Name  | Total | Time  |
-------------------------                 -------------------------
| Julie | 12    | 12:07 |                 | Julie | 7     | 12:01 |
| Frank | 3     | 12:03 |                 | Frank | 3     | 12:03 |
-------------------------                 | Julie | 8     | 12:03 |
                                          | Julie | 12    | 12:07 |
                                          ..... [12:01, 12:07] ....
These extensions are relatively straightforward and easy to use when necessary. But the really important thing regarding stream and table selection is the choice of good defaults for times when they aren’t explicitly provided. Such defaults should honor the classic, table-biased behavior of SQL that everyone is accustomed to, while also operating intuitively in a world that includes streams. They should also be easy to remember. The goal here is to help maintain a natural feel to the system, while also greatly decreasing the frequency with which we must use explicit extensions. A good choice of defaults that satisfies all of these requirements is:

If all of the inputs are tables, the output is a TABLE.

If any of the inputs are streams, the output is a STREAM.

What’s additionally important to call out here is that these physical renderings of a time-varying relation are really only necessary when you want to materialize the TVR in some way, either to view it directly or write it to some output table or stream. Given a SQL system that operates under the covers in terms of full-fidelity time-varying relations, intermediate results (e.g., WITH AS or SELECT INTO statements) can remain as full-fidelity TVRs in whatever format the system naturally deals in, with no need to render them into some other, more limited concrete manifestation.

And that’s really it for stream and table selection. Beyond the ability to deal in streams and tables directly, we also need some better tools for reasoning about time if we want to support robust, out-of-order stream processing within SQL. Let’s now look in more detail about what those entail.

Temporal Operators
The foundation of robust, out-of-order processing is the event-time timestamp: that small piece of metadata that captures the time at which an event occurred rather than the time at which it is observed. In a SQL world, event time is typically just another column of data for a given TVR, one which is natively present in the source data themselves.13 In that sense, this idea of materializing a record’s event time within the record itself is something SQL already handles naturally by putting a timestamp in a regular column.

Before we go any further, let’s look at an example. To help tie all of this SQL stuff together with the concepts we’ve explored previously in the book, we resurrect our running example of summing up nine scores from various members of a team to arrive at that team’s total score. If you recall, those scores look like Figure 8-6 when plotted on X = event-time/Y = processing-time axes.


Figure 8-6. Data points in our running example
If we were to imagine these data as a classic SQL table, they might look something like this, ordered by event time (left-to-right in Figure 8-6):

12:10> SELECT TABLE *, Sys.MTime as ProcTime
       FROM UserScores ORDER BY EventTime;
------------------------------------------------
| Name  | Team  | Score | EventTime | ProcTime |
------------------------------------------------
| Julie | TeamX |     5 |  12:00:26 | 12:05:19 |
| Frank | TeamX |     9 |  12:01:26 | 12:08:19 |
| Ed    | TeamX |     7 |  12:02:26 | 12:05:39 |
| Julie | TeamX |     8 |  12:03:06 | 12:07:06 |
| Amy   | TeamX |     3 |  12:03:39 | 12:06:13 |
| Fred  | TeamX |     4 |  12:04:19 | 12:06:39 |
| Naomi | TeamX |     3 |  12:06:39 | 12:07:19 |
| Becky | TeamX |     8 |  12:07:26 | 12:08:39 |
| Naomi | TeamX |     1 |  12:07:46 | 12:09:00 |
------------------------------------------------
If you recall, we saw this table way back in Chapter 2 when I first introduced this dataset. This rendering provides a little more detail on the data than we’ve typically shown, explicitly highlighting the fact that the nine scores themselves belong to seven different users, each a member of the same team. SQL provides a nice, concise way to see the data laid out fully before we begin diving into examples.

Another nice thing about this view of the data is that it fully captures the event time and processing time for each record. You can imagine the event-time column as being just another piece of the original data, and the processing-time column as being something supplied by the system (in this case, using a hypothetical Sys.MTime column that records the processing-time modification timestamp of a given row; that is, the time at which that row arrived in the source table), capturing the ingress time of the records themselves into the system.

The fun thing about SQL is how easy it is to view your data in different ways. For example, if we instead want to see the data in processing-time order (bottom-to-top in Figure 8-6), we could simply update the ORDER BY clause:

12:10> SELECT TABLE *, Sys.MTime as ProcTime
       FROM UserScores ORDER BY ProcTime;
-----------------------------------------------
| Name  | Team  | Score | EventTime | ProcTime |
-----------------------------------------------
| Julie | TeamX |     5 |  12:00:26 | 12:05:19 |
| Ed    | TeamX |     7 |  12:02:26 | 12:05:39 |
| Amy   | TeamX |     3 |  12:03:39 | 12:06:13 |
| Fred  | TeamX |     4 |  12:04:19 | 12:06:39 |
| Julie | TeamX |     8 |  12:03:06 | 12:07:06 |
| Naomi | TeamX |     3 |  12:06:39 | 12:07:19 |
| Frank | TeamX |     9 |  12:01:26 | 12:08:19 |
| Becky | TeamX |     8 |  12:07:26 | 12:08:39 |
| Naomi | TeamX |     1 |  12:07:46 | 12:09:00 |
------------------------------------------------
As we learned earlier, these table renderings of the data are really a partial-fidelity view of the complete underlying TVR. If we were to instead query the full table-oriented TVR (but only for the three most important columns, for the sake of brevity), it would expand to something like this:

12:10> SELECT TVR Score, EventTime, Sys.MTime as ProcTime
       FROM UserScores ORDER BY ProcTime;
-----------------------------------------------------------------------
| [-inf, 12:05:19)                 | [12:05:19, 12:05:39)             |
| -------------------------------- | -------------------------------- |
|                                  | Score                            |
| -------------------------------- | -------------------------------- |
| -------------------------------- |                                  |
|                                  | -------------------------------- |
|                                  |                                  |
-----------------------------------------------------------------------
| [12:05:39, 12:06:13)             | [12:06:13, 12:06:39)             |
| -------------------------------- | -------------------------------- |
|                                  | Score                            |
| -------------------------------- | -------------------------------- |
|                                  | 5                                |
|                                  | 7                                |
| -------------------------------- |                                  |
|                                  | -------------------------------- |
-----------------------------------------------------------------------
| [12:06:39, 12:07:06)             | [12:07:06, 12:07:19)             |
| -------------------------------- | -------------------------------- |
|                                  | Score                            |
| -------------------------------- | -------------------------------- |
|                                  | 5                                |
|                                  | 7                                |
|                                  | 3                                |
|                                  | 4                                |
| -------------------------------- |                                  |
|                                  | -------------------------------- |
-----------------------------------------------------------------------
| [12:07:19, 12:08:19)             | [12:08:19, 12:08:39)             |
| -------------------------------- | -------------------------------- |
|                                  | Score                            |
| -------------------------------- | -------------------------------- |
|                                  | 5                                |
|                                  | 7                                |
|                                  | 3                                |
|                                  | 4                                |
|                                  | 8                                |
|                                  | 3                                |
| -------------------------------- |                                  |
|                                  | -------------------------------- |
|                                  |                                  |
-----------------------------------------------------------------------
| [12:08:39, 12:09:00)             | [12:09:00, now)                  |
| -------------------------------- | -------------------------------- |
|                                  | Score                            |
| -------------------------------- | -------------------------------- |
|                                  | 5                                |
|                                  | 7                                |
|                                  | 3                                |
|                                  | 4                                |
|                                  | 8                                |
|                                  | 3                                |
|                                  | 9                                |
|                                  | 8                                |
| -------------------------------- |                                  |
|                                  | -------------------------------- |
-----------------------------------------------------------------------
That’s a lot of data. Alternatively, the STREAM version would render much more compactly in this instance; thanks to there being no explicit grouping in the relation, it looks essentially identical to the point-in-time TABLE rendering earlier, with the addition of the trailing footer describing the range of processing time captured in the stream so far, plus the note that the system is still waiting for more data in the stream (assuming we’re treating the stream as unbounded; we’ll see a bounded version of the stream shortly):

12:00> SELECT STREAM Score, EventTime, Sys.MTime as ProcTime FROM UserScores;
--------------------------------
| Score | EventTime | ProcTime |
--------------------------------
|     5 |  12:00:26 | 12:05:19 |
|     7 |  12:02:26 | 12:05:39 |
|     3 |  12:03:39 | 12:06:13 |
|     4 |  12:04:19 | 12:06:39 |
|     8 |  12:03:06 | 12:07:06 |
|     3 |  12:06:39 | 12:07:19 |
|     9 |  12:01:26 | 12:08:19 |
|     8 |  12:07:26 | 12:08:39 |
|     1 |  12:07:46 | 12:09:00 |
........ [12:00, 12:10] ........
But this is all just looking at the raw input records without any sort of transformations. Much more interesting is when we start altering the relations. When we’ve explored this example in the past, we’ve always started with classic batch processing to sum up the scores over the entire dataset, so let’s do the same here. The first example pipeline (previously provided as Example 6-1) looked like Example 8-1 in Beam.

Example 8-1. Summation pipeline
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals =
  input.apply(Sum.integersPerKey());
And rendered in the streams and tables view of the world, that pipeline’s execution looked like Figure 8-7.

Figure 8-7. Streams and tables view of classic batch processing
Given that we already have our data placed into an appropriate schema, we won’t be doing any parsing in SQL; instead, we focus on everything in the pipeline after the parse transformation. And because we’re going with the classic batch model of retrieving a single answer only after all of the input data have been processed, the TABLE and STREAM views of the summation relation would look essentially identical (recall that we’re dealing with bounded versions of our dataset for these initial, batch-style examples; as a result, this STREAM query actually terminates with a line of dashes and an END-OF-STREAM marker):

12:10> SELECT TABLE SUM(Score) as Total, MAX(EventTime),
       MAX(Sys.MTime) as "MAX(ProcTime)" FROM UserScores GROUP BY Team;
------------------------------------------
| Total | MAX(EventTime) | MAX(ProcTime) |
------------------------------------------
|    48 |       12:07:46 |      12:09:00 |
------------------------------------------
12:00> SELECT STREAM SUM(Score) as Total, MAX(EventTime),
       MAX(Sys.MTime) as "MAX(ProcTime)" FROM UserScores GROUP BY Team;
------------------------------------------
| Total | MAX(EventTime) | MAX(ProcTime) |
------------------------------------------
|    48 |       12:07:46 |      12:09:00 |
------ [12:00, 12:10] END-OF-STREAM ------
More interesting is when we start adding windowing into the mix. That will give us a chance to begin looking more closely at the temporal operations that need to be added to SQL to support robust stream processing.

Where: windowing
As we learned in Chapter 6, windowing is a modification of grouping by key, in which the window becomes a secondary part of a hierarchical key. As with classic programmatic batch processing, you can window data into more simplistic windows quite easily within SQL as it exists now by simply including time as part of the GROUP BY parameter. Or, if the system in question provides it, you can use a built-in windowing operation. We look at SQL examples of both in a moment, but first, let’s revisit the programmatic version from Chapter 3. Thinking back to Example 6-2, the windowed Beam pipeline looked like that shown in Example 8-2.

Example 8-2. Summation pipeline
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES)))
  .apply(Sum.integersPerKey());
And the execution of that pipeline (in streams and tables rendering from Figure 6-5), looked like the diagrams presented in Figure 8-8.

Figure 8-8. Streams and tables view of windowed summation on a batch engine
As we saw before, the only material change from Figure 8-7 to 8-8 is that the table created by the SUM operation is now partitioned into fixed, two-minute windows of time, yielding four windowed answers at the end rather than the single global sum that we had previously.

To do the same thing in SQL, we have two options: implicitly window by including some unique feature of the window (e.g., the end timestamp) in the GROUP BY statement, or use a built-in windowing operation. Let’s look at both.

First, ad hoc windowing. In this case, we perform the math of calculating windows ourselves in our SQL statement:

12:10> SELECT TABLE SUM(Score) as Total, 
         "[" || EventTime / INTERVAL '2' MINUTES || ", " || 
           (EventTime / INTERVAL '2' MINUTES) + INTERVAL '2' MINUTES ||
           ")" as Window, 
         MAX(Sys.MTime) as "MAX(ProcTime)"
       FROM UserScores
       GROUP BY Team, EventTime / INTERVAL '2' MINUTES;
------------------------------------------------
| Total | Window               | MAX(ProcTime) |
------------------------------------------------
| 14    | [12:00:00, 12:02:00) | 12:08:19      |
| 18    | [12:02:00, 12:04:00) | 12:07:06      |
| 4     | [12:04:00, 12:06:00) | 12:06:39      |
| 12    | [12:06:00, 12:08:00) | 12:09:00      |
------------------------------------------------
We can also achieve the same result using an explicit windowing statement such as those supported by Apache Calcite:

12:10> SELECT TABLE SUM(Score) as Total,
         TUMBLE(EventTime, INTERVAL '2' MINUTES) as Window,
         MAX(Sys.MTime) as 'MAX(ProcTime)' 
       FROM UserScores
       GROUP BY Team, TUMBLE(EventTime, INTERVAL '2' MINUTES);
------------------------------------------------
| Total | Window               | MAX(ProcTime) |
------------------------------------------------
| 14    | [12:00:00, 12:02:00) | 12:08:19      |
| 18    | [12:02:00, 12:04:00) | 12:07:06      |
| 4     | [12:04:00, 12:06:00) | 12:06:39      |
| 12    | [12:06:00, 12:08:00) | 12:09:00      |
------------------------------------------------
This then begs the question: if we can implicitly window using existing SQL constructs, why even bother supporting explicit windowing constructs? There are two reasons, only the first of which is apparent in this example (we’ll see the other one in action later on in the chapter):

Windowing takes care of the window-computation math for you. It’s a lot easier to consistently get things right when you specify basic parameters like width and slide directly rather than computing the window math yourself.14

Windowing allows the concise expression of more complex, dynamic groupings such as sessions. Even though SQL is technically able to express the every-element-within-some-temporal-gap-of-another-element relationship that defines session windows, the corresponding incantation is a tangled mess of analytic functions, self joins, and array unnesting that no mere mortal could be reasonably expected to conjure on their own.

Both are compelling arguments for providing first-class windowing constructs in SQL, in addition to the ad hoc windowing capabilities that already exist.

At this point, we’ve seen what windowing looks like from a classic batch/classic relational perspective when consuming the data as a table. But if we want to consume the data as a stream, we get back to that third question from the Beam Model: when in processing time do we materialize outputs?

When: triggers
The answer to that question, as before, is triggers and watermarks. However, in the context of SQL, there’s a strong argument to be made for having a different set of defaults than those we introduced with the Beam Model in Chapter 3: rather than defaulting to using a single watermark trigger, a more SQL-ish default would be to take a cue from materialized views and trigger on every element. In other words, any time a new input arrives, we produce a corresponding new output.

A SQL-ish default: per-record triggers
There are two compelling benefits to using trigger-every-record as the default:

Simplicity
The semantics of per-record updates are easy to understand; materialized views have operated this way for years.

Fidelity
As in change data capture systems, per-record triggering yields a full-fidelity stream rendering of a given time-varying relation; no information is lost as part of the conversion.

The downside is primarily cost: triggers are always applied after a grouping operation, and the nature of grouping often presents an opportunity to reduce the cardinality of data flowing through the system, thus commensurately reducing the cost of further processing those aggregate results downstream. Even so, the benefits in clarity and simplicity for use cases where cost is not prohibitive arguably outweigh the cognitive complexity of defaulting to a non-full-fidelity trigger up front.

Thus, for our first take at consuming aggregate team scores as a stream, let’s see what things would look like using a per-record trigger. Beam itself doesn’t have a precise per-record trigger, so, as demonstrated in Example 8-3, we instead use a repeated AfterCount(1) trigger, which will fire immediately any time a new record arrives.

Example 8-3. Per-record trigger
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(Repeatedly(AfterCount(1)))
  .apply(Sum.integersPerKey());
A streams and tables rendering of this pipeline would then look something like that depicted in Figure 8-9.

Figure 8-9. Streams and tables view of windowed summation on a streaming engine with per-record triggering
An interesting side effect of using per-record triggers is how it somewhat masks the effect of data being brought to rest because they are then immediately put back into motion again by the trigger. Even so, the aggregate artifact from the grouping remains at rest in the table, as the ungrouped stream of values flows away from it.

Moving back to SQL, we can see now what the effect of rendering the corresponding time-value relation as a stream would be. It (unsurprisingly) looks a lot like the stream of values in the animation in Figure 8-9:

12:00> SELECT STREAM SUM(Score) as Total, 
         TUMBLE(EventTime, INTERVAL '2' MINUTES) as Window,
         MAX(Sys.MTime) as 'MAX(ProcTime)'' 
       FROM UserScores
       GROUP BY Team, TUMBLE(EventTime, INTERVAL '2' MINUTES);
------------------------------------------------
| Total | Window               | MAX(ProcTime) |
------------------------------------------------
| 5     | [12:00:00, 12:02:00) | 12:05:19      |
| 7     | [12:02:00, 12:04:00) | 12:05:39      |
| 10    | [12:02:00, 12:04:00) | 12:06:13      |
| 4     | [12:04:00, 12:06:00) | 12:06:39      |
| 18    | [12:02:00, 12:04:00) | 12:07:06      |
| 3     | [12:06:00, 12:08:00) | 12:07:19      |
| 14    | [12:00:00, 12:02:00) | 12:08:19      |
| 11    | [12:06:00, 12:08:00) | 12:08:39      |
| 12    | [12:06:00, 12:08:00) | 12:09:00      |
................ [12:00, 12:10] ................
But even for this simple use case, it’s pretty chatty. If we’re building a pipeline to process data for a large-scale mobile application, we might not want to pay the cost of processing downstream updates for each and every upstream user score. This is where custom triggers come in.

Watermark triggers
If we were to switch the Beam pipeline to use a watermark trigger, for example, we could get exactly one output per window in the stream version of the TVR, as demonstrated in Example 8-4 and shown in Figure 8-10.

Example 8-4. Watermark trigger
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(AfterWatermark())
  .apply(Sum.integersPerKey());
Figure 8-10. Windowed summation with watermark triggering
To get the same effect in SQL, we’d need language support for specifying a custom trigger. Something like an EMIT <when> statement, such as EMIT WHEN WATERMARK PAST <column>. This would signal to the system that the table created by the aggregation should be triggered into a stream exactly once per row, when the input watermark for the table exceeds the timestamp value in the specified column (which in this case happens to be the end of the window).

Let’s look at this relation rendered as a stream. From the perspective of understanding when trigger firings occur, it’s also handy to stop relying on the MTime values from the original inputs and instead capture the current timestamp at which rows in the stream are emitted:

12:00> SELECT STREAM SUM(Score) as Total,
         TUMBLE(EventTime, INTERVAL '2' MINUTES) as Window,
         CURRENT_TIMESTAMP as EmitTime
       FROM UserScores
       GROUP BY Team, TUMBLE(EventTime, INTERVAL '2' MINUTES)
       EMIT WHEN WATERMARK PAST WINDOW_END(Window);
-------------------------------------------
| Total | Window               | EmitTime |
-------------------------------------------
| 5     | [12:00:00, 12:02:00) | 12:06:00 |
| 18    | [12:02:00, 12:04:00) | 12:07:30 |
| 4     | [12:04:00, 12:06:00) | 12:07:41 |
| 12    | [12:06:00, 12:08:00) | 12:09:22 |
............. [12:00, 12:10] ..............
The main downside here is the late data problem due to the use of a heuristic watermark, as we encountered in previous chapters. In light of late data, a nicer option might be to also immediately output an update any time a late record shows up, using a variation on the watermark trigger that supported repeated late firings, as shown in Example 8-5 and Figure 8-11.

Example 8-5. Watermark trigger with late firings
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(AfterWatermark()
                   .withLateFirings(AfterCount(1))))
  .apply(Sum.integersPerKey());
Figure 8-11. Windowed summation with on-time/late triggering
We can do the same thing in SQL by allowing the specification of two triggers:

A watermark trigger to give us an initial value: WHEN WATERMARK PAST <column>, with the end of the window used as the timestamp <column>.

A repeated delay trigger for late data: AND THEN AFTER <duration>, with a <duration> of 0 to give us per-record semantics.

Now that we’re getting multiple rows per window, it can also be useful to have another two system columns available: the timing of each row/pane for a given window relative to the watermark (Sys.EmitTiming), and the index of the pane/row for a given window (Sys.EmitIndex, to identify the sequence of revisions for a given row/window):

12:00> SELECT STREAM SUM(Score) as Total,
         TUMBLE(EventTime, INTERVAL '2' MINUTES) as Window,
         CURRENT_TIMESTAMP as EmitTime,
         Sys.EmitTiming, Sys.EmitIndex 
       FROM UserScores
       GROUP BY Team, TUMBLE(EventTime, INTERVAL '2' MINUTES)
       EMIT WHEN WATERMARK PAST WINDOW_END(Window)
         AND THEN AFTER 0 SECONDS;
----------------------------------------------------------------------------
| Total | Window               | EmitTime | Sys.EmitTiming | Sys.EmitIndex |
----------------------------------------------------------------------------
| 5     | [12:00:00, 12:02:00) | 12:06:00 | on-time        | 0             |
| 18    | [12:02:00, 12:04:00) | 12:07:30 | on-time        | 0             |
| 4     | [12:04:00, 12:06:00) | 12:07:41 | on-time        | 0             |
| 14    | [12:00:00, 12:02:00) | 12:08:19 | late           | 1             |
| 12    | [12:06:00, 12:08:00) | 12:09:22 | on-time        | 0             |
.............................. [12:00, 12:10] ..............................
For each pane, using this trigger, we’re able to get a single on-time answer that is likely to be correct, thanks to our heuristic watermark. And for any data that arrives late, we can get an updated version of the row amending our previous results.

Repeated delay triggers
The other main temporal trigger use case you might want is repeated delayed updates; that is, trigger a window one minute (in processing time) after any new data for it arrive. Note that this is different than triggering on aligned boundaries, as you would get with a microbatch system. As Example 8-6 shows, triggering via a delay relative to the most recent new record arriving for the window/row helps spread triggering load out more evenly than a bursty, aligned trigger would. It also does not require any sort of watermark support. Figure 8-12 presents the results.

Example 8-6. Repeated triggering with one-minute delays
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
               .triggering(Repeatedly(UnalignedDelay(ONE_MINUTE)))
  .apply(Sum.integersPerKey());
Figure 8-12. Windowed summation with repeated one-minute-delay triggering
The effect of using such a trigger is very similar to the per-record triggering we started out with but slightly less chatty thanks to the additional delay introduced in triggering, which allows the system to elide some number of the rows being produced. Tweaking the delay allows us to tune the volume of data generated, and thus balance the tensions of cost and timeliness as appropriate for the use case.

Rendered as a SQL stream, it would look something like this:

12:00> SELECT STREAM SUM(Score) as Total,
         TUMBLE(EventTime, INTERVAL '2' MINUTES) as Window,
         CURRENT_TIMESTAMP as EmitTime,
         Sys.EmitTiming, SysEmitIndex
       FROM UserScores
       GROUP BY Team, TUMBLE(EventTime, INTERVAL '2' MINUTES)
       EMIT AFTER 1 MINUTE;
----------------------------------------------------------------------------
| Total | Window               | EmitTime | Sys.EmitTiming | Sys.EmitIndex |
----------------------------------------------------------------------------
| 5     | [12:00:00, 12:02:00) | 12:06:19 | n/a            | 0             |
| 10    | [12:02:00, 12:04:00) | 12:06:39 | n/a            | 0             |
| 4     | [12:04:00, 12:06:00) | 12:07:39 | n/a            | 0             |
| 18    | [12:02:00, 12:04:00) | 12:08:06 | n/a            | 1             |
| 3     | [12:06:00, 12:08:00) | 12:08:19 | n/a            | 0             |
| 14    | [12:00:00, 12:02:00) | 12:09:19 | n/a            | 1             |
| 12    | [12:06:00, 12:08:00) | 12:09:22 | n/a            | 1             |
.............................. [12:00, 12:10] ..............................
Data-driven triggers
Before moving on to the final question in the Beam Model, it’s worth briefly discussing the idea of data-driven triggers. Because of the dynamic way types are handled in SQL, it might seem like data-driven triggers would be a very natural addition to the proposed EMIT <when> clause. For example, what if we want to trigger our summation any time the total score exceeds 10? Wouldn’t something like EMIT WHEN Score > 10 work very naturally?

Well, yes and no. Yes, such a construct would fit very naturally. But when you think about what would actually be happening with such a construct, you essentially would be triggering on every record, and then executing the Score > 10 predicate to decide whether the triggered row should be propagated downstream. As you might recall, this sounds a lot like what happens with a HAVING clause. And, indeed, you can get the exact same effect by simply prepending HAVING Score > 10 to the end of the query. At which point, it begs the question: is it worth adding explicit data-driven triggers? Probably not. Even so, it’s still encouraging to see just how easy it is to get the desired effect of data-driven triggers using standard SQL and well-chosen defaults.

How: accumulation
So far in this section, we’ve been ignoring the Sys.Undo column that I introduced toward the beginning of this chapter. As a result, we’ve defaulted to using accumulating mode to answer the question of how refinements for a window/row relate to one another. In other words, any time we observed multiple revisions of an aggregate row, the later revisions built upon the previous revisions, accumulating new inputs together with old ones. I opted for this approach because it matches the approach used in an earlier chapter, and it’s a relatively straightforward translation from how things work in a table world.

That said, accumulating mode has some major drawbacks. In fact, as we discussed in Chapter 2, it’s plain broken for any query/pipeline with a sequence of two or more grouping operations due to over counting. The only sane way to allow for the consumption of multiple revisions of a row within a system that allows for queries containing more than one serial grouping operation is if it operates by default in accumulating and retracting mode. Otherwise, you run into issues where a given input record is included multiple times in a single aggregation due to the blind incorporation of multiple revisions for a single row.

So, when we come to the question of incorporating accumulation mode semantics into a SQL world, the option that fits best with our goal of providing an intuitive and natural experience is if the system uses retractions by default under the covers.15 As noted when I introduced the Sys.Undo column earlier, if you don’t care about the retractions (as in the examples in this section up until now), you don’t need to ask for them. But if you do ask for them, they should be there.

Retractions in a SQL world
To see what I mean, let’s look at another example. To motivate the problem appropriately, let’s look at a use case that’s relatively impractical without retractions: building session windows and writing them incrementally to a key/value store like HBase. In this case, we’ll be producing incremental sessions from our aggregation as they are built up. But in many cases, a given session will simply be an evolution of one or more previous sessions. In that case, you’d really like to delete the previous session(s) and replace it/them with the new one. But how do you do that? The only way to tell whether a given session replaces another one is to compare them to see whether the new one overlaps the old one. But that means duplicating some of the session-building logic in a separate part of your pipeline. And, more important, it means that you no longer have idempotent output, and you’ll thus need to jump through a bunch of extra hoops if you want to maintain end-to-end exactly-once semantics. Far better would be for the pipeline to simply tell you which sessions were removed and which were added in their place. This is what retractions give you.

To see this in action (and in SQL), let’s modify our example pipeline to compute session windows with a gap duration of one minute. For simplicity and clarity, we go back to using the default per-record trigger. Note that I’ve also shifted a few of the data points within processing time for these session examples to make the diagram cleaner; event-time timestamps remain the same. The updated dataset looks like this (with shifted processing-time timestamps highlighted in yellow):

12:00> SELECT STREAM Score, EventTime, Sys.MTime as ProcTime 
       FROM UserScoresForSessions;
--------------------------------
| Score | EventTime | ProcTime |
--------------------------------
|     5 |  12:00:26 | 12:05:19 |
|     7 |  12:02:26 | 12:05:39 |
|     3 |  12:03:39 | 12:06:13 |
|     4 |  12:04:19 | 12:06:46 |  # Originally 12:06:39
|     3 |  12:06:39 | 12:07:19 |
|     8 |  12:03:06 | 12:07:33 |  # Originally 12:07:06
|     8 |  12:07:26 | 12:08:13 |  # Originally 12:08:39
|     9 |  12:01:26 | 12:08:19 |
|     1 |  12:07:46 | 12:09:00 |
........ [12:00, 12:10] ........
To begin with, let’s look at the pipeline without retractions. After it’s clear why that pipeline is problematic for the use case of writing incremental sessions to a key/value store, we’ll then look at the version with retractions.

The Beam code for the nonretracting pipeline would look something like Example 8-7. Figure 8-13 shows the results.

Example 8-7. Session windows with per-record triggering and accumulation but no retractions
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(Sessions.withGapDuration(ONE_MINUTE))
               .triggering(Repeatedly(AfterCount(1))
               .accumulatingFiredPanes())
  .apply(Sum.integersPerKey());
Figure 8-13. Session window summation with accumulation but no retractions
And finally, rendered in SQL, the output stream would look like this:

12:00> SELECT STREAM SUM(Score) as Total,
         SESSION(EventTime, INTERVAL '1' MINUTE) as Window,
         CURRENT_TIMESTAMP as EmitTime
       FROM UserScoresForSessions
       GROUP BY Team, SESSION(EventTime, INTERVAL '1' MINUTE);
-------------------------------------------
| Total | Window               | EmitTime |
-------------------------------------------
| 5     | [12:00:26, 12:01:26) | 12:05:19 |
| 7     | [12:02:26, 12:03:26) | 12:05:39 |
| 3     | [12:03:39, 12:04:39) | 12:06:13 |
| 7     | [12:03:39, 12:05:19) | 12:06:46 |
| 3     | [12:06:39, 12:07:39) | 12:07:19 |
| 22    | [12:02:26, 12:05:19) | 12:07:33 |
| 11    | [12:06:39, 12:08:26) | 12:08:13 |
| 36    | [12:00:26, 12:05:19) | 12:08:19 |
| 12    | [12:06:39, 12:08:46) | 12:09:00 |
............. [12:00, 12:10] ..............
The important thing to notice in here (in the animation as well as the SQL rendering) is what the stream of incremental sessions looks like. From our holistic viewpoint, it’s pretty easy to visually identify in the animation which later sessions supersede those that came before. But imagine receiving elements in this stream one by one (as in the SQL listing) and needing to write them to HBase in a way that eventually results in the HBase table containing only the two final sessions (with values 36 and 12). How would you do that? Well, you’d need to do a bunch of read-modify-write operations to read all of the existing sessions for a key, compare them with the new session, determine which ones overlap, issue deletes for the obsolete sessions, and then finally issue a write for the new session—all at significant additional cost, and with a loss of idempotence, which would ultimately leave you unable to provide end-to-end, exactly-once semantics. It’s just not practical.

Contrast this then with the same pipeline, but with retractions enabled, as demonstrated in Example 8-8 and depicted in Figure 8-14.

Example 8-8. Session windows with per-record triggering, accumulation, and retractions
PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(Sessions.withGapDuration(ONE_MINUTE))
               .triggering(Repeatedly(AfterCount(1))
               .accumulatingAndRetractingFiredPanes())
  .apply(Sum.integersPerKey());
Figure 8-14. Session window summation with accumulation and retractions
And, lastly, in SQL form. For the SQL version, we’re assuming that the system is using retractions under the covers by default, and individual retraction rows are then materialized in the stream any time we request the special Sys.Undo column.16 As I described originally, the value of that column is that it allows us to distinguish retraction rows (labeled undo in the Sys.Undo column) from normal rows (unlabeled in the Sys.Undo column here for clearer contrast, though they could just as easily be labeled redo, instead):

12:00> SELECT STREAM SUM(Score) as Total,
         SESSION(EventTime, INTERVAL '1' MINUTE) as Window,
         CURRENT_TIMESTAMP as EmitTime,
         Sys.Undo as Undo
       FROM UserScoresForSessions
       GROUP BY Team, SESSION(EventTime, INTERVAL '1' MINUTE);
--------------------------------------------------
| Total | Window               | EmitTime | Undo |
--------------------------------------------------
| 5     | [12:00:26, 12:01:26) | 12:05:19 |      |
| 7     | [12:02:26, 12:03:26) | 12:05:39 |      |
| 3     | [12:03:39, 12:04:39) | 12:06:13 |      |
| 3     | [12:03:39, 12:04:39) | 12:06:46 | undo |
| 7     | [12:03:39, 12:05:19) | 12:06:46 |      |
| 3     | [12:06:39, 12:07:39) | 12:07:19 |      |
| 7     | [12:02:26, 12:03:26) | 12:07:33 | undo |
| 7     | [12:03:39, 12:05:19) | 12:07:33 | undo |
| 22    | [12:02:26, 12:05:19) | 12:07:33 |      |
| 3     | [12:06:39, 12:07:39) | 12:08:13 | undo |
| 11    | [12:06:39, 12:08:26) | 12:08:13 |      |
| 5     | [12:00:26, 12:01:26) | 12:08:19 | undo |
| 22    | [12:02:26, 12:05:19) | 12:08:19 | undo |
| 36    | [12:00:26, 12:05:19) | 12:08:19 |      |
| 11    | [12:06:39, 12:08:26) | 12:09:00 | undo |
| 12    | [12:06:39, 12:08:46) | 12:09:00 |      |
................. [12:00, 12:10] .................
With retractions included, the sessions stream no longer just includes new sessions, but also retractions for the old sessions that have been replaced. With this stream, it’s trivial17 to properly build up the set of sessions in HBase over time: you simply write new sessions as they arrive (unlabeled redo rows) and delete old sessions as they’re retracted (undo rows). Much better!

Discarding mode, or lack thereof
With this example, we’ve shown how simply and naturally you can incorporate retractions into SQL to provide both accumulating mode and accumulating and retracting mode semantics. But what about discarding mode?

For specific use cases such as very simple pipelines that partially aggregate high-volume input data via a single grouping operation and then write them into a storage system, which itself supports aggregation (e.g., a database-like system), discarding mode can be extremely valuable as a resource-saving option. But outside of those relatively narrow use cases, discarding mode is confusing and error-prone. As such, it’s probably not worth incorporating directly into SQL. Systems that need it can provide it as an option outside of the SQL language itself. Those that don’t can simply provide the more natural default of accumulating and retracting mode, with the option to ignore retractions when they aren’t needed.

Summary
This has been a long journey but a fascinating one. We’ve covered a ton of information in this chapter, so let’s take a moment to reflect on it all.

First, we reasoned that the key difference between streaming and nonstreaming data processing is the added dimension of time. We observed that relations (the foundational data object from relational algebra, which itself is the basis for SQL) themselves evolve over time, and from that derived the notion of a TVR, which captures the evolution of a relation as a sequence of classic snapshot relations over time. From that definition, we were able to see that the closure property of relational algebra remains intact in a world of TVRs, which means that the entire suite of relational operators (and thus SQL constructs) continues to function as one would expect as we move from a world of point-in-time snapshot relations into a streaming-compatible world of TVRs.

Second, we explored the biases inherent in both the Beam Model and the classic SQL model as they exist today, coming to the conclusion that Beam has a stream-oriented approach, whereas SQL takes a table-oriented approach.

And finally, we looked at the hypothetical language extensions needed to add support for robust stream processing to SQL,18 as well as some carefully chosen defaults that can greatly decrease the need for those extensions to be used:

Table/stream selection
Given that any time-varying relation can be rendered in two different ways (table or stream), we need the ability to choose which rendering we want when materializing the results of a query. We introduced the TABLE, STREAM, and TVR keywords to provide a nice explicit way to choose the desired rendering.

Even better is not needing to explicitly specify a choice, and that’s where good defaults come in. If all the inputs are tables, a good default is for the output to be a table, as well; this gives you the classic relational query behavior everyone is accustomed to. Conversely, if any of the inputs are streams, a reasonable default is for the output to be a stream, as well.

Windowing
Though you can declare some types of simple windows declaratively using existing SQL constructs, there is still value in having explicit windowing operators:

Windowing operators encapsulate the window-computation math.

Windowing allows the concise expression of complex, dynamic groupings like sessions.

Thus, the addition of simple windowing constructs for use in grouping can help make queries less error prone while also providing capabilities (like sessions) that are impractical to express in declarative SQL as it exists today.

Watermarks
This isn’t so much a SQL extension as it is a system-level feature. If the system in question integrates watermarks under the covers, they can be used in conjunction with triggers to generate streams containing a single, authoritative version of a row only after the input for that row is believed to be complete. This is critical for use cases in which it’s impractical to poll a materialized view table for results, and instead the output of the pipeline must be consumed directly as a stream. Examples are notifications and anomaly detection.

Triggers
Triggers define the shape of a stream as it is created from a TVR. If unspecified, the default should be per-record triggering, which provides straightforward and natural semantics matching those of materialized views. Beyond the default, there are essentially two main types of useful triggers:

Watermark triggers, for yielding a single output per window when the inputs to that window are believed to be complete.

Repeated delay triggers, for providing periodic updates.

Combinations of those two can also be useful, especially in the case of heuristic watermarks, to provide the early/on-time/late pattern we saw earlier.

Special system columns
When consuming a TVR as a stream, there are some interesting metadata that can be useful and which are most easily exposed as system-level columns. We looked at four:

Sys.MTime
The processing time at which a given row was last modified in a TVR.

Sys.EmitTiming
The timing of the row emit relative to the watermark (early, on-time, late).

Sys.EmitIndex
The zero-based index of the emit version for this row.19

Sys.Undo
Whether the row is a normal row or a retraction (undo). By default, the system should operate with retractions under the covers, as is necessary any time a series of more than one grouping operation might exist. If the Sys.Undo column is not projected when rendering a TVR as a stream, only normal rows will be returned, providing a simple way to toggle between accumulating and accumulating and retracting modes.

Stream processing with SQL doesn’t need to be difficult. In fact, stream processing in SQL is quite common already in the form of materialized views. The important pieces really boil down to capturing the evolution of datasets/relations over time (via time-varying relations), providing the means of choosing between physical table or stream representations of those time-varying relations, and providing the tools for reasoning about time (windowing, watermarks, and triggers) that we’ve been talking about throughout this book. And, critically, you need good defaults to minimize how often these extensions need to be used in practice.

1 What I mean by “valid relation” here is simply a relation for which the application of a given operator is well formed. For example, for the SQL query SELECT x FROM y, a valid relation y would be any relation containing an attribute/column named x. Any relation not containing a such-named attribute would be invalid and, in the case of a real database system, would yield a query execution error.

2 Much credit to Julian Hyde for this name and succinct rendering of the concept.

3 Note that the Sys.Undo name used here is riffing off the concise undo/redo nomenclature from Apache Flink, which I think is a very clean way to capture the ideas of retraction and nonretraction rows.

4 Now, in this example, it’s not too difficult to figure out that the new value of 8 should replace the old value of 7, given that the mapping is 1:1. But we’ll see a more complicated example later on when we talk about sessions that is much more difficult to handle without having retractions as a guide.

5 And indeed, this is a key point to remember. There are some systems that advocate treating streams and tables as identical, claiming that we can simply treat streams like never-ending tables. That statement is accurate inasmuch as the true underlying primitive is the time-varying relation, and all relational operations may be applied equally to any time-varying relation, regardless of whether the actual physical manifestation is a stream or a table. But that sort of approach conflates the two very different types of views that tables and streams provide for a given time-varying relation. Pretending that two very different things are the same might seem simple on the surface, but it’s not a road toward understanding, clarity, and correctness.

6 Here referring to tables in the sense of tables that can vary over time; that is, the table-based TVRs we’ve been looking at.

7 This one courtesy Julian Hyde.

8 Though there are a number of efforts in flight across various projects that are trying to simplify the specification of triggering/ungrouping semantics. The most compelling proposal, made independently within both the Flink and Beam communities, is that triggers should simply be specified at the outputs of a pipeline and automatically propagated up throughout the pipeline. In this way, one would describe only the desired shape of the streams that actually create materialized output; the shape of all other streams in the pipeline would be implicitly derived from there.

9 Though, of course, a single SQL query has vastly more expressive power than a single MapReduce, given the far less-confining set of operations and composition options available.

10 Note that we’re speaking conceptually here; there are of course a multitude of optimizations that can be applied in actual execution; for example, looking up specific rows via an index rather than scanning the entire table.

11 It’s been brought to my attention multiple times that the “MATERIALIZED” aspect of these queries is just an optimization: semantically speaking, these queries could just as easily be replaced with generic CREATE VIEW statements, in which case the database might instead simply rematerialize the entire view each time it is referenced. This is true. The reason I use the MATERIALIZED variant here is that the semantics of a materialized view are to incrementally update the view table in response to a stream of changes, which is indicative of the streaming nature behind them. That said, the fact that you can instead provide a similar experience by re-executing a bounded query each time a view is accessed provides a nice link between streams and tables as well as a link between streaming systems and the way batch systems have been historically used for processing data that evolves over time. You can either incrementally process changes as they occur or you can reprocess the entire input dataset from time to time. Both are valid ways of processing an evolving table of data.

12 Though it’s probably fair to say that SQL’s table bias is likely an artifact of SQL’s roots in batch processing.

13 For some use cases, capturing and using the current processing time for a given record as its event time going forward can be useful (for example, when logging events directly into a TVR, where the time of ingress is the natural event time for that record).

14 Maths are easy to get wrong.

15 It’s sufficient for retractions to be used by default and not simply always because the system only needs the option to use retractions. There are specific use cases; for example, queries with a single grouping operation whose results are being written into an external storage system that supports per-key updates, where the system can detect retractions are not needed and disable them as an optimization.

16 Note that it’s a little odd for the simple addition of a new column in the SELECT statement to result in a new rows appearing in a query. A fine alternative approach would be to require Sys.Undo rows to be filtered out via a WHERE clause when not needed.

17 Not that this triviality applies only in cases for which eventual consistency is sufficient. If you need to always have a globally coherent view of all sessions at any given time, you must 1) be sure to write/delete (via tombstones) each session at its emit time, and 2) only ever read from the HBase table at a timestamp that is less than the output watermark from your pipeline (to synchronize reads against the multiple, independent writes/deletes that happen when sessions merge). Or better yet, cut out the middle person and serve the sessions from your state tables directly.

18 To be clear, they’re not all hypothetical. Calcite has support for the windowing constructs described in this chapter.

19 Note that the definition of “index” becomes complicated in the case of merging windows like sessions. A reasonable approach is to take the maximum of all of the previous sessions being merged together and increment by one.