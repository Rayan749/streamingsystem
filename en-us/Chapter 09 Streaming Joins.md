# Chapter 9. Streaming Joins
When I first began learning about joins, it was an intimidating topic; LEFT, OUTER, SEMI, INNER, CROSS: the language of joins is expressive and expansive. Add on top of that the dimension of time that streaming brings to the table, and you’re left with what appears to be a challengingly complex topic. The good news is that joins really aren’t the frightening beast with nasty, pointy teeth that they might initially appear to be. As is the case with so many other complex topics, after you understand the central ideas and themes of joins, the broader landscape that’s built on top of these basics suddenly becomes so much more accessible. So please join me now as we explore the fascinating topic of...well, joins.

All Your Joins Are Belong to Streaming
What does it mean to join two datasets? We understand intuitively that joins are just a specific type of grouping operation: by joining together data that share some property (i.e., key), we collect together some number of previously unrelated individual data elements into a group of related elements. And as we learned in Chapter 6, grouping operations always consume a stream and yield a table. Knowing these two things, it’s only a small leap to then arrive at the conclusion that forms the basis for this entire chapter: at their hearts, all joins are streaming joins.

What’s great about this fact is that it actually makes the topic of streaming joins that much more tractable. All of the tools we’ve learned for reasoning about time within the context of streaming grouping operations (windowing, watermarks, triggers, etc.) continue to apply in the case of streaming joins. What’s perhaps intimidating is that adding streaming to the mix seems like it could only serve to complicate things. But as you’ll see in the examples that follow, there’s a certain elegant simplicity and consistency to modeling all joins as streaming joins. Instead of feeling like there are a confounding multitude of different join approaches, it becomes clear that nearly all types of joins really boil down to minor variations on the same pattern. In the end, that clarity of insight helps makes joins (streaming or otherwise) much less intimidating.

To give us something concrete to reason about, let’s consider a number of different types of joins as they’re applied the following datasets, conveniently named Left and Right to match the common nomenclature:

12:10> SELECT TABLE * FROM Left;        12:10> SELECT TABLE * FROM Right;
--------------------                    --------------------
| Num | Id | Time  |                    | Num | Id | Time  |
--------------------                    --------------------
| 1   | L1 | 12:02 |                    | 2   | R2 | 12:01 |
| 2   | L2 | 12:06 |                    | 3   | R3 | 12:04 |
| 3   | L3 | 12:03 |                    | 4   | R4 | 12:05 |
--------------------                    --------------------
Each contains three columns:

Num
A single number.

Id
A portmanteau of the first letter in the name of the corresponding table (“L” or “R”) and the Num, thus providing a way to uniquely identify the source of a given cell in join results.

Time
The arrival time of the given record in the system, which becomes important when considering streaming joins.

To keep things simple, note that our initial datasets will have strictly unique join keys. When we get to SEMI joins, we’ll introduce some more complicated datasets to highlight join behavior in the presence of duplicate keys.

We first look at unwindowed joins in a great deal of depth because windowing often affects join semantics in only a minor way. After we exhaust our appetite for unwindowed joins, we then touch upon some of the more interesting points of joins in a windowed context.

Unwindowed Joins
It’s a popular myth that streaming joins over unbounded data always require windowing. But by applying the concepts we learned in Chapter 6, we can see that’s simply not true. Joins (both windowed and unwindowed) are simply another type of grouping operation, and grouping operations yield tables. Thus, if we want to consume the table created by an unwindowed join (or, equivalently, joins within a single global window covering all of time) as a stream, we need only apply an ungrouping (or trigger) operation that isn’t of the “wait until we’ve seen all the input” variety. Windowing the join into a nonglobal window and using a watermark trigger (i.e., a “wait until we’ve seen all the input in a finite temporal chunk of the stream” trigger) is indeed one option, but so is triggering on every record (i.e., materialized view semantics) or periodically as processing time advances, regardless of whether the join is windowed or not. Because it makes the examples easy to follow, we assume the use of an implicit default per-record trigger in all of the following unwindowed join examples that observe the join results as a stream.

Now, onto joins themselves. ANSI SQL defines five types of joins: FULL OUTER, LEFT OUTER, RIGHT OUTER, INNER, and CROSS. We look at the first four in depth, and discuss the last only briefly in the next paragraph. We also touch on two other interesting, but less-often encountered (and less well supported, at least using standard syntax) variations: ANTI and SEMI joins.

On the surface, it sounds like a lot of variations. But as you’ll see, there’s really only one type of join at the core: the FULL OUTER join. A CROSS join is just a FULL OUTER join with a vacuously true join predicate; that is, it returns every possible pairing of a row from the left table with a row from the right table. All of the other join variations simply reduce down to some logical subset of the FULL OUTER join.1 As a result, after you understand the commonality between all the different join types, it becomes a lot easier to keep them all in your head. It also makes reasoning about them in the context of streaming all that much simpler.

One last note here before we get started: we’ll be primarily considering equi joins with at most 1:1 cardinality, by which I mean joins in which the join predicate is an equality statement and there is at most one matching row on each side of the join. This keeps the examples simple and concise. When we get to SEMI joins, we’ll expand our example to consider joins with arbitrary N:M cardinality, which will let us observe the behavior of more arbitrary predicate joins.

FULL OUTER
Because they form the conceptual foundation for each of the other variations, we first look at FULL OUTER joins. Outer joins embody a rather liberal and optimistic interpretation of the word “join”: the result of FULL OUTER–joining two datasets is essentially the full list of rows in both datasets,2 with rows in the two datasets that share the same join key combined together, but unmatched rows for either side included unjoined.

For example, if we FULL OUTER–join our two example datasets into a new relation containing only the joined IDs, the result would look something like this:

12:10> SELECT TABLE 
         Left.Id as L, 
         Right.Id as R,
       FROM Left FULL OUTER JOIN Right
       ON L.Num = R.Num;
---------------
| L    | R    |
---------------
| L1   | null | 
| L2   | R2   |
| L3   | R3   |
| null | R4   |
---------------
We can see that the FULL OUTER join includes both rows that satisfied the join predicate (e.g., “L2, R2” and “L3, R3”), but it also includes partial rows that failed the predicate (e.g., “L1, null” and “null, R4”, where the null is signaling the unjoined portion of the data).

Of course, that’s just a point-in-time snapshot of this FULL OUTER–join relation, taken after all of the data have arrived in the system. We’re here to learn about streaming joins, and streaming joins by definition involve the added dimension of time. As we know from Chapter 8, if we want to understand how a given dataset/relation changes over time, we want to speak in terms of time-varying relations (TVRs). So to best understand how the join evolves over time, let’s look now at the full TVR for this join (with changes between each snapshot relation highlighted in yellow):

12:10> SELECT TVR
         Left.Id as L,
         Right.Id as R,
       FROM Left FULL OUTER JOIN Right
       ON L.Num = R.Num;
-------------------------------------------------------------------------
| [-inf, 12:01)   | [12:01, 12:02)  | [12:02, 12:03)  | [12:03, 12:04)  |
| --------------- | --------------- | --------------- | --------------- |
|                 | L               | R               |                 |
| --------------- | --------------- | --------------- | --------------- |
| --------------- |                 | null            | R2              |
|                 | --------------- |                 | null            |
|                 |                 | --------------- |                 |
|                 |                 |                 | --------------- |
-------------------------------------------------------------------------
| [12:04, 12:05)  | [12:05, 12:06)  | [12:06, 12:07)  |
| --------------- | --------------- | --------------- |
|                 | L               | R               |
| --------------- | --------------- | --------------- |
|                 | L1              | null            |
|                 | null            | L2              |
|                 | L3              | L3              |
| --------------- |                 | null            |
|                 | --------------- | --------------- |
-------------------------------------------------------
And, as you might then expect, the stream rendering of this TVR would capture the specific deltas between each of those snapshots:

12:00> SELECT STREAM 
         Left.Id as L,
         Right.Id as R, 
         CURRENT_TIMESTAMP as Time,
         Sys.Undo as Undo
       FROM Left FULL OUTER JOIN Right
       ON L.Num = R.Num;
