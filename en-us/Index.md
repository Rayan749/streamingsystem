# Index
A
accumulating and retracting mode, How: Accumulation, How: Accumulation, How: accumulation
early/on-time/late triggers, How: Accumulation
accumulating mode (accumulation), How: Accumulation, How: Accumulation, How: accumulation
accumulation, Roadmap, How: Accumulation-How: Accumulation
accumulating and retracting mode, How: Accumulation
accumulating mode, How: Accumulation
accumulation mode in processing-time window via ingress time, Processing-Time Windowing via Ingress Time
discarding mode, How: Accumulation
in processsing-time windowing via triggers, Processing-Time Windowing via Triggers
in streaming SQL, How: accumulation
discarding mode, Discarding mode, or lack thereof
retractions, Retractions in a SQL world-Retractions in a SQL world
in streams and tables model, How: Accumulation
side-by-side comparison of modes, How: Accumulation
accumulators, Incremental Combining
accuracy
in lambda architecture processing, Why Exactly Once Matters
vs. completeness in exactly-once processing, Accuracy Versus Completeness-Problem Definition
aggregations
grouping and summation via incremental combination, Incremental Combining
incrementalization of, Incremental Combining
parallelization of, Incremental Combining
properties of, Incremental Combining
aligned delays (processing time in triggers), When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
allowed lateness, When: Allowed Lateness (i.e., Garbage Collection)-When: Allowed Lateness (i.e., Garbage Collection)
ANTI joins, ANTI
Apache Beam, Beam-Summary
blending of batch and streaming, When: Triggers
code snippets for, The What, Where, When, and How of Data Processing
Java SDK pseudo-code, What: Transformations
CombineFn API, Incremental Combining
components, Beam
conversion attribution with, Conversion Attribution with Apache Beam-Conversion Attribution with Apache Beam
innovation in, Beam
portability layer, Beam
streaming SQL in, What Is Streaming SQL?
Apache Calcite, What Is Streaming SQL?
Apache Flink, On the Greatly Exaggerated Limitations of Streaming, What Is Streaming SQL?, Flink-Flink, Summary
adoption of Dataflow/Beam programming model, Flink
consistency mechanism, Flink
end-to-end exactly once, Problem Definition
exactly-once processing in, Apache Flink
factors in its rapid rise to stream procesing juggernaut, Flink
highly efficient snapshotting implementation, Flink
impressive performance of, Flink
savepoints feature, Flink
snapshotting paper, On the Greatly Exaggerated Limitations of Streaming
watermarks in, case study, Case Study: Watermarks in Apache Flink
Apache Kafka, Exactly Once in Sources, Kafka-Kafka, Summary
application of durability and replayability to stream data, Kafka
capabilities of, Kafka
Kafka's Streams API, Kafka
popularization of stream and table theory, Kafka
Apache Spark, Spark-Spark, Summary
current developments in, Spark
end-to-end exactly once, Problem Definition
Spark Streaming, Apache Spark Streaming, Spark
ingress times as event times, When/Where: Processing-Time Windows
manually building up sessions in, Where: Session Windows
snapshotting paper, On the Greatly Exaggerated Limitations of Streaming
Apache Storm, Storm-Storm, Summary
bringing low-latency data processing to the masses, Storm
history of its creation, Storm
append-only logs, Stream-and-Table Basics Or: a Special Theory of Stream and Table Relativity
approximation algorithms, Approximation algorithms
AS OF SYSTEM TIME construct (SQL), Streams and Tables
assignment (window), Where: Custom Windowing
in fixed windows, Variations on Fixed Windows
in session windows, Variations on Session Windows
in streams and tables model, Where: Windowing
associativity, Incremental Combining
at least once guarantee, Ensuring Exactly Once in Shuffle
B
base subscription (Google Cloud Pub/Sub case study), Case Study: Source Watermarks for Google Cloud Pub/Sub
batch processing
blending with streaming, When: Triggers
commonalities between batch and streaming, Cloud Dataflow
event-time and processing-time view of, What: Transformations
persistent state in, The Inevitability of Failure
streams and tables view, Temporal Operators
streams and tables view of windowed summation on batch engine, Where: Windowing
unified batch plus streaming programming model, Cloud Dataflow, Beam
vs. streams and tables, Batch Processing Versus Streams and Tables-Reconciling with Batch Processing
reconciling the two, Reconciling with Batch Processing
batch systems, On the Greatly Exaggerated Limitations of Streaming
bounded data processing with, Bounded Data
processing of unbounded data, Unbounded Data: Batch
processing state and output in example mobile game with user scores, What: Transformations
streaming systems providing superset of, On the Greatly Exaggerated Limitations of Streaming
Beam Model, The What, Where, When, and How of Data Processing, Streams and Tables
(see also Apache Beam)
correct implementation to produce accurate results, Exactly-Once and Side Effects
holistic view of streams and tables in, A Holistic View of Streams and Tables in the Beam Model-A Holistic View of Streams and Tables in the Beam Model
relationship to streams and tables model, A Holistic View of Streams and Tables in the Beam Model
windowing in, Summary
Bloom filters, Bloom Filters
bounded data, Terminology: What Is Streaming?
processing, Bounded Data
bounded datasets
recomputation on failure, The Inevitability of Failure
bounded sessions, Bounded sessions
buffering
in event-time windows, Windowing by event time
C
Calcite (see Apache Calcite)
CAP theorem, Storm
cardinality
in unwindowed joins, Unwindowed Joins
SEMI join, SEMI
of datasets, Terminology: What Is Streaming?, Streams and Tables
reducing for a stream, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
Chandy Lamport distributed snapshots, Summary, Flink
checkpointing
in bounded datasets on batch processing pipelines, The Inevitability of Failure
in Flink, Flink
in processing of unbounded datasets, The Inevitability of Failure
partial progress within a pipeline, Correctness and Efficiency
persistent state over time, On the Greatly Exaggerated Limitations of Streaming
use to make nondeterministic processing deterministic in Dataflow, Addressing Determinism
closure property (relational algebra), Relational Algebra, Summary
remaining intact when applied to time-varying relations, Time-Varying Relations
Cloud Dataflow, The What, Where, When, and How of Data Processing, Cloud Dataflow-Cloud Dataflow, Summary
balancing correctness, latency, and cost, Cloud Dataflow
Dataflow Model paper, Cloud Dataflow
exactly-once processing in, Exactly-Once and Side Effects
serverless aspect of, Cloud Dataflow
unified batch and streaming programming model
key aspects of, Cloud Dataflow
unified batch plus streaming programming model, Cloud Dataflow
watermarks in, case study, Case Study: Watermarks in Google Cloud Dataflow-Case Study: Watermarks in Google Cloud Dataflow
Cloud Pub/Sub, Heuristic Watermark Creation
as example source, Example Source: Cloud Pub/Sub
as nondeterministic source, Exactly Once in Sources
watermarks for, case study, Case Study: Source Watermarks for Google Cloud Pub/Sub-Case Study: Source Watermarks for Google Cloud Pub/Sub
combination, incremental (see incremental combination)
CombineFn class (Beam), Incremental Combining
combiner lifting optimization, Graph Optimization, Flume
commutativity, Incremental Combining
completeness
accuracy vs., in exactly-once processing, Accuracy Versus Completeness
concept provided by watermarks, Definition
drawback of event-time windows in, Windowing by event time
watermarks for reasoning about input completeness, Cloud Dataflow
watermarks giving notion of, When: Watermarks
completeness triggers, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
watermarks, When: Watermarks
complexity in lambda architecture processing, Why Exactly Once Matters
consistency
consistency mechanism in Flink, Flink
strong consistency for exactly-once processing, On the Greatly Exaggerated Limitations of Streaming
constitution of a dataset, Terminology: What Is Streaming?, Streams and Tables
conversion attribution, Case Study: Conversion Attribution-Case Study: Conversion Attribution
with Apache Beam, Conversion Attribution with Apache Beam-Conversion Attribution with Apache Beam
correctness
Apache Storm and, Storm
balancing with latency and cost in Cloud Dataflow, Cloud Dataflow
in batch and streaming systems, On the Greatly Exaggerated Limitations of Streaming
MillWheel and, MillWheel
persistent stte as basis for, Correctness and Efficiency
supposed limitations of streaming systems, On the Greatly Exaggerated Limitations of Streaming
custom windowing, Where: Custom Windowing-One Size Does Not Fit All
benefits of, One Size Does Not Fit All
variations on fixed windows, Variations on Fixed Windows-Per-element/key fixed windows
per-element/key fixed windows, Per-element/key fixed windows
unaligned fixed windows, Unaligned fixed windows
variations on session windows, Variations on Session Windows-Bounded sessions
bounded sessions, Bounded sessions
D
data processing
difficulty of, MapReduce
what, where, when, and how of, The What, Where, When, and How of Data Processing-Summary, Cloud Dataflow
what, transformations, What: Transformations-What: Transformations
when and how in streaming systems, Going Streaming: When and How-How: Accumulation
where, windowing, Where: Windowing-Where: Windowing
data processing patterns, Data Processing Patterns-Windowing by event time
bounded data, Bounded Data
unbounded data, batch processing of, Unbounded Data: Batch
unbounded data, streaming, Unbounded Data: Streaming-Windowing by event time
data processing, large scale (see large-scale data processing, evolution of)
data types, flexibility in, Generalized State, Conversion Attribution with Apache Beam
data-driven triggers, Data-driven triggers
data-driven windows, sessions as example, Where: Session Windows
database systems, Stream-and-Table Basics Or: a Special Theory of Stream and Table Relativity
Dataflow (see Google Cloud Dataflow)
Dataflow Model, The What, Where, When, and How of Data Processing
Dataflow Model paper, Window merging, Cloud Dataflow
datasets
cardinality of, Terminology: What Is Streaming?
constitution of, Terminology: What Is Streaming?
determinism
addressing in exactly-once processing, Addressing Determinism
in sources, Exactly Once in Sources
nondeterministic components in side effects, Exactly Once in Sinks
discarding mode (accumulation), How: Accumulation, How: Accumulation
early/on-time/late triggers, How: Accumulation
in streaming SQL, Discarding mode, or lack thereof
domain specific languages (DSLs), Beam
DSLs (domain specific languages), Beam
duplicated work, minimizing with persistent state, Correctness and Efficiency
duplicates, detecting in shuffle, Ensuring Exactly Once in Shuffle
dynamic windows, How: Accumulation
(see also sessions)
dynamic work rebalancing (or liquid sharding), Flume
E
early/on-time/late triggers, When: Early/On-Time/Late Triggers FTW!-When: Early/On-Time/Late Triggers FTW!
accumulating and retracting mode version, How: Accumulation
discarding mode version, How: Accumulation
early panes, When: Early/On-Time/Late Triggers FTW!
in bounded session window, Bounded sessions
in streams and tables model, When: Triggers
late panes, When: Early/On-Time/Late Triggers FTW!
on-time pane, When: Early/On-Time/Late Triggers FTW!
watermark trigger with late firing in streaming SQL, Watermark triggers
with allowed lateness, When: Allowed Lateness (i.e., Garbage Collection)-When: Allowed Lateness (i.e., Garbage Collection)
with session windows and retractions, Where: Session Windows
efficiency
inefficiencies in MapReduce jobs, Flume
persistent data, minimizing work duplicated and data persistend, Correctness and Efficiency
end-to-end exactly once, Problem Definition, Kafka
event time
distribution of messages by, Definition
in SQL table UserScores (example), What: Transformations
in streaming SQL, Temporal Operators
skew and watermarks, When: Watermarks
view of batch processing, What: Transformations
vs. processing time, Event Time Versus Processing Time
watermarks, Roadmap, Processing-Time Watermarks
windowing based on, When/Where: Processing-Time Windows
windowing by, Windowing by event time
drawbacks of, Windowing by event time
event-time windowing
over two different processing-time orderings of same input, Event-Time Windowing
reasons for using in processing-time windowing, Processing-Time Windowing via Ingress Time
exactly-once processing, On the Greatly Exaggerated Limitations of Streaming, Exactly-Once and Side Effects-Summary, MillWheel
accuracy vs. completeness, Accuracy Versus Completeness, Problem Definition
problem definition, Problem Definition
side effects, Side Effects
determinism and, Addressing Determinism
end-to-end, Kafka
ensuring exactly once in shuffles, Ensuring Exactly Once in Shuffle
in Apache Flink, Apache Flink, Flink
in Apache Spark Streaming, Other Systems
in conversion attribution pipeline, Case Study: Conversion Attribution
performance, Performance-Garbage Collection
garbage collection, Garbage Collection
graph optimization, Graph Optimization
optimization using Bloom filters, Bloom Filters
use cases
example sink, files, Example Sink: Files
example sink, Google BigQuery, Example Sink: Google BigQuery
example source, Cloud Pub/Sub, Use Cases
why exactly once matters, Why Exactly Once Matters
F
failures, inevitability of, The Inevitability of Failure
fault-tolerance in large-scale data processing, MapReduce
files, using as sinks, Example Sink: Files
filtering, Filtering
filtering relation (WHERE clause), time-varying relation applied to, Time-Varying Relations
fixed windows, Windowing
unbounded data processing via in batch systems, Fixed windows
variations on, Variations on Fixed Windows-Per-element/key fixed windows
per-element/key fixed windows, Per-element/key fixed windows
unaligned fixed windows, Unaligned fixed windows
windowed joins in, Fixed Windows-Fixed Windows
flexibility
flexible triggering and accumulation modes, Cloud Dataflow
needs in streaming persistent state, Generalized State
in conversion attribution using Apache Beam, Conversion Attribution with Apache Beam
shortcomings of implicit approaches, Generalized State
Flume, Flume-Flume, Summary
combined batch and streaming approach in, Cloud Dataflow
combiner lifting optimization, Flume
dynamic work rebalancing (or liquid sharding), Flume
extension to support streaming semantics, Flume
FlumeJava paper, Flume
fusion optimizations, Flume
high-level pipelines in, Flume
migration away from MapReduce to Dax execution engine, Flume
MillWheel integration with, MillWheel
optimization of MapReduce jobs, Flume
FlumeJava, Flume
(see also Flume)
FULL OUTER joins, FULL OUTER, Unwindowed Joins, Temporal validity joins
fusion optimization on pipeline graph, Graph Optimization, Flume
G
garbage collection
bits of persistent state not needed, Correctness and Efficiency
in exactly-once processing, Garbage Collection
generalized state, Generalized State-Conversion Attribution with Apache Beam
case study, conversion attribution, Case Study: Conversion Attribution-Case Study: Conversion Attribution
flexibility in, Summary
in conversion attribution, Case Study: Conversion Attribution-Case Study: Conversion Attribution
in conversion attribution using Apache Beam, Conversion Attribution with Apache Beam-Conversion Attribution with Apache Beam
Google BigQuery, use as a sink, Example Sink: Google BigQuery
Google Cloud Dataflow (see Cloud Dataflow; Dataflow Model)
Google Cloud Pub/Sub (see Cloud Pub/Sub)
Google technologies in large-scale data processing, The Evolution of Large-Scale Data Processing
graph optimization in Dataflow, Graph Optimization
GROUP BY statement with HAVING clause (SQL), The SQL Model: A Table-Biased Approach
grouping operations
grouping via incremental combination, Incremental Combining
grouping/ungrouping in Beam Model, The Beam Model: A Stream-Biased Approach
grouping/ungrouping in SQL, The SQL Model: A Table-Biased Approach
in materialized views, Materialized views
grouping/ungrouping in streams and tables, When: Triggers
in Beam Model processing, A Holistic View of Streams and Tables in the Beam Model
joins as, All Your Joins Are Belong to Streaming
raw grouping of inputs, Raw Grouping
grouping relation, time-varying relation applied to, Time-Varying Relations
grouping transformations, What: Transformations
H
Hadoop, Hadoop, Summary
Spark as successor to, Spark
HAVING clause in GROUP BY statment (SQL), The SQL Model: A Table-Biased Approach
HDFS (Hadoop Distributed File System), Hadoop
Heron, Storm
heuristic watermarks, When: Watermarks, Source Watermark Creation
allowed lateness and, When: Allowed Lateness (i.e., Garbage Collection)
applying to same dataset with a perfect watermark, When: Watermarks
creation of, Heuristic Watermark Creation
from dynamic sets of time-ordered logs, Heuristic Watermark Creation
from Google Cloud Pub/Sub, Heuristic Watermark Creation
early/on-time/late triggers and, When: Early/On-Time/Late Triggers FTW!
high volumes of data, handling in conversion attribution pipeline, Case Study: Conversion Attribution
hopping windows (see sliding windows)
I
implicit state, Implicit State, Incremental Combining
implicit tables in SQL, The SQL Model: A Table-Biased Approach
in-band watermarks, Case Study: Watermarks in Apache Flink
inaccuracy problems in lambda architecture, Why Exactly Once Matters
inconsistency in lambda architecture processing, Why Exactly Once Matters
incremental combination, Incremental Combining-Incremental Combining, Summary
incrementalization of aggregations, Incremental Combining
ingress time
processing-time windowing via, Processing-Time Windowing via Ingress Time-Processing-Time Windowing via Ingress Time
use in achieving processing-time windowing, When/Where: Processing-Time Windows
ingress timestamping, watermark creation by, Perfect Watermark Creation
inner joins, Inner joins
INNER joins, INNER, Temporal validity joins
input completeness, Cloud Dataflow
input tables (SQL), The SQL Model: A Table-Biased Approach, Materialized views
input watermarks, Watermark Propagation
for Average Session Lengths stage, Understanding Watermark Propagation
J
Java pseudo-code in Apache Beam examples, What: Transformations
joins, Inner joins, Streaming Joins
(see also inner joins)
(see also streaming joins)
all joins as streaming joins, All Your Joins Are Belong to Streaming
K
kappa architecture, On the Greatly Exaggerated Limitations of Streaming
keys, values, windows and partitioning in Beam Model, A Holistic View of Streams and Tables in the Beam Model
L
lambda architecture, On the Greatly Exaggerated Limitations of Streaming, Summary, Storm
issues with, Why Exactly Once Matters
using Spark Streaming instead of, Spark
large-scale data processing, evolution of, The Evolution of Large-Scale Data Processing-Summary
Apache Beam, Beam-Beam
Apache Kafka, Kafka-Kafka
Apache Storm, Storm-Storm
Cloud Dataflow, Cloud Dataflow-Cloud Dataflow
Flume, Flume-Flume
Hadoop, Hadoop
MapReduce, MapReduce-MapReduce
MillWheel, MillWheel-MillWheel
timeline of systems discussed, The Evolution of Large-Scale Data Processing
late data
and heuristic watermarks, Heuristic Watermark Creation
and perfect watermarks, Perfect Watermark Creation
late panes, When: Early/On-Time/Late Triggers FTW!
latency
balancing with correctness and cost in Cloud Dataflow, Cloud Dataflow
improvements in Apache Storm, Storm
in lambda architecture processing, Why Exactly Once Matters
low-latency and eventually correct results with lambda architecture, Storm
system vs. data, distinguishing with processing-time watermarks, Processing-Time Watermarks
lateness (allowed), When: Allowed Lateness (i.e., Garbage Collection)-When: Allowed Lateness (i.e., Garbage Collection)
LEFT OUTER joins, LEFT OUTER
liquid sharding, Flume
logical abstraction of execution environment, Cloud Dataflow
logical vs. physical operations in Beam Model, A Holistic View of Streams and Tables in the Beam Model
and how they relate to streams and tables, A Holistic View of Streams and Tables in the Beam Model
logs
dymanic sets of time-ordered logs, watermark creation from, Heuristic Watermark Creation
static sets of time-ordered logs, creation of watermarks, Perfect Watermark Creation
M
MapReduce, The Evolution of Large-Scale Data Processing-MapReduce, Summary
Combiners, Incremental Combining
functionality of overall Map and Reduce phases, MapReduce
history of massive-scale sorting experiments at Google, MapReduce
MapReduce paper, MapReduce
shortcomings of, Flume
simplicity and scalability, MapReduce
stages answering what questions, What: Transformations
streams and tables analysis of, A Streams and Tables Analysis of MapReduce-Reconciling with Batch Processing
Map as streams/tables, Map as streams/tables
Reduce as streams/tables, Reduce as streams/tables
visualization of a job, MapReduce
materialized views, Stream-and-Table Basics Or: a Special Theory of Stream and Table Relativity, Materialized views-Materialized views, Unwindowed Joins
merging windows, Where: Custom Windowing, Where: Windowing
in parallelized aggregations, Incremental Combining
in session windows, Variations on Session Windows
in streams and tables model, Window merging
no merging in fixed windows, Variations on Fixed Windows
microbatch vs. true streaming debate, Spark
MillWheel, MillWheel-MillWheel, Summary
fault-tolerant stream processing at internet scale, MillWheel
Flume and, Flume
low-latency, strongly consistent processing of unbounded, out-of-order data, MillWheel
MillWheel paper, On the Greatly Exaggerated Limitations of Streaming
N
network remnants, Garbage Collection
nongrouping operations, A General Theory of Stream and Table Relativity, Summary
nongrouping transformations, What: Transformations
O
OLTP (Online Transaction Processing) tables
STREAM queries and, Streams and Tables
on-time pane, When: Early/On-Time/Late Triggers FTW!
open source ecosystem, Hadoop and, Hadoop
operators (relational algebra)
applied to time-varying relations, Time-Varying Relations
applying to valid relations, Relational Algebra
out-of-band watermark aggregation, Case Study: Watermarks in Apache Flink
out-of-order data
handling in conversion attribution pipeline, Case Study: Conversion Attribution-Case Study: Conversion Attribution
in temporal validity windows, Temporal validity windows
unbounded, processing in MillWheel, MillWheel
outer joins, When: Watermarks
output tables (SQL), The SQL Model: A Table-Biased Approach, Materialized views
output timestamps
watermark propagation and, Watermark Propagation and Output Timestamps-Watermark Propagation and Output Timestamps
watermarks and
with overlapping windows, The Tricky Case of Overlapping Windows
output watermarks, Watermark Propagation
components of, Watermark Propagation
for Mobile Sessions and Console Sessions stages, Understanding Watermark Propagation
P
parallelization of aggregations, Incremental Combining
per-element/key fixed windows, Per-element/key fixed windows
percentile watermarks, Percentile Watermarks-Percentile Watermarks
perfect watermarks, When: Watermarks, Source Watermark Creation
allowed lateness and, When: Allowed Lateness (i.e., Garbage Collection)
applying to same dataset with a heuristic watermark, When: Watermarks
creation of, Perfect Watermark Creation
by ingress timestamping, Perfect Watermark Creation
early/on-time/late triggers and, When: Early/On-Time/Late Triggers FTW!
performance
in exactly-once shuffle delivery, Performance-Garbage Collection
garbage collection, Garbage Collection
graph optimization, Graph Optimization
optimizing with Bloom filters, Bloom Filters
optimizing in conversion attribution pipeline, Case Study: Conversion Attribution
persistent state, The Practicalities of Persistent State-Summary, MillWheel
generalized state, Generalized State-Conversion Attribution with Apache Beam
in conversion attribution, Case Study: Conversion Attribution-Case Study: Conversion Attribution
in conversion attribution using Apache Beam, Conversion Attribution with Apache Beam-Conversion Attribution with Apache Beam
implicit state, Implicit State-Incremental Combining
in incremental combining, Incremental Combining
raw grouping of inputs, Raw Grouping
motivation for, Motivation-Correctness and Efficiency
correctness and efficiency, Correctness and Efficiency
inevitability of failure, The Inevitability of Failure
persistent timers, MillWheel
physical stages and fusion in Beam Model, A Holistic View of Streams and Tables in the Beam Model
post-declaration of triggers, The Beam Model: A Stream-Biased Approach
predeclaration of triggers, The Beam Model: A Stream-Biased Approach
processing time, When: Watermarks
conversion to event time in watermarks, When: Watermarks
delays in triggers, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
event time vs., Event Time Versus Processing Time
in SQL table UserScores (example), What: Transformations
in streaming SQL, Temporal Operators
shifting input observation order in, When/Where: Processing-Time Windows
view of batch processing, What: Transformations
watermarks, Processing-Time Watermarks-Processing-Time Watermarks
windowing by, Windowing by processing time
processing-time windowing, Summary
event-time windowing comparing two processing-time orderings of same iput, Event-Time Windowing
via ingress time, Processing-Time Windowing via Ingress Time-Processing-Time Windowing via Ingress Time
via triggers, Processing-Time Windowing via Triggers-Processing-Time Windowing via Ingress Time
processing-time windows, Advanced Windowing-Processing-Time Windowing via Ingress Time
downside to, When/Where: Processing-Time Windows
via ingress time, When/Where: Processing-Time Windows
via triggers, When/Where: Processing-Time Windows
Pub/Sub (see Google Cloud Pub/Sub)
Q
Questioning the Lambda Architecture post, On the Greatly Exaggerated Limitations of Streaming
R
raw grouping, Raw Grouping, Summary
merging windows, Incremental Combining
relational algebra, Relational Algebra
defining time-varying relations in terms of, Time-Varying Relations-Time-Varying Relations
relations (in databases), Relational Algebra
repeated delay triggers, Repeated delay triggers, Summary
repeated update triggers, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
firing with every new record, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
reprocessing the input, The Inevitability of Failure
Reshuffle transform, Exactly Once in Sinks, Example Sink: Google BigQuery
resilient distributed datasets (RDDs), Apache Spark Streaming, Spark
retractions (accumulating and retracting mode), How: Accumulation
in session window, Where: Session Windows
in streaming SQL, Retractions in a SQL world-Retractions in a SQL world
in Sys.Undo column (hypothetical) in streaming SQL, Streams and Tables
RIGHT OUTER joins, RIGHT OUTER
RPCs (remote procedure calls), use in shuffle and issues with RPCs, Ensuring Exactly Once in Shuffle
runners in Apache Beam, Beam
innovation in, Beam
S
savepoints, Flink
scalability
in large-scale data processing, MapReduce
in MapReduce, MapReduce, Summary
SCAN-AND-STREAM trigger, Materialized views
scheduling of processing, flexibility in, Generalized State, Conversion Attribution with Apache Beam
SDKs (software development kits) in Apache Beam, Beam
SELECT statement (SQL), STREAM and TABLE keywords after, Stream and Table Selection
SEMI joins, SEMI-Unwindowed Joins
session windows, Where: Session Windows-Where: Session Windows, Summary
variations on, Variations on Session Windows-Bounded sessions
bounded sessions, Bounded sessions
sessions, Sessions, Windowing
calculating length per user across two input pipelines, Understanding Watermark Propagation
interest from windowing standpoint, Where: Session Windows
manually building up on Spark Streaming 1.x (blog post), Where: Session Windows
retractions and, How: Accumulation
shuffles in a pipeline, Problem Definition-Problem Definition
ensuring exactly once in, Ensuring Exactly Once in Shuffle
Reshuffle transform, Exactly Once in Sinks
side effects
idempotent and robust in replay, Exactly Once in Sinks
in exactly-once processing, Side Effects
sinks, The SQL Model: A Table-Biased Approach
example sink, files, Example Sink: Files
example sink, Google BigQuery, Example Sink: Google BigQuery
sliding windows, Windowing
snapshots
Flink's highly efficient snapshotting implementation, Flink, Flink
in Apache Flink, Apache Flink
source watermarks, creation of, Case Study: Watermarks in Apache Flink
sources, The SQL Model: A Table-Biased Approach
exactly-once processing in, Exactly Once in Sources
example surce, Cloud Pub/Sub, Example Source: Cloud Pub/Sub
spam attacks, protecting against in conversion attribution pipeline, Case Study: Conversion Attribution
Spark Streaming (see Apache Spark Streaming)
Spark Streaming paper, On the Greatly Exaggerated Limitations of Streaming
SQL
Spark integration with, Spark
support for time-varying relations in, Streams and Tables
table-biased approach, The SQL Model: A Table-Biased Approach-Materialized views
types of joins defined in ANSI SQL, Unwindowed Joins
State Management in Apache Flink, Flink
Storm (see Apache Storm)
STREAM keyword (hypothetical, in SQL), Streams and Tables, Stream and Table Selection
STREAM queries (hypothetical, in SQL)
providing alternate data history to table-based TVR query, Streams and Tables
relation to OLTP tables, Streams and Tables
Sys.Undo column referenced from, Streams and Tables
streaming, Streaming 101-Summary
commonalities between batch and streaming, Cloud Dataflow
greatly exaggerated limitations of, On the Greatly Exaggerated Limitations of Streaming
microbatch vs. true streaming debate, Spark
support for stream processing in SQL materialized views, Materialized views-Materialized views
terminology, Terminology: What Is Streaming?-Event Time Versus Processing Time
unified batch plus streaming programming model, Cloud Dataflow, Beam
Zeitgeist pipeline, true streaming use case, MillWheel
streaming joins, Streaming Joins-Summary
unwindowed joins, Unwindowed Joins-Unwindowed Joins
FULL OUTER, FULL OUTER
INNER, INNER
LEFT OUTER, LEFT OUTER
RIGHT OUTER, RIGHT OUTER
SEMI, SEMI-Unwindowed Joins
windowed joins, Windowed Joins-Watermarks and temporal validity joins
fixed windows, Fixed Windows-Fixed Windows
temporal validity, Temporal Validity-Watermarks and temporal validity joins
streaming SQL, Streaming SQL-Summary
complete definition of, What Is Streaming SQL?
looking backward, stream and table biases, Looking Backward: Stream and Table Biases-Materialized views
looking forward, toward robust streaming, Looking Forward: Toward Robust Streaming SQL-Discarding mode, or lack thereof
stream and table selection, Stream and Table Selection
temporal operators, Temporal Operators-Discarding mode, or lack thereof
relational algebra as theoretical foundation of SQL, Relational Algebra
temporal validity window in, Temporal validity windows
time-varying relations, Time-Varying Relations-Time-Varying Relations
unwindowed joins
ANTI, ANTI
streaming systems, Terminology: What Is Streaming?
when and how of data processing, Going Streaming: When and How-How: Accumulation
how, accumulation, How: Accumulation-How: Accumulation
when, allowed lateness, When: Allowed Lateness (i.e., Garbage Collection)-When: Allowed Lateness (i.e., Garbage Collection)
when, early/on-time/late triggers, When: Early/On-Time/Late Triggers FTW!-When: Early/On-Time/Late Triggers FTW!
when, triggers, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!-When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
when, watermarks, When: Watermarks-When: Watermarks
streams, Terminology: What Is Streaming?
and tables, Streams and Tables-Summary
as data in motion, Toward a General Theory of Stream and Table Relativity
persistent forms of, Correctness and Efficiency
time-varying relations in, Streams and Tables
streams and tables
batch processing vs., Batch Processing Versus Streams and Tables-Reconciling with Batch Processing
how batch processing fits into stream/table theory, Reconciling with Batch Processing
how streams relate to bounded/unbounded data, Reconciling with Batch Processing
streams and tables analysis of MapReduce, A Streams and Tables Analysis of MapReduce-Reconciling with Batch Processing
comparing classic SQL and Beam Model, looking backward, Looking Backward: Stream and Table Biases-Materialized views
stream-biased approch in Beam Model, The Beam Model: A Stream-Biased Approach-The Beam Model: A Stream-Biased Approach
table-biased approach in SQL, The SQL Model: A Table-Biased Approach-Materialized views
conversions to and from in MapReduce, MapReduce
general theory of stream and table relativity, A General Theory of Stream and Table Relativity-A General Theory of Stream and Table Relativity
holistic view of in Beam Model, A Holistic View of Streams and Tables in the Beam Model-A Holistic View of Streams and Tables in the Beam Model
Kafka as embodiment of relationship between, Kafka
popularization of theory by Apache Kafka, Kafka
relationship between Beam Model and, Streams and Tables
special theory of stream and table relativity, Stream-and-Table Basics Or: a Special Theory of Stream and Table Relativity
table/stream selection for TVRs in streaming SQL, Summary
time-varying relations in, Streams and Tables-Streams and Tables
toward a general theory of stream and table relativity, Toward a General Theory of Stream and Table Relativity
view of windowed summation, Raw Grouping
what, where, when, and how of, What, Where, When, and How in a Streams and Tables World
how, accumulation, How: Accumulation
what, transformations, What: Transformations-What: Transformations
when, triggers, When: Triggers-When: Triggers
where, windowing, Where: Windowing-Window merging
strong consistency in a streaming system, On the Greatly Exaggerated Limitations of Streaming
subscriptions in Google Cloud Pub/Sub case study, Case Study: Source Watermarks for Google Cloud Pub/Sub
summation via incremental combination, Incremental Combining
Sys.EmitIndex column (hypothetical, in SQL), Watermark triggers, Summary
Sys.EmitTiming column (hypothetical, in SQL), Watermark triggers, Summary
Sys.MTime column (hypothetical, in SQL), Temporal Operators, Summary
Sys.Undo column (hypothetical, in SQL), Streams and Tables, How: accumulation, Summary
distinguishing between normal rows and rows retracting a previous value, Streams and Tables
T
TABLE keyword (hypothetical, in SQL), Streams and Tables, Stream and Table Selection
tables, Terminology: What Is Streaming?
as data at rest, Toward a General Theory of Stream and Table Relativity
conversion of streams from/to in SQL, The SQL Model: A Table-Biased Approach
explicit and implicit in SQL, The SQL Model: A Table-Biased Approach
persistent state, The Practicalities of Persistent State
table-based TVR vs. STREAM query TVR, Streams and Tables
time-varying relations in, Streams and Tables
tables and streams (see streams and tables)
temporal operators (in streaming SQL), Temporal Operators-Discarding mode, or lack thereof
triggers, When: triggers-Data-driven triggers
windowing, Where: windowing-Where: windowing
temporal tables (SQL), Streams and Tables
temporal validity, Temporal Validity-Watermarks and temporal validity joins
temporal validity joins, Temporal validity joins-Watermarks and temporal validity joins
watermarks and, Watermarks and temporal validity joins
temporal validity windows, Temporal validity windows-Temporal validity windows
time
event time vs. processing time, Event Time Versus Processing Time
partitioning in windowed joins, Windowed Joins
time-agnostic processing of unbounded data, Time-agnostic
filtering, Filtering
inner joins, Inner joins
tools for reasoning about, On the Greatly Exaggerated Limitations of Streaming
time-varying relations, Time-Varying Relations-Time-Varying Relations, Stream and Table Selection, Summary
defining in terms of relational algebra, Time-Varying Relations-Time-Varying Relations
for FULL OUTER joins, FULL OUTER
in temporal validity joins, Temporal validity joins
in temporal validity windows, Temporal validity windows
relationship with stream and table theory, Streams and Tables-Streams and Tables
timers, Generalized State, MillWheel
timestamps
event, Definition
system timestamp for Bloom filters, Bloom Filters
watermarks and, Definition
timing out a join, providing reference point for, Windowed Joins
tools for reasoning about time, On the Greatly Exaggerated Limitations of Streaming
tracking subscription (Google Cloud Pub/Sub case study), Case Study: Source Watermarks for Google Cloud Pub/Sub
transformations, What: Transformations-What: Transformations
in streams and tables model, What: Transformations-What: Transformations
triggers, Roadmap, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!-When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
completeness, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
watermark completeness trigger, When: Watermarks
early/on-time/late, When: Early/On-Time/Late Triggers FTW!-When: Early/On-Time/Late Triggers FTW!
with allowed lateness, When: Allowed Lateness (i.e., Garbage Collection)-When: Allowed Lateness (i.e., Garbage Collection)
in processing-time window via ingress time, Processing-Time Windowing via Ingress Time
in streaming SQL, When: triggers-Data-driven triggers, Summary
data-driven triggers, Data-driven triggers
repeated delay triggers, Repeated delay triggers
watermark triggers, Watermark triggers-Watermark triggers
in streams and tables model, When: Triggers-When: Triggers
blending of batch and streaming, When: Triggers
early/on-time/late firings, When: Triggers
per-record triggering in streaming engine, When: Triggers
trigger guarantees (or lack of), When: Triggers
windowed summation with heuristic watermark on streaming engine, When: Triggers
in unwindowed joins, Unwindowed Joins
predeclaration or post-declaration options, Beam Model and, The Beam Model: A Stream-Biased Approach
processing-time delays in
aligned delays, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
unaligned delays, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
processing-time windowing via, Processing-Time Windowing via Triggers-Processing-Time Windowing via Ingress Time
repeated update, When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
SCAN-AND-STREAM trigger in materialized views, Materialized views
use in achieving processing-time windowing, When/Where: Processing-Time Windows
tumbling windows (see fixed windows)
Twitter Heron, Storm
U
unaligned delays (processing time in triggers), When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things!
unaligned windows, Where: Session Windows
unaligned fixed windows, Unaligned fixed windows
unbounded data, Terminology: What Is Streaming?
processing by batch systems, Unbounded Data: Batch
processing by streaming systems, Unbounded Data: Streaming-Windowing by event time
time-agnostic processing, Time-agnostic
using approximation algorithms, Approximation algorithms
UnboundedReader.getCurrentRecordId method, Exactly Once in Sources
ungrouping operations, The Beam Model: A Stream-Biased Approach
(see also grouping operations)
unified batch plus streaming programming model
in Apache Beam, Beam
unpredictability in lambda architecture processing, Why Exactly Once Matters
unwindowed joins, Unwindowed Joins-Unwindowed Joins
ANTI, ANTI
FULL OUTER, FULL OUTER
INNER, INNER
LEFT OUTER, LEFT OUTER
RIGHT OUTER, RIGHT OUTER
SEMI, SEMI-Unwindowed Joins
V
visibility (watermarks), Definition
W
watermark triggers
in streaming SQL, Watermark triggers-Watermark triggers, Summary
watermarks, Roadmap, When: Watermarks-When: Watermarks, Watermarks-Summary, MillWheel-MillWheel
about, Definition
allowed lateness and, When: Allowed Lateness (i.e., Garbage Collection)
and temporal validity joins, Watermarks and temporal validity joins
applying perfect and heuristic watermarks to same dataset, When: Watermarks
as class of functions, When: Watermarks
case study, watermarks for Google Cloud Pub/Sub, Case Study: Source Watermarks for Google Cloud Pub/Sub-Case Study: Source Watermarks for Google Cloud Pub/Sub
case study, watermarks in Apache Flink, Case Study: Watermarks in Apache Flink
case study, watermarks in Google Cloud Dataflow, Case Studies-Case Study: Watermarks in Google Cloud Dataflow
for exactly-once garbage collection in Dataflow, Garbage Collection
function, converting processing time to event time, When: Watermarks
heuristic, When: Watermarks
in processing-time windowing via ingress time, Processing-Time Windowing via Ingress Time
in streaming SQL, Summary
percentile, Percentile Watermarks-Percentile Watermarks
perfect, When: Watermarks
processing-time, Processing-Time Watermarks-Processing-Time Watermarks
propagation, Watermark Propagation-The Tricky Case of Overlapping Windows
and output timestamps, Watermark Propagation and Output Timestamps-Watermark Propagation and Output Timestamps
with overlapping windows, The Tricky Case of Overlapping Windows
source watermark creation, Source Watermark Creation-Heuristic Watermark Creation
heuristic watermarks, Heuristic Watermark Creation
perfect watermarks, Perfect Watermark Creation
too fast, When: Watermarks
too slow, When: Watermarks
use in Cloud Dataflow, Cloud Dataflow
Windmill, MillWheel
windowing, Event Time Versus Processing Time, Where: Windowing-Where: Windowing, Summary, All Your Joins Are Belong to Streaming
(see also unwindowed joins)
accumulation modes for a window, Roadmap
by event time, Windowing by event time
by processing time, Windowing by processing time
custom, Where: Custom Windowing-One Size Does Not Fit All
benefits of, One Size Does Not Fit All
elements of custom windowing strategy, Where: Custom Windowing
variations on fixed windows, Variations on Fixed Windows-Per-element/key fixed windows
variations on session windows, Variations on Session Windows-Bounded sessions
different strategies for, Windowing
dynamic windows and retractions, How: Accumulation
end of window and output timestamp, Watermark Propagation and Output Timestamps
fixed windows, Windowing
in Cloud Dataflow, Cloud Dataflow
in streaming SQL, Where: windowing-Where: windowing, Summary
in streams and tables model, Where: Windowing-Window merging
lifetime of windows, When: Allowed Lateness (i.e., Garbage Collection)
nondeterministic records in windowed aggregation, Exactly Once in Sinks
overlapping windows and output timestamp, The Tricky Case of Overlapping Windows
session windows, Where: Session Windows-Where: Session Windows
sessions, Windowing
sessions in unbounded data processing by batch systems, Sessions
sliding windows, Windowing
summation code example, Where: Windowing
summation on a batch engine, Where: Windowing
summation on streaming dataset with perfect and heuristic watermarks, When: Watermarks
triggers for output, Roadmap
unbounded data processing via fixed windows in batch systems, Fixed windows
windowed file writes, Example Sink: Files
windowed joins, Windowed Joins-Watermarks and temporal validity joins
fixed windows, Fixed Windows-Fixed Windows
temporal validity, Temporal Validity-Watermarks and temporal validity joins
write and read granularity, flexibility in, Generalized State, Conversion Attribution with Apache Beam
Z
Zeitgeist pipeline, true streaming use case, MillWheel