------------------------------
| L    | R    | Time  | Undo |
------------------------------
| null | R2   | 12:01 |      |
| L1   | null | 12:02 |      |
| L3   | null | 12:03 |      |
| L3   | null | 12:04 | undo |
| L3   | R3   | 12:04 |      |
| null | R4   | 12:05 |      |
| null | R2   | 12:06 | undo |
| L2   | R2   | 12:06 |      |
....... [12:00, 12:10] .......
Note the inclusion of the Time and Undo columns, to highlight the times when given rows materialize in the stream, and also call out instances when an update to a given row first results in a retraction of the previous version of that row. The undo/retraction rows are critical if this stream is to capture a full-fidelity view of the TVR over time.

So, although each of these three renderings of the join (table, TVR, stream) are distinct from one another, it’s also pretty clear how they’re all just different views on the same data: the table snapshot shows us the overall dataset as it exists after all the data have arrived, and the TVR and stream versions capture (in their own ways) the evolution of the entire relation over the course of its existence.

With that basic familiarity of FULL OUTER joins in place, we now understand all of the core concepts of joins in a streaming context. No windowing needed, no custom triggers, nothing particularly painful or unintuitive. Just a per-record evolution of the join over time, as you would expect. Even better, all of the other types of joins are just variations on this theme (conceptually, at least), essentially just an additional filtering operation performed on the per-record stream of the FULL OUTER join. Let’s now look at each of them in more detail.

LEFT OUTER
LEFT OUTER joins are a just a FULL OUTER join with any unjoined rows from the right dataset removed. This is most clearly seen by taking the original FULL OUTER join and graying out the rows that would be filtered. For a LEFT OUTER join, that would look like the following, where every row with an unjoined left side is filtered out of the original FULL OUTER join:

                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left LEFT OUTER JOIN Right          FROM Left LEFT OUTER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | null | R2   | 12:01 |      |
| L2   | R2   |                          | L1   | null | 12:02 |      |
| L3   | R3   |                          | L3   | null | 12:03 |      |
| null | R4   |                          | L3   | null | 12:04 | undo |
---------------                          | L3   | R3   | 12:04 |      |
                                         | null | R4   | 12:05 |      |
                                         | null | R2   | 12:06 | undo |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
To see what the table and stream would actually look like in practice, let’s look at the same queries again, but this time with the grayed-out rows omitted entirely:

                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left LEFT OUTER JOIN Right          FROM Left LEFT OUTER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | L1   | null | 12:02 |      |
| L2   | R2   |                          | L3   | null | 12:03 |      |
| L3   | R3   |                          | L3   | null | 12:04 | undo |
---------------                          | L3   | R3   | 12:04 |      |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
RIGHT OUTER
RIGHT OUTER joins are the converse of a left join: all unjoined rows from the left dataset in the full outer join are right out, *cough*, removed:

                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left RIGHT OUTER JOIN Right         FROM Left RIGHT OUTER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | null | R2   | 12:01 |      |
| L2   | R2   |                          | L1   | null | 12:02 |      |
| L3   | R3   |                          | L3   | null | 12:03 |      |
| null | R4   |                          | L3   | null | 12:04 | undo |
---------------                          | L3   | R3   | 12:04 |      |
                                         | null | R4   | 12:05 |      |
                                         | null | R2   | 12:06 | undo |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
And here we see how the queries rendered as the actual RIGHT OUTER join would appear:

                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left RIGHT OUTER JOIN Right         FROM Left RIGHT OUTER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L2   | R2   |                          | null | R2   | 12:01 |      |
| L3   | R3   |                          | L3   | R3   | 12:04 |      |
| null | R4   |                          | null | R4   | 12:05 |      |
---------------                          | null | R2   | 12:06 | undo |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
INNER
INNER joins are essentially the intersection of the LEFT OUTER and RIGHT OUTER joins. Or, to think of it subtractively, the rows removed from the original FULL OUTER join to create an INNER join are the union of the rows removed from the LEFT OUTER and RIGHT OUTER joins. As a result, all rows that remain unjoined on either side are absent from the INNER join:

                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left INNER JOIN Right               FROM Left INNER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | null | R2   | 12:01 |      |
| L2   | R2   |                          | L1   | null | 12:02 |      |
| L3   | R3   |                          | L3   | null | 12:03 |      |
| null | R4   |                          | L3   | null | 12:04 | undo |
---------------                          | L3   | R3   | 12:04 |      |
                                         | null | R4   | 12:05 |      |
                                         | null | R2   | 12:06 | undo |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
And again, more succinctly rendered as the INNER join would look in reality:

                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left INNER JOIN Right               FROM Left INNER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L2   | R2   |                          | L3   | R3   | 12:04 |      |
| L3   | R3   |                          | L2   | R2   | 12:06 |      |
---------------                          ....... [12:00, 12:10] .......
Given this example, you might be inclined to think retractions never play a part in INNER join streams because they were all filtered out in this example. But imagine if the value in the Left table for the row with a Num of 3 were updated from “L3” to “L3v2” at 12:07. In addition to resulting in a different value on the left side for our final TABLE query (again performed at 12:10, which is after the update to row 3 on the Left arrived), it would also result in a STREAM that captures both the removal of the old value via a retraction and the addition of the new value:

                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM LeftV2 INNER JOIN Right             FROM LeftV2 INNER JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                           ------------------------------
| L    | R    |                           | L    | R    | Time  | Undo |
---------------                           ------------------------------
| L2   | R2   |                           | L3   | R3   | 12:04 |      |
| L3v2 | R3   |                           | L2   | R2   | 12:06 |      |
---------------                           | L3   | R3   | 12:07 | undo | 
                                          | L3v2 | R3   | 12:07 |      |
                                          ....... [12:00, 12:10] .......
ANTI
ANTI joins are the obverse of the INNER join: they contain all of the unjoined rows. Not all SQL systems support a clean ANTI join syntax, but I’ll use the most straightforward one here for clarity:

                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left ANTI JOIN Right                FROM Left ANTI JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          -------------------------------
| L    | R    |                          | L    |    R | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | null | R2   | 12:01 |      |
| L2   | R2   |                          | L1   | null | 12:02 |      |
| L3   | R3   |                          | L3   | null | 12:03 |      |
| null | R4   |                          | L3   | null | 12:04 | undo |
---------------                          | L3   | R3   | 12:04 |      |
                                         | null | R4   | 12:05 |      |
                                         | null | R2   | 12:06 | undo |
                                         | L2   | R2   | 12:06 |      |
                                         ....... [12:00, 12:10] .......
What’s slightly interesting about the stream rendering of the ANTI join is that it ends up containing a bunch of false-starts and retractions for rows which eventually do end up joining; in fact, the ANTI join is as heavy on retractions as the INNER join is light. The more concise versions would look like this:

                                         12:00> SELECT STREAM Left.Id as L, 
12:10> SELECT TABLE                               Right.Id as R,
         Left.Id as L,                            Sys.EmitTime as Time, 
         Right.Id as R                            Sys.Undo as Undo 
       FROM Left ANTI JOIN Right               FROM Left ANTI JOIN Right
       ON L.Num = R.Num;                        ON L.Num = R.Num;
---------------                          ------------------------------
| L    | R    |                          | L    | R    | Time  | Undo |
---------------                          ------------------------------
| L1   | null |                          | null | R2   | 12:01 |      |
| null | R4   |                          | L1   | null | 12:02 |      |
---------------                          | L3   | null | 12:03 |      | 
                                         | L3   | null | 12:04 | undo |
                                         | null | R4   | 12:05 |      |
                                         | null | R2   | 12:06 | undo |
                                         ....... [12:00, 12:10] .......
SEMI
We now come to SEMI joins, and SEMI joins are kind of weird. At first glance, they basically look like inner joins with one side of the joined values being dropped. And, indeed, in cases for which the cardinality relationship of <side-being-kept>:<side-being-dropped> is N:M with M ≤ 1, this works (note that we’ll be using kept=Left, dropped=Right for all the examples that follow). For example, on the Left and Right datasets we’ve used so far (which had cardinalities of 0:1, 1:0, and 1:1 for the joined data), the INNER and SEMI join variations look identical:

12:10> SELECT TABLE            12:10> SELECT TABLE
  Left.Id as L                   Left.Id as L
FROM Left INNER JOIN           FROM Left SEMI JOIN
Right ON L.Num = R.Num;        Right ON L.Num = R.Num;
---------------                ---------------
| L    | R    |                | L    | R    |
---------------                ---------------
| L1   | null |                | L1   | null |
| L2   | R2   |                | L2   | R2   |
| L3   | R3   |                | L3   | R3   |
| null | R4   |                | null | R4   |
---------------                ---------------
However, there’s an additional subtlety to SEMI joins in the case of N:M cardinality with M > 1: because the values on the M side are not being returned, the SEMI join simply predicates the join condition on there being any matching row on the right, rather than repeatedly yielding a new result for every matching row.

To see this clearly, let’s switch to a slightly more complicated pair of input relations that highlight the N:M join cardinality of the rows contained therein. In these relations, the N_M column states what the cardinality relationship of rows is between the left and right sides, and the Id column (as before) provides an identifier that is unique for each row in each of the input relations:

12:15> SELECT TABLE * FROM LeftNM;    12:15> SELECT TABLE * FROM RightNM;
---------------------                 ---------------------
| N_M | Id  |  Time |                 | N_M | Id  |  Time |
---------------------                 ---------------------
| 1:0 | L2  | 12:07 |                 | 0:1 | R1  | 12:02 |
| 1:1 | L3  | 12:01 |                 | 1:1 | R3  | 12:14 |
| 1:2 | L4  | 12:05 |                 | 1:2 | R4A | 12:03 |
| 2:1 | L5A | 12:09 |                 | 1:2 | R4B | 12:04 |
| 2:1 | L5B | 12:08 |                 | 2:1 | R5  | 12:06 |
| 2:2 | L6A | 12:12 |                 | 2:2 | R6A | 12:11 |
| 2:2 | L6B | 12:10 |                 | 2:2 | R6B | 12:13 |
---------------------                 ---------------------
With these inputs, the FULL OUTER join expands to look like these:

                                       12:00> SELECT STREAM
                                                COALESCE(LeftNM.N_M, 
12:15> SELECT TABLE                                      RightNM.N_M) as N_M, 
         COALESCE(LeftNM.N_M,                     LeftNM.Id as L,
                  RightNM.N_M) as N_M,            RightNM.Id as R, 
         LeftNM.Id as L,                        Sys.EmitTime as Time, 
         RightNM.Id as R,                         Sys.Undo as Undo
       FROM LeftNM                            FROM LeftNM 
         FULL OUTER JOIN RightNM                FULL OUTER JOIN RightNM
         ON LeftNM.N_M = RightNM.N_M;           ON LeftNM.N_M = RightNM.N_M;
---------------------                  ------------------------------------
| N_M | L    | R    |                  | N_M | L    | R    | Time  | Undo |
---------------------                  ------------------------------------
| 0:1 | null | R1   |                  | 1:1 | L3   | null | 12:01 |      |
| 1:0 | L2   | null |                  | 0:1 | null | R1   | 12:02 |      |
| 1:1 | L3   | R3   |                  | 1:2 | null | R4A  | 12:03 |      |
| 1:2 | L4   | R4A  |                  | 1:2 | null | R4B  | 12:04 |      |
| 1:2 | L4   | R4B  |                  | 1:2 | null | R4A  | 12:05 | undo |
| 2:1 | L5A  | R5   |                  | 1:2 | null | R4B  | 12:05 | undo |
| 2:1 | L5B  | R5   |                  | 1:2 | L4   | R4A  | 12:05 |      |
| 2:2 | L6A  | R6A  |                  | 1:2 | L4   | R4B  | 12:05 |      |
| 2:2 | L6A  | R6B  |                  | 2:1 | null | R5   | 12:06 |      |
| 2:2 | L6B  | R6A  |                  | 1:0 | L2   | null | 12:07 |      |
| 2:2 | L6B  | R6B  |                  | 2:1 | null | R5   | 12:08 | undo |
---------------------                  | 2:1 | L5B  | R5   | 12:08 |      |
                                       | 2:1 | L5A  | R5   | 12:09 |      |
                                       | 2:2 | L6B  | null | 12:10 |      |
                                       | 2:2 | L6B  | null | 12:11 | undo |
                                       | 2:2 | L6B  | R6A  | 12:11 |      |
                                       | 2:2 | L6A  | R6A  | 12:12 |      |
                                       | 2:2 | L6A  | R6B  | 12:13 |      |
                                       | 2:2 | L6B  | R6B  | 12:13 |      |
                                       | 1:1 | L3   | null | 12:14 | undo |
                                       | 1:1 | L3   | R3   | 12:14 |      |
                                       .......... [12:00, 12:15] ..........
As a side note, one additional benefit of these more complicated datasets is that the multiplicative nature of joins when there are multiple rows on each side matching the same predicate begins to become more clear (e.g., the “2:2” rows, which expand from two rows in each the inputs to four rows in the output; if the dataset had a set of “3:3” rows, they’d expand from three rows in each of the inputs to nine rows in the output, and so on).

But back to the subtleties of SEMI joins. With these datasets, it becomes much clearer what the difference between the filtered INNER join and the SEMI join is: the INNER join yields duplicate values for any of the rows where the N:M cardinality has M > 1, whereas the SEMI join doesn’t (note that I’ve highlighted the duplicate rows in the INNER join version in red, and included in gray the portions of the full outer join that are omitted in the respective INNER and SEMI versions):

12:15> SELECT TABLE                       12:15> SELECT TABLE
         COALESCE(LeftNM.N_M,                      COALESCE(LeftNM.N_M,
                  RightNM.N_M) as N_M,                      RightNM.N_M) as N_M,
         LeftNM.Id as L                            LeftNM.Id as L
       FROM LeftNM INNER JOIN RightNM            FROM LeftNM SEMI JOIN RightNM
       ON LeftNM.N_M = RightNM.N_M;              ON LeftNM.N_M = RightNM.N_M;
---------------------                     ---------------------
| N_M | L    | R    |                     | N_M | L    | R    |
---------------------                     ---------------------
| 0:1 | null | R1   |                     | 0:1 | null | R1   |
| 1:0 | L2   | null |                     | 1:0 | L2   | null |
| 1:1 | L3   | R3   |                     | 1:1 | L3   | R3   |
| 1:2 | L4   | R5A  |                     | 1:2 | L4   | R5A  |
| 1:2 | L4   | R5B  |                     | 1:2 | L4   | R5B  |
| 2:1 | L5A  | R5   |                     | 2:1 | L5A  | R5   |
| 2:1 | L5B  | R5   |                     | 2:1 | L5B  | R5   |
| 2:2 | L6A  | R6A  |                     | 2:2 | L6A  | R6A  |
| 2:2 | L6A  | R6B  |                     | 2:2 | L6A  | R6B  |
| 2:2 | L6B  | R6A  |                     | 2:2 | L6B  | R6A  |
| 2:2 | L6B  | R6B  |                     | 2:2 | L6B  | R6B  |
---------------------                     ---------------------
Or, rendered more succinctly:

12:15> SELECT TABLE                       12:15> SELECT TABLE
         COALESCE(LeftNM.N_M,                      COALESCE(LeftNM.N_M,
                  RightNM.N_M) as N_M,                      RightNM.N_M) as N_M,
         LeftNM.Id as L                            LeftNM.Id as L
       FROM LeftNM INNER JOIN RightNM            FROM LeftNM SEMI JOIN RightNM
       ON LeftNM.N_M = RightNM.N_M;              ON LeftNM.N_M = RightNM.N_M;
-------------                             -------------
| N_M | L   |                             | N_M | L   |
-------------                             -------------
| 1:1 | L3  |                             | 1:1 | L3  |
| 1:2 | L4  |                             | 1:2 | L4  |
| 1:2 | L4  |                             | 2:1 | L5A |
| 2:1 | L5A |                             | 2:1 | L5B |
| 2:1 | L5B |                             | 2:2 | L6A |
| 2:2 | L6A |                             | 2:2 | L6B |
| 2:2 | L6A |                             -------------
| 2:2 | L6B |
| 2:2 | L6B |
-------------
The STREAM renderings then provide a bit of context as to which rows are filtered out—they are simply the later-arriving duplicate rows (from the perspective of the columns being projected):

12:00> SELECT STREAM                        12:00> SELECT STREAM
         COALESCE(LeftNM.N_M,                        COALESCE(LeftNM.N_M,
                  RightNM.N_M) as N_M,                        RightNM.N_M) as N_M,
         LeftNM.Id as L                              LeftNM.Id as L
         Sys.EmitTime as Time,                       Sys.EmitTime as Time,
         Sys.Undo as Undo,                           Sys.Undo as Undo,
       FROM LeftNM INNER JOIN RightNM              FROM LeftNM SEMI JOIN RightNM
       ON LeftNM.N_M = RightNM.N_M;                ON LeftNM.N_M = RightNM.N_M;
------------------------------------        ------------------------------------
| N_M | L    | R    | Time  | Undo |        | N_M | L    | R    | Time  | Undo |
------------------------------------        ------------------------------------
| 1:1 | L3   | null | 12:01 |      |        | 1:1 | L3   | null | 12:01 |      |
| 0:1 | null | R1   | 12:02 |      |        | 0:1 | null | R1   | 12:02 |      |
| 1:2 | null | R4A  | 12:03 |      |        | 1:2 | null | R4A  | 12:03 |      |
| 1:2 | null | R4B  | 12:04 |      |        | 1:2 | null | R4B  | 12:04 |      |
| 1:2 | null | R4A  | 12:05 | undo |        | 1:2 | null | R4A  | 12:05 | undo |
| 1:2 | null | R4B  | 12:05 | undo |        | 1:2 | null | R4B  | 12:05 | undo |
| 1:2 | L4   | R4A  | 12:05 |      |        | 1:2 | L4   | R4A  | 12:05 |      |
| 1:2 | L4   | R4B  | 12:05 |      |        | 1:2 | L4   | R4B  | 12:05 |      |
| 2:1 | null | R5   | 12:06 |      |        | 2:1 | null | R5   | 12:06 |      |
| 1:0 | L2   | null | 12:07 |      |        | 1:0 | L2   | null | 12:07 |      |
| 2:1 | null | R5   | 12:08 | undo |        | 2:1 | null | R5   | 12:08 | undo |
| 2:1 | L5B  | R5   | 12:08 |      |        | 2:1 | L5B  | R5   | 12:08 |      |
| 2:1 | L5A  | R5   | 12:09 |      |        | 2:1 | L5A  | R5   | 12:09 |      |
| 2:2 | L6B  | null | 12:10 |      |        | 2:2 | L6B  | null | 12:10 |      |
| 2:2 | L6B  | null | 12:10 | undo |        | 2:2 | L6B  | null | 12:10 | undo |
| 2:2 | L6B  | R6A  | 12:11 |      |        | 2:2 | L6B  | R6A  | 12:11 |      |
| 2:2 | L6A  | R6A  | 12:12 |      |        | 2:2 | L6A  | R6A  | 12:12 |      |
| 2:2 | L6A  | R6B  | 12:13 |      |        | 2:2 | L6A  | R6B  | 12:13 |      |
| 2:2 | L6B  | R6B  | 12:13 |      |        | 2:2 | L6B  | R6B  | 12:13 |      |
| 1:1 | L3   | null | 12:14 | undo |        | 1:1 | L3   | null | 12:14 | undo |
| 1:1 | L3   | R3   | 12:14 |      |        | 1:1 | L3   | R3   | 12:14 |      |
.......... [12:00, 12:15] ..........        .......... [12:00, 12:15] ..........
And again, rendered succinctly:

12:00> SELECT STREAM                        12:00> SELECT STREAM
         COALESCE(LeftNM.N_M,                        COALESCE(LeftNM.N_M,
                  RightNM.N_M) as N_M,                        RightNM.N_M) as N_M,
         LeftNM.Id as L                              LeftNM.Id as L
         Sys.EmitTime as Time,                       Sys.EmitTime as Time,
         Sys.Undo as Undo,                           Sys.Undo as Undo,
       FROM LeftNM INNER JOIN RightNM              FROM LeftNM SEMI JOIN RightNM
       ON LeftNM.N_M = RightNM.N_M;                ON LeftNM.N_M = RightNM.N_M;
----------------------------                ----------------------------
| N_M | L   | Time  | Undo |                | N_M | L   | Time  | Undo |
----------------------------                ----------------------------
| 1:2 | L4  | 12:05 |      |                | 1:2 | L4  | 12:05 |      |
| 1:2 | L4  | 12:05 |      |                | 2:1 | L5B | 12:08 |      |
| 2:1 | L5B | 12:08 |      |                | 2:1 | L5A | 12:09 |      |
| 2:1 | L5A | 12:09 |      |                | 2:2 | L6B | 12:11 |      |
| 2:2 | L6B | 12:11 |      |                | 2:2 | L6A | 12:12 |      |
| 2:2 | L6A | 12:12 |      |                | 1:1 | L3  | 12:14 |      |
| 2:2 | L6A | 12:13 |      |                ...... [12:00, 12:15] ......
| 2:2 | L6B | 12:13 |      |
| 1:1 | L3  | 12:14 |      |
...... [12:00, 12:15] ......      
As we’ve seen over the course of a number of examples, there’s really nothing special about streaming joins. They function exactly as we might expect given our knowledge of streams and tables, with join streams capturing the history of the join over time as it evolves. This is in contrast to join tables, which simply capture a snapshot of the entire join as it exists at a specific point in time, as we’re perhaps more accustomed.

But, even more important, viewing joins through the lens of stream-table theory has lent some additional clarity. The core underlying join primitive is the FULL OUTER join, which is a stream → table grouping operation that collects together all the joined and unjoined rows in a relation. All of the other variants we looked at in detail (LEFT OUTER, RIGHT OUTER, INNER, ANTI, and SEMI) simply add an additional layer of filtering on the joined stream following the FULL OUTER join.3

Windowed Joins
Having looked at a variety of unwindowed joins, let’s next explore what windowing adds to the mix. I would argue that there are two motivations for windowing your joins:

To partition time in some meaningful way
An obvious case is fixed windows; for example, daily windows, for which events that occurred in the same day should be joined together for some business reason (e.g., daily billing tallies). Another might be limiting the range of time within a join for performance reasons. However, it turns out there are even more sophisticated (and useful) ways of partitioning time in joins, including one particularly interesting use case that no streaming system I’m aware of today supports natively: temporal validity joins. More on this in just a bit.

To provide a meaningful reference point for timing out a join
This is useful for a number of unbounded join situations, but it is perhaps most obviously beneficial for use cases like outer joins, for which it is unknown a priori if one side of the join will ever show up. For classic batch processing (including standard interactive SQL queries), outer joins are timed out only when the bounded input dataset has been fully processed. But when processing unbounded data, we can’t wait for all data to be processed. As we discussed in Chapters 2 and 3, watermarks provide a progress metric for gauging the completeness of an input source in event time. But to make use of that metric for timing out a join, we need some reference point to compare against. Windowing a join provides that reference by bounding the extent of the join to the end of the window. After the watermark passes the end of the window, the system may consider the input for the window complete. At that point, just as in the bounded join case, it’s safe to time out any unjoined rows and materialize their partial results.

That said, as we saw earlier, windowing is absolutely not a requirement for streaming joins. It makes a lot of sense in a many cases, but by no means is it a necessity.

In practice, most of the use cases for windowed joins (e.g., daily windows) are relatively straightforward and easy to extrapolate from the concepts we’ve learned up until now. To see why, we look briefly at what it means to apply fixed windows to some of the join examples we already encountered. After that, we spend the rest of this chapter investigating the much more interesting (and mind-bending) topic of temporal validity joins, looking first in detail at what I mean by temporal validity windows, and then moving on to looking at what joins mean within the context of such windows.

Fixed Windows
Windowing a join adds the dimension of time into the join criteria themselves. In doing so, the window serves to scope the set of rows being joined to only those contained within the window’s time interval. This is perhaps more clearly seen with an example, so let’s take our original Left and Right tables and window them into five-minute fixed windows:

12:10> SELECT TABLE *,                     12:10> SELECT TABLE *,
       TUMBLE(Time, INTERVAL '5' MINUTE)          TUMBLE(Time, INTERVAL '5' MINUTE)
       as Window FROM Left;                       as Window FROM Right
-------------------------------------      -------------------------------------
| Num | Id | Time  | Window         |      | Num | Id | Time  | Window         |
-------------------------------------      -------------------------------------
| 1   | L1 | 12:02 | [12:00, 12:05) |      | 2   | R2 | 12:01 | [12:00, 12:05) |
| 2   | L2 | 12:06 | [12:05, 12:10) |      | 3   | R3 | 12:04 | [12:00, 12:05) |
| 3   | L3 | 12:03 | [12:00, 12:05) |      | 4   | R4 | 12:05 | [12:05, 12:06) |
-------------------------------------      -------------------------------------
In our previous Left and Right examples, the join criterion was simply Left.Num = Right.Num. To turn this into a windowed join, we would expand the join criteria to include window equality, as well: Left.Num = Right.Num AND Left.Window = Right.Window. Knowing that, we can already infer from the preceding windowed tables how our join is going to change (highlighted for clarity): because the L2 and R2 rows do not fall within the same five-minute fixed window, they will not be joined together in the windowed variant of our join.

And indeed, if we compare the unwindowed and windowed variants side-by-side as tables, we can see this clearly (with the corresponding L2 and R2 rows highlighted on each side of the join):

                                 12:10> SELECT TABLE 
                                          Left.Id as L,
                                          Right.Id as R,
                                          COALESCE(
                                            TUMBLE(Left.Time, INTERVAL '5' MINUTE),
                                            TUMBLE(Right.Time, INTERVAL '5' MINUTE)
12:10> SELECT TABLE                       ) AS Window
         Left.Id as L,                  FROM Left
         Right.Id as R,                   FULL OUTER JOIN Right 
       FROM Left                          ON L.Num = R.Num AND 
         FULL OUTER JOIN Right              TUMBLE(Left.Time, INTERVAL '5' MINUTE) =
         ON L.Num = R.Num;                  TUMBLE(Right.Time, INTERVAL '5' MINUTE);
---------------                  --------------------------------
| L    | R    |                  | L    | R    | Window         |
---------------                  --------------------------------
| L1   | null |                  | L1   | null | [12:00, 12:05) |
| L2   | R2   |                  | null | R2   | [12:00, 12:05) |
| L3   | R3   |                  | L3   | R3   | [12:00, 12:05) |
| null | R4   |                  | L2   | null | [12:05, 12:10) |
---------------                  | null | R4   | [12:05, 12:10) |
                                 --------------------------------
The difference is also readily apparent when comparing the unwindowed and windowed joins as streams. As I’ve highlighted in the example that follows, they differ primarily in their final rows. The unwindowed side completes the join for Num = 2, yielding a retraction for the unjoined R2 row in addition to a new row for the completed L2, R2 join. The windowed side, on the other hand, simply yields an unjoined L2 row because L2 and R2 fall within different five-minute windows:

                                 12:10> SELECT STREAM 
                                          Left.Id as L,
                                          Right.Id as R,
                                          Sys.EmitTime as Time,
                                          COALESCE(
                                            TUMBLE(Left.Time, INTERVAL '5' MINUTE),
12:10> SELECT STREAM                        TUMBLE(Right.Time, INTERVAL '5' MINUTE)
         Left.Id as L,                    ) AS Window,
         Right.Id as R,                 Sys.Undo as Undo
         Sys.EmitTime as Time,          FROM Left
         Sys.Undo as Undo                 FULL OUTER JOIN Right
       FROM Left                          ON L.Num = R.Num AND
         FULL OUTER JOIN Right              TUMBLE(Left.Time, INTERVAL '5' MINUTE) =
         ON L.Num = R.Num;                  TUMBLE(Right.Time, INTERVAL '5' MINUTE);
------------------------------   -----------------------------------------------
| L    | R    | Time  | Undo |   | L    | R    | Time  | Window         | Undo |
------------------------------   -----------------------------------------------
| null | R2   | 12:01 |      |   | null | R2   | 12:01 | [12:00, 12:05) |      |
| L1   | null | 12:02 |      |   | L1   | null | 12:02 | [12:00, 12:05) |      |
| L3   | null | 12:03 |      |   | L3   | null | 12:03 | [12:00, 12:05) |      |
| L3   | null | 12:04 | undo |   | L3   | null | 12:04 | [12:00, 12:05) | undo |
| L3   | R3   | 12:04 |      |   | L3   | R3   | 12:04 | [12:00, 12:05) |      |
| null | R4   | 12:05 |      |   | null | R4   | 12:05 | [12:05, 12:10) |      |
| null | R2   | 12:06 | undo |   | L2   | null | 12:06 | [12:05, 12:10) |      |
| L2   | R2   | 12:06 |      |   ............... [12:00, 12:10] ................
....... [12:00, 12:10] .......
And with that, we now understand the effects of windowing on a FULL OUTER join. By applying the rules we learned in the first half of the chapter, it’s then easy to derive the windowed variants of LEFT OUTER, RIGHT OUTER, INNER, ANTI, and SEMI joins, as well. I will leave most of these derivations as an exercise for you to complete, but to give a single example, LEFT OUTER join, as we learned, is just the FULL OUTER join with null columns on the left side of the join removed (again, with L2 and R2 rows highlighted to compare the differences):

                                 12:10> SELECT TABLE 
                                          Left.Id as L,
                                          Right.Id as R,
                                          COALESCE(
                                            TUMBLE(Left.Time, INTERVAL '5' MINUTE),
                                            TUMBLE(Right.Time, INTERVAL '5' MINUTE)
12:10> SELECT TABLE                       ) AS Window
         Left.Id as L,                  FROM Left
         Right.Id as R,                   LEFT OUTER JOIN Right 
       FROM Left                          ON L.Num = R.Num AND 
         LEFT OUTER JOIN Right              TUMBLE(Left.Time, INTERVAL '5' MINUTE) =
         ON L.Num = R.Num;                  TUMBLE(Right.Time, INTERVAL '5' MINUTE);
---------------                  --------------------------------
| L    | R    |                  | L    | R    | Window         |
---------------                  --------------------------------
| L1   | null |                  | L1   | null | [12:00, 12:05) |
| L2   | R2   |                  | L2   | null | [12:05, 12:10) |
| L3   | R3   |                  | L3   | R3   | [12:00, 12:05) |
---------------                  --------------------------------
By scoping the region of time for the join into fixed five-minute intervals, we chopped our datasets into two distinct windows of time: [12:00, 12:05) and [12:05, 12:10). The exact same join logic we observed earlier was then applied within those regions, yielding a slightly different outcome for the case in which the L2 and R2 rows fell into separate regions. And at a basic level, that’s really all there is to windowed joins.

Temporal Validity
Having looked at the basics of windowed joins, we now spend the rest of the chapter looking at a somewhat more advanced approach: temporal validity windowing.
Temporal validity windows
Temporal validity windows apply in situations in which the rows in a relation effectively slice time into regions wherein a given value is valid. More concretely, imagine a financial system for performing currency conversions.4 Such a system might contain a time-varying relation that captured the current conversion rates for various types of currency. For example, there might be a relation for converting from different currencies to Yen, like this:

12:10> SELECT TABLE * FROM YenRates;
--------------------------------------
| Curr | Rate | EventTime | ProcTime |
--------------------------------------
| USD  | 102  | 12:00:00  | 12:04:13 |
| Euro | 114  | 12:00:30  | 12:06:23 |
| Yen  | 1    | 12:01:00  | 12:05:18 |
| Euro | 116  | 12:03:00  | 12:09:07 |
| Euro | 119  | 12:06:00  | 12:07:33 |
--------------------------------------
To highlight what I mean by saying that temporal validity windows “effectively slice time into regions wherein a given value is valid,” consider only the Euro-to-Yen conversion rates in that relation:

12:10> SELECT TABLE * FROM YenRates WHERE Curr = "Euro";
--------------------------------------
| Curr | Rate | EventTime | ProcTime |
--------------------------------------
| Euro | 114  | 12:00:30  | 12:06:23 |
| Euro | 116  | 12:03:00  | 12:09:07 |
| Euro | 119  | 12:06:00  | 12:07:33 |
--------------------------------------
From a database engineering perspective, we understand that these values don’t mean that the rate for converting Euros to Yen is 114 ¥/€ at precisely 12:00, 116 ¥/€ at 12:03, 119 ¥/€ at 12:06, and undefined at all other times. Instead, we know that the intent of this table is to capture the fact that the conversion rate for Euros to Yen is undefined until 12:00, 114 ¥/€ from 12:00 to 12:03, 116 ¥/€ from 12:03 to 12:06, and 119 ¥/€ from then on. Or drawn out in a timeline:

        Undefined              114 ¥/€                116 ¥/€              119 ¥/€
|----[-inf, 12:00)----|----[12:00, 12:03)----|----[12:03, 12:06)----|----[12:06, now)----→
Now, if we knew all of the rates ahead of time, we could capture these regions explicitly in the row data themselves. But if we instead need to build up these regions incrementally, based only upon the start times at which a given rate becomes valid, we have a problem: the region for a given row will change over time depending on the rows that come after it. This is a problem even if the data arrive in order (because every time a new rate arrives, the previous rate changes from being valid forever to being valid until the arrival time of the new rate), but is further compounded if they can arrive out of order. For example, using the processing-time ordering in the preceding YenRates table, the sequence of timelines our table would effectively represent over time would be as follows:

Range of processing time | Event-time validity timeline during that range of processing-time
=========================|==============================================================================
                         |
                         |      Undefined
        [-inf, 12:06:23) | |--[-inf, +inf)---------------------------------------------------------→
                         |
                         |      Undefined          114 ¥/€
    [12:06:23, 12:07:33) | |--[-inf, 12:00)--|--[12:00, +inf)--------------------------------------→
                         |
                         |      Undefined          114 ¥/€                              119 ¥/€
    [12:07:33, 12:09:07) | |--[-inf, 12:00)--|--[12:00, 12:06)---------------------|--[12:06, +inf)→
                         |
                         |      Undefined          114 ¥/€            116 ¥/€           119 ¥/€
         [12:09:07, now) | |--[-inf, 12:00)--|--[12:00, 12:03)--|--[12:03, 12:06)--|--[12:06, +inf)→
Or, if we wanted to render this as a time-varying relation (with changes between each snapshot relation highlighted in yellow):

12:10> SELECT TVR * FROM YenRatesWithRegion ORDER BY EventTime;
---------------------------------------------------------------------------------------------
| [-inf, 12:06:23)                            | [12:06:23, 12:07:33)                        |
| ------------------------------------------- | ------------------------------------------- |
|                                             | Curr                                        |
| ------------------------------------------- | ------------------------------------------- |
| ------------------------------------------- |                                             |
|                                             | ------------------------------------------- |
---------------------------------------------------------------------------------------------
| [12:07:33, 12:09:07)                        | [12:09:07, +inf)                            |
| ------------------------------------------- | ------------------------------------------- |
|                                             | Curr                                        |
| ------------------------------------------- | ------------------------------------------- |
|                                             | Euro                                        |
|                                             | Euro                                        |
| ------------------------------------------- |                                             |
|                                             | ------------------------------------------- |
---------------------------------------------------------------------------------------------
What’s important to note here is that half of the changes involve updates to multiple rows. That maybe doesn’t sound so bad, until you recall that the difference between each of these snapshots is the arrival of exactly one new row. In other words, the arrival of a single new input row results in transactional modifications to multiple output rows. That sounds less good. On the other hand, it also sounds a lot like the multirow transactions involved in building up session windows. And indeed, this is yet another example of windowing providing benefits beyond simple partitioning of time: it also affords the ability to do so in ways that involve complex, multirow transactions.

To see this in action, let’s look at an animation. If this were a Beam pipeline, it would probably look something like the following:

PCollection<Currency, Decimal> yenRates = ...;
PCollection<Decimal> validYenRates = yenRates
    .apply(Window.into(new ValidityWindows())
    .apply(GroupByKey.<Currency, Decimal>create());
Rendered in a streams/tables animation, that pipeline would look like that shown in Figure 9-1.

Figure 9-1. Temporal validity windowing over time
This animation highlights a critical aspect of temporal validity: shrinking windows. Validity windows must be able to shrink over time, thereby diminishing the reach of their validity and splitting any data contained therein across the two new windows. See the code snippets on GitHub for an example partial implementation.5

In SQL terms, the creation of these validity windows would look something like the following (making using of a hypothetical VALIDITY_WINDOW construct), viewed as a table:

12:10> SELECT TABLE 
         Curr,
         MAX(Rate) as Rate,
         VALIDITY_WINDOW(EventTime) as Window
       FROM YenRates 
       GROUP BY
         Curr,
         VALIDITY_WINDOW(EventTime)
       HAVING Curr = "Euro";
--------------------------------
| Curr | Rate | Window         |
--------------------------------
| Euro | 114  | [12:00, 12:03) |
| Euro | 116  | [12:03, 12:06) |
| Euro | 119  | [12:06, +inf)  |
--------------------------------
VALIDITY WINDOWS IN STANDARD SQL
Note that it’s possible to describe validity windows in standard SQL using a three-way self-join:

SELECT
  r1.Curr,
  MAX(r1.Rate) AS Rate,
  r1.EventTime AS WindowStart,
  r2.EventTime AS WIndowEnd
FROM YenRates r1
LEFT JOIN YenRates r2
  ON r1.Curr = r2.Curr
     AND r1.EventTime < r2.EventTime
LEFT JOIN YenRates r3
  ON r1.Curr = r3.Curr
     AND r1.EventTime < r3.EventTime 
     AND r3.EventTime < r2.EventTime
WHERE r3.EventTime IS NULL
GROUP BY r1.Curr, WindowStart, WindowEnd
HAVING r1.Curr = 'Euro';
Thanks to Martin Kleppmann for pointing this out.

Or, perhaps more interestingly, viewed as a stream:

12:00> SELECT STREAM
         Curr,
         MAX(Rate) as Rate,
         VALIDITY_WINDOW(EventTime) as Window,
         Sys.EmitTime as Time,
         Sys.Undo as Undo,
       FROM YenRates
       GROUP BY
         Curr,
         VALIDITY_WINDOW(EventTime) 
       HAVING Curr = "Euro";
--------------------------------------------------
| Curr | Rate | Window         | Time     | Undo |
--------------------------------------------------
| Euro | 114  | [12:00, +inf)  | 12:06:23 |      |
| Euro | 114  | [12:00, +inf)  | 12:07:33 | undo |
| Euro | 114  | [12:00, 12:06) | 12:07:33 |      | 
| Euro | 119  | [12:06, +inf)  | 12:07:33 |      |
| Euro | 114  | [12:00, 12:06) | 12:09:07 | undo | 
| Euro | 114  | [12:00, 12:03) | 12:09:07 |      |
| Euro | 116  | [12:03, 12:06) | 12:09:07 |      |
................. [12:00, 12:10] .................
Great, we have an understanding of how to use point-in-time values to effectively slice up time into ranges within which those values are valid. But the real power of these temporal validity windows is when they are applied in the context of joining them with other data. That’s where temporal validity joins come in.

Temporal validity joins
To explore the semantics of temporal validity joins, suppose that our financial application contains another time-varying relation, one that tracks currency-conversion orders from various currencies to Yen:

12:10> SELECT TABLE * FROM YenOrders;
----------------------------------------
| Curr | Amount | EventTime | ProcTime |
----------------------------------------
| Euro | 2      | 12:02:00  | 12:05:07 |
| USD  | 1      | 12:03:00  | 12:03:44 |
| Euro | 5      | 12:05:00  | 12:08:00 |
| Yen  | 50     | 12:07:00  | 12:10:11 |
| Euro | 3      | 12:08:00  | 12:09:33 |
| USD  | 5      | 12:10:00  | 12:10:59 |
----------------------------------------
And for simplicity, as before, let’s focus on the Euro conversions:

12:10> SELECT TABLE * FROM YenOrders WHERE Curr = "Euro";
----------------------------------------
| Curr | Amount | EventTime | ProcTime |
----------------------------------------
| Euro | 2      | 12:02:00  | 12:05:07 |
| Euro | 5      | 12:05:00  | 12:08:00 |
| Euro | 3      | 12:08:00  | 12:09:33 |
----------------------------------------
We’d like to robustly join these orders to the YenRates relation, treating the rows in YenRates as defining validity windows. As such, we’ll actually want to join to the validity-windowed version of the YenRates relation we constructed at the end of the last section:

12:10> SELECT TABLE
         Curr,
         MAX(Rate) as Rate,
         VALIDITY_WINDOW(EventTime) as Window
       FROM YenRates
       GROUP BY
         Curr,
         VALIDITY_WINDOW(EventTime)
       HAVING Curr = "Euro";
--------------------------------
| Curr | Rate | Window         |
--------------------------------
| Euro | 114  | [12:00, 12:03) |
| Euro | 116  | [12:03, 12:06) |
| Euro | 119  | [12:06, +inf)  |
--------------------------------
Fortunately, after we have our conversion rates placed into validity windows, a windowed join between those rates and the YenOrders relation gives us exactly what we want:

12:10> WITH ValidRates AS
         (SELECT
            Curr,
            MAX(Rate) as Rate,
            VALIDITY_WINDOW(EventTime) as Window
          FROM YenRates
          GROUP BY
            Curr,
            VALIDITY_WINDOW(EventTime))
       SELECT TABLE
         YenOrders.Amount as "E",
         ValidRates.Rate as "Y/E", 
         YenOrders.Amount * ValidRates.Rate as "Y",
         YenOrders.EventTime as Order, 
         ValidRates.Window as "Rate Window"
       FROM YenOrders FULL OUTER JOIN ValidRates 
         ON YenOrders.Curr = ValidRates.Curr
           AND WINDOW_START(ValidRates.Window) <= YenOrders.EventTime
           AND YenOrders.EventTime < WINDOW_END(ValidRates.Window)
       HAVING Curr = "Euro";
-------------------------------------------
| E | Y/E | Y   | Order  | Rate Window    |
-------------------------------------------
| 2 | 114 | 228 | 12:02  | [12:00, 12:03) |
| 5 | 116 | 580 | 12:05  | [12:03, 12:06) |
| 3 | 119 | 357 | 12:08  | [12:06, +inf)  |
-------------------------------------------
Thinking back to our original YenRates and YenOrders relations, this joined relation indeed looks correct: each of the three conversions ended up with the (eventually) appropriate rate for the given window of event time within which their corresponding order fell. So we have a decent sense that this join is doing what we want in terms of providing us the eventual correctness we want.

That said, this simple snapshot view of the relation, taken after all the values have arrived and the dust has settled, belies the complexity of this join. To really understand what’s going on here, we need to look at the full TVR. First, recall that the validity-windowed conversion rate relation was actually much more complex than the previous simple table snapshot view might lead you to believe. For reference, here’s the STREAM version of the validity windows relation, which better highlights the evolution of those conversion rates over time:

12:00> SELECT STREAM
         Curr,
         MAX(Rate) as Rate,
         VALIDITY(EventTime) as Window,
         Sys.EmitTime as Time,
         Sys.Undo as Undo,
       FROM YenRates
       GROUP BY
         Curr,
         VALIDITY(EventTime)
       HAVING Curr = "Euro";
--------------------------------------------------
| Curr | Rate | Window         | Time     | Undo |
--------------------------------------------------
| Euro | 114  | [12:00, +inf)  | 12:06:23 |      |
| Euro | 114  | [12:00, +inf)  | 12:07:33 | undo |
| Euro | 114  | [12:00, 12:06) | 12:07:33 |      | 
| Euro | 119  | [12:06, +inf)  | 12:07:33 |      |
| Euro | 114  | [12:00, 12:06) | 12:09:07 | undo | 
| Euro | 114  | [12:00, 12:03) | 12:09:07 |      |
| Euro | 116  | [12:03, 12:06) | 12:09:07 |      |
................. [12:00, 12:10] .................
As a result, if we look at the full TVR for our validity-windowed join, you can see that the corresponding evolution of this join over time is much more complicated, due to the out-of-order arrival of values on both sides of the join:

12:10> WITH ValidRates AS
         (SELECT
            Curr,
            MAX(Rate) as Rate,
            VALIDITY_WINDOW(EventTime) as Window
          FROM YenRates
          GROUP BY
            Curr,
            VALIDITY_WINDOW(EventTime))
       SELECT TVR
         YenOrders.Amount as "E",
         ValidRates.Rate as "Y/E", 
         YenOrders.Amount * ValidRates.Rate as "Y",
         YenOrders.EventTime as Order,
         ValidRates.Window as "Rate Window"
       FROM YenOrders FULL OUTER JOIN ValidRates 
         ON YenOrders.Curr = ValidRates.Curr
           AND WINDOW_START(ValidRates.Window) <= YenOrders.EventTime
           AND YenOrders.EventTime < WINDOW_END(ValidRates.Window)
       HAVING Curr = "Euro";
-------------------------------------------------------------------------------------------
| [-inf, 12:05:07)                           | [12:05:07, 12:06:23)                       |
| ------------------------------------------ | ------------------------------------------ |
|                                            | E                                          |
| ------------------------------------------ | ------------------------------------------ |
| ------------------------------------------ |                                            |
|                                            | ------------------------------------------ |
-------------------------------------------------------------------------------------------
| [12:06:23, 12:07:33)                       | [12:07:33, 12:08:00)                       |
| ------------------------------------------ | ------------------------------------------ |
|                                            | E                                          |
| ------------------------------------------ | ------------------------------------------ |
|                                            | 2                                          |
| ------------------------------------------ |                                            |
|                                            | ------------------------------------------ |
-------------------------------------------------------------------------------------------
| [12:08:00, 12:09:07)                       | [12:09:07, 12:09:33)                       |
| ------------------------------------------ | ------------------------------------------ |
|                                            | E                                          |
| ------------------------------------------ | ------------------------------------------ |
|                                            | 2                                          |
|                                            | 5                                          |
|                                            |                                            |
| ------------------------------------------ | ------------------------------------------ |
-------------------------------------------------------------------------------------------
| [12:09:33, now)                            |
| ------------------------------------------ |
|                                            |
| ------------------------------------------ |
|                                            |
|                                            |
|                                            |
| ------------------------------------------ |
----------------------------------------------
In particular, the result for the 5 € order is originally quoted at 570 ¥ because that order (which happened at 12:05) originally falls into the validity window for the 114 ¥/€ rate. But when the 116 ¥/€ rate for event time 12:03 arrives out of order, the result for the 5 € order must be updated from 570 ¥ to 580 ¥. This is also evident if you observe the results of the join as a stream (here I’ve highlighted the incorrect 570 ¥ in red, and the retraction for 570 ¥ and subsequent corrected value of 580 ¥ in blue):

12:00> WITH ValidRates AS
         (SELECT
            Curr,
            MAX(Rate) as Rate,
            VALIDITY_WINDOW(EventTime) as Window
          FROM YenRates
          GROUP BY
            Curr,
            VALIDITY_WINDOW(EventTime))
       SELECT STREAM
         YenOrders.Amount as "E",
         ValidRates.Rate as "Y/E", 
         YenOrders.Amount * ValidRates.Rate as "Y",
         YenOrders.EventTime as Order,
         ValidRates.Window as "Rate Window",
         Sys.EmitTime as Time,
         Sys.Undo as Undo
       FROM YenOrders FULL OUTER JOIN ValidRates 
         ON YenOrders.Curr = ValidRates.Curr
           AND WINDOW_START(ValidRates.Window) <= YenOrders.EventTime
           AND YenOrders.EventTime < WINDOW_END(ValidRates.Window)
       HAVING Curr = “Euro”;
------------------------------------------------------------
| E | Y/E | Y   | Order | Rate Window    | Time     | Undo | 
------------------------------------------------------------
| 2 |     |     | 12:02 |                | 12:05:07 |      |
| 2 |     |     | 12:02 |                | 12:06:23 | undo |
| 2 | 114 | 228 | 12:02 | [12:00, +inf)  | 12:06:23 |      |
| 2 | 114 | 228 | 12:02 | [12:00, +inf)  | 12:07:33 | undo |
| 2 | 114 | 228 | 12:02 | [12:00, 12:06) | 12:07:33 |      |
|   | 119 |     |       | [12:06, +inf)  | 12:07:33 |      |
| 5 | 114 | 570 | 12:05 | [12:00, 12:06) | 12:08:00 |      |
| 2 | 114 | 228 | 12:02 | [12:00, 12:06) | 12:09:07 | undo |
| 5 | 114 | 570 | 12:05 | [12:00, 12:06) | 12:09:07 | undo |
| 2 | 114 | 228 | 12:02 | [12:00, 12:03) | 12:09:07 |      |
| 5 | 116 | 580 | 12:05 | [12:03, 12:06) | 12:09:07 |      |
|   | 119 |     |       | [12:06, +inf)  | 12:09:33 | undo |
| 3 | 119 | 357 | 12:08 | [12:06, +inf)  | 12:09:33 |      |
...................... [12:00, 12:10] ......................
It’s worth calling out that this is a fairly messy stream due to the use of a FULL OUTER join. In reality, when consuming conversion orders as a stream, you probably don’t care about unjoined rows; switching to an INNER join helps eliminate those rows. You probably also don’t care about cases for which the rate window changes, but the actual conversion value isn’t affected. By removing the rate window from the stream, we can further decrease its chattiness:

12:00> WITH ValidRates AS
         (SELECT
            Curr,
            MAX(Rate) as Rate,
            VALIDITY_WINDOW(EventTime) as Window
          FROM YenRates
          GROUP BY
            Curr,
            VALIDITY_WINDOW(EventTime))
       SELECT STREAM
         YenOrders.Amount as "E",
         ValidRates.Rate as "Y/E", 
         YenOrders.Amount * ValidRates.Rate as "Y",
         YenOrders.EventTime as Order,
         ValidRates.Window as "Rate Window",
         Sys.EmitTime as Time,
         Sys.Undo as Undo
       FROM YenOrders INNER JOIN ValidRates 
         ON YenOrders.Curr = ValidRates.Curr
           AND WINDOW_START(ValidRates.Window) <= YenOrders.EventTime
           AND YenOrders.EventTime < WINDOW_END(ValidRates.Window)
       HAVING Curr = "Euro";
-------------------------------------------
| E | Y/E | Y   | Order | Time     | Undo |
-------------------------------------------
| 2 | 114 | 228 | 12:02 | 12:06:23 |      |
| 5 | 114 | 570 | 12:05 | 12:08:00 |      |
| 5 | 114 | 570 | 12:05 | 12:09:07 | undo |
| 5 | 116 | 580 | 12:05 | 12:09:07 |      |
| 3 | 119 | 357 | 12:08 | 12:09:33 |      |
............. [12:00, 12:10] ..............
Much nicer. We can now see that this query very succinctly does what we originally set out to do: join two TVRs for currency conversion rates and orders in a robust way that is tolerant of data arriving out of order. Figure 9-2 visualizes this query as an animated diagram. In it, you can also very clearly see the way the overall structure of things change as they evolve over time.

Figure 9-2. Temporal validity join, converting Euros to Yen with per-record triggering
Watermarks and temporal validity joins
With this example, we’ve highlighted the first benefit of windowed joins called out at the beginning of this section: windowing a join allows you to partition that join within time for some practical business need. In this case, the business need was slicing time into regions of validity for our currency conversion rates.

Before we call it a day, however, it turns out that this example also provides an opportunity to highlight the second point I called out: the fact that windowing a join can provide a meaningful reference point for watermarks. To see how that’s useful, imagine changing the previous query to replace the implicit default per-record trigger with an explicit watermark trigger that would fire only once when the watermark passed the end of the validity window in the join (assuming that we have a watermark available for both of our input TVRs that accurately tracks the completeness of those relations in event time as well as an execution engine that knows how to take those watermarks into consideration). Now, instead of our stream containing multiple outputs and retractions for rates arriving out of order, we could instead end up with a stream containing a single, correct converted result per order, which is clearly even more ideal than before:

12:00> WITH ValidRates AS
         (SELECT
            Curr,
            MAX(Rate) as Rate,
            VALIDITY_WINDOW(EventTime) as Window
          FROM YenRates
          GROUP BY
            Curr,
            VALIDITY_WINDOW(EventTime))
       SELECT STREAM
         YenOrders.Amount as "E",
         ValidRates.Rate as "Y/E", 
         YenOrders.Amount * ValidRates.Rate as "Y",
         YenOrders.EventTime as Order,
         Sys.EmitTime as Time,
         Sys.Undo as Undo
       FROM YenOrders INNER JOIN ValidRates 
         ON YenOrders.Curr = ValidRates.Curr
           AND WINDOW_START(ValidRates.Window) <= YenOrders.EventTime
           AND YenOrders.EventTime < WINDOW_END(ValidRates.Window)
       HAVING Curr = "Euro"
       EMIT WHEN WATERMARK PAST WINDOW_END(ValidRates.Window);
-------------------------------------------
| E | Y/E | Y   | Order | Time     | Undo |
-------------------------------------------
| 2 | 114 | 228 | 12:02 | 12:08:52 |      |
| 5 | 116 | 580 | 12:05 | 12:10:04 |      |
| 3 | 119 | 357 | 12:08 | 12:10:13 |      |
............. [12:00, 12:11] ..............
Or, rendered as an animation, which clearly shows how joined results are not emitted into the output stream until the watermark moves beyond them, as demonstrated in Figure 9-3.

Figure 9-3. Temporal validity join, converting Euros to Yen with watermark triggering
Either way, it’s impressive to see how this query encapsulates such a complex set of interactions into a clean and concise rendering of the desired results.

Summary
In this chapter, we analyzed the world of joins (using the join vocabulary of SQL) within the context of stream processing. We began with unwindowed joins and saw how, conceptually, all joins are streaming joins as the core. We saw how the foundation for essentially all of the other join variations is the FULL OUTER join, and discussed the specific alterations that occur as part of LEFT OUTER, RIGHT OUTER, INNER, ANTI, SEMI, and even CROSS joins. In addition, we saw how all of those different join patterns interact in a world of TVRs and streams.

We next moved on to windowed joins, and learned that windowing a join is typically motivated by one or both of the following benefits:

The ability to partition the join within time for some business need

The ability to tie results from the join to the progress of a watermark

And, finally, we explored in depth one of the more interesting and useful types of windows with respect to joining: temporal validity windows. We saw how temporal validity windows very naturally carve time into regions of validity for given values, based only on the specific points in time where those values change. We learned that joins within validity windows require a windowing framework that supports windows that can split over time, which is something no existing streaming system today supports natively. And we saw how concisely validity windows allowed us to solve the problem of joining TVRs for currency conversion rates and orders together in a robust, natural way.

Joins are often one of the more intimidating aspects of data processing, streaming or otherwise. However, by understanding the theoretical foundation of joins and how straightforwardly we can derive all the different types of joins from that basic foundation, joins become a much less frightening beast, even with the additional dimension of time that streaming adds to the mix.

1 From a conceptual perspective, at least. There are many different ways to implement each of these types of joins, some of which are likely much more efficient than performing an actual FULL OUTER join and then filtering down its results, especially when the rest of the query and the distribution of the data are taken into consideration.

2 Again, ignoring what happens when there are duplicate join keys; more on this when we get to SEMI joins.

3 From a conceptual perspective, at least. There are, of course, many different ways to implement each of these types of joins, some of which might be much more efficient than performing an actual FULL OUTER join and then filtering down its results, depending on the rest of the query and the distribution of the data.

4 Note that the example data and the temporal join use case motivating it are lifted almost wholesale from Julian Hyde’s excellent “Streams, joins, and temporal tables” document.

5 It’s a partial implementation because it only works if the windows exist in isolation, as in Figure 9-1. As soon as you mix the windows with other data, such as the joining examples below, you would need some mechanism for splitting the data from the shrunken window into two separate windows, which Beam does not currently provide.