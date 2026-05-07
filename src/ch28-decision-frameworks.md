# Decision Frameworks

A book that has spent twenty-six chapters describing formats
should not now resolve them into a flowchart that says "for X
problem, use Y format." That kind of recommendation is what most
articles about serialization formats produce, and it is wrong
often enough that the genre is mostly noise. The decision is not
amenable to flowcharting. The decision is amenable to
*structured questions you ask yourself in a useful order*, where
the answer to each question narrows the candidate set and
clarifies what the remaining trade is. This chapter is that set
of questions.

The order matters. The first questions are the ones whose answers
disqualify the most candidates; later questions discriminate among
the survivors. The questions are not all the same kind: some are
about the workload (how many records, how often, how big), some
are about the deployment topology (who controls what, what gets
upgraded together), some are about the team (what languages, what
operational appetite), some are about the data itself (what types
matter, how much nesting). All of them are important, and asking
them in the wrong order leads to wrong answers.

## Question 1: Are you crossing a process, machine, or time boundary?

This is the question chapter 1 introduced and that the rest of
the book has reinforced. The boundary determines which formats
are even candidates.

For a *process boundary* (two parts of one running program, two
processes on one machine, one process talking to itself across
threads), the relevant axis is encode/decode latency, not size.
Zero-copy formats are first-class candidates here: FlatBuffers,
Cap'n Proto, rkyv, Apache Arrow IPC. Parse-style formats also
work but rarely justify their parse cost.

For a *machine boundary* (one host to another, possibly different
architectures, possibly different language runtimes), the relevant
axes are size, schema portability, and encode/decode speed in
roughly that order. Schema-first formats (Protobuf, Avro, Thrift,
Bond) are first-class candidates. Self-describing formats
(MessagePack, CBOR) are candidates when schemalessness is
preferred. Zero-copy formats are candidates if the bytes are
loaded into memory and accessed many times before being discarded;
they are usually not candidates when the bytes are
written-once-read-once.

For a *time boundary* (bytes written today, read in years), the
relevant axes are schema durability, format stability, and the
availability of decoders into the indefinite future. ASN.1 has
the longest track record. Avro's Object Container Files put the
schema in the file. Parquet's columnar layout with metadata is
designed for archival. Most of the schemaless formats are also
defensible because they carry their own type information.
Schema-required formats without schema-with-data (Protobuf,
Cap'n Proto, FlatBuffers) require an out-of-band schema archival
plan that is often missing.

The boundary question disqualifies whole families. If you are
crossing a time boundary, you should not be using FlatBuffers
without an explicit plan to archive the schema for as long as
the bytes need to be read. If you are crossing a process boundary,
you should not be using JSON unless you have measured the cost and
decided to pay it.

## Question 2: Who controls the schema, and how often does it change?

The answer is one of: *I control both producer and consumer, and
I can deploy them together; I control both but they deploy
independently; I control one side but not the other; I control
neither.*

If you control both sides and deploy them together, you have no
schema-evolution problem. Almost any format works. Choose on
other axes (size, speed, ergonomics).

If you control both sides but they deploy independently
(microservices in heterogeneous deployment), you have a schema-
evolution problem. The format choice should be driven by which
evolution model fits your deployment topology. Tagged-field
formats (Protobuf, Thrift) are easier on small teams that can
follow conventions; resolution-based formats (Avro) are stronger
when a registry can enforce compatibility automatically. Both
require operational discipline that schemaless formats do not.

If you control one side but not the other (a public API, an SDK
consumed by external clients), the format must be resilient to
heterogeneous client implementations. Self-describing formats
work well; schema-first formats with mature codegen work also.
The choice often falls on whichever format has the broadest
client library coverage in the languages you expect clients to
use.

If you control neither side (interop with an external standard),
the format is determined for you. Read the spec; build to it.

## Question 3: What is the workload's read/write ratio?

A serialization format's costs are paid asymmetrically. Encoding
costs are paid by writers; decoding costs are paid by readers;
storage costs are paid wherever the bytes live. Different formats
shift these costs differently.

If reads dominate writes by orders of magnitude, zero-copy
formats become attractive. The decode cost amortizes to nothing;
the encode cost is paid once. FlatBuffers and Cap'n Proto are
designed for this. Apache Arrow IPC is too, in the in-memory
case. Even Parquet's metadata-driven access pattern is a kind of
read-favoring optimization.

If writes dominate reads, the encode cost matters more. Compact
varint formats (Protobuf, MessagePack) are competitive here;
zero-copy formats are penalized because their alignment
constraints add bytes that the writer pays to lay down.

If read and write rates are roughly equal, the formats with
balanced costs (Avro, MessagePack, Protobuf) are usually the
right answer. The choice between them is on other axes.

For analytical workloads with heavily-skewed read patterns
(many queries reading small column subsets out of huge tables),
columnar formats win unambiguously. Parquet for at-rest, Arrow
for in-flight.

## Question 4: Does the workload require deterministic encoding?

Determinism — the property that the same value always encodes to
the same bytes — is required when bytes are signed, hashed, or
content-addressed. Cryptographic protocols, blockchain consensus,
content-addressable storage, deterministic builds, and
signature-verification all need it.

If determinism is required, the candidate set narrows
substantially. ASN.1 DER is fully deterministic. CBOR has a
deterministic encoding subset. Borsh, SCALE, and SBE are
deterministic by construction. Cap'n Proto has a canonical
encoding mode. Protobuf, Thrift, MessagePack, Avro, BSON,
FlatBuffers, and the columnar formats are not deterministic by
spec; making them deterministic requires careful canonicalization
that is often not worth the effort.

If determinism is not required, this question disqualifies
nothing.

## Question 5: What is the language landscape?

Some formats are language-portable; some are not. The format
choice must include the question of whether good
implementations exist in every language that needs to consume
the bytes.

Formats with broad language support: Protobuf, JSON Schema,
MessagePack, CBOR, BSON, Avro. These have first-class
implementations in most languages used in production.

Formats with moderate language support: FlatBuffers, Cap'n
Proto, Thrift, Parquet, Arrow. These have good support in major
languages and weaker support in long-tail languages.

Formats with single-language or single-ecosystem support: rkyv
(Rust), bincode (Rust), postcard (Rust), Smile (JVM), Hessian
(JVM-centric, with thin support elsewhere), NBT (Minecraft
ecosystem).

If your language landscape is broad, choosing a single-language
format imposes a translation layer at every boundary. If your
landscape is narrow, the single-language format may give better
ergonomics than a portable one.

## Question 6: Does the data have structural features that constrain the choice?

Some data shapes are awkward in some formats. Worth checking
explicitly:

- *Heavily nested structures*: Parquet, ORC, and Cap'n Proto
  have substantial machinery for nesting. FlatBuffers' nesting
  story is workable but uses tables-within-tables in ways that
  add overhead. NBT's nesting is uniform and clean. Schemaless
  formats handle nesting trivially.
- *Cyclic references*: most formats cannot represent them
  directly. Hessian preserves them via its object-reference
  mechanism. Most others require flattening at the application
  layer.
- *Heavy repeated string columns*: dictionary-encoded columnar
  formats (Parquet, ORC) handle this dramatically better than
  any row-oriented format. Smile's key sharing helps in the
  schemaless case.
- *Floating point with NaN preservation*: most formats preserve
  NaN bit patterns; some (older JSON-based encodings) do not.
  Check if it matters.
- *Arbitrary-precision integers and decimals*: CBOR, Ion, ASN.1,
  and a few others handle these natively. Most formats do not
  and require encoding-as-strings or as-bytes.
- *Optionality at scale*: formats vary in how cheaply they
  encode "field absent." Avro, Hessian, and the schemaless
  formats are cheapest. Borsh, Postcard, and Bond are middle.
  SBE is most expensive (requires sentinel encoding).

If your data has one of these features and the candidate format
handles it badly, the workaround usually becomes a permanent
operational cost.

## Question 7: What is the operational appetite of the team?

This is the question most often unasked. Schema-evolution-aware
formats require operational infrastructure: a schema registry, a
breaking-change detector, a build process that regenerates code
on schema changes. Zero-copy formats require alignment
discipline. Columnar formats require batching infrastructure.
Determinism requires either a format that provides it by spec
or a canonicalization layer the team builds and maintains.

If the team has the appetite for the surrounding infrastructure,
formats that demand it (Protobuf with Buf, Avro with Confluent
Schema Registry, Parquet with table format projects) work well.
If the team does not, formats that demand less infrastructure
(MessagePack, CBOR, JSON) work better despite their other
drawbacks.

The single most common cause of long-term format-choice regret
is that a team chose a format whose operational requirements
exceeded what the team was prepared to build, and so the format
runs without its intended supports and produces the bugs the
supports were meant to prevent.

## A decision matrix

Putting the questions together, the typical paths look like:

| Workload                                    | Recommended            | Backup choice           |
|---------------------------------------------|------------------------|-------------------------|
| Internal microservice RPC, polyglot         | Protobuf + gRPC        | Avro + registry         |
| Internal microservice RPC, single-language  | rkyv (Rust) or Bond    | Protobuf                |
| Public API, broad clients                   | JSON                   | Protobuf with JSON      |
| Streaming events with schema registry       | Avro                   | Protobuf                |
| Analytical at-rest                          | Parquet                | ORC                     |
| Analytical in-memory interchange            | Arrow IPC              | Feather V2 (= Arrow)    |
| Game asset loading                          | FlatBuffers            | rkyv (Rust-only)        |
| Hot-path RPC with read-dominance            | Cap'n Proto            | FlatBuffers             |
| Cryptographic signing                       | ASN.1 DER              | CBOR canonical          |
| Blockchain on-chain encoding                | Borsh / SCALE          | Custom format           |
| Embedded telemetry                          | postcard               | CBOR                    |
| Configuration and human edits               | not binary             | Protobuf prototext      |
| Document database                           | BSON                   | (avoid; use a database) |
| Telecom protocol                            | ASN.1 PER              | (no good alternative)   |
| Robotics middleware                         | ROS msgs (CDR)         | (no alternative)        |

The matrix is a starting point, not an answer. Read the
relevant chapters; weigh the trade-offs against your specific
context.

## A few principles to internalize

A handful of cross-cutting observations from the book worth
pulling out:

*The format is an answer to a question; choose the format
whose question you are actually asking.* If the question is
"how do I encode my Rust types densely for a single-machine
cache," postcard is right and Avro is wrong. If the question is
"how do I move records between heterogeneous services with
strong evolution guarantees," Avro is right and postcard is
wrong. The questions are not interchangeable.

*Operational maturity outranks technical merit.* Bond is
slightly better-designed than Protobuf in several specific
ways and has lost decisively. The reason is that Protobuf has
better tooling, better documentation, broader community
knowledge, and better integration with the surrounding stack.
The same calculus applies to most format choices: the format
with the larger ecosystem and better tools usually wins,
regardless of whether it is technically superior.

*Schemaless is not free.* Schemaless formats look like they
let you skip the work of defining a schema. They do not. The
schema still exists; it is in the application code,
distributed across producers and consumers, undocumented and
unenforced. Schemaless formats are the right choice when the
schemalessness is what you actually want; they are the wrong
choice when you wanted a schema but didn't want to do the
work.

*Zero-copy is a constraint, not a feature.* Zero-copy formats
trade encoding flexibility for read latency. The trade is
right for some workloads and wrong for most. Choose zero-copy
when the read latency dominates your performance budget, not
because the marketing materials claim speed.

*Determinism is a feature you must choose explicitly.* Most
formats are non-deterministic by default. If you need
deterministic encoding, choose a format that provides it by
spec or commit to building canonicalization yourself. Both are
real costs; pretending otherwise leads to silent hashing bugs
years later.

*Almost all benchmarks lie.* The next chapter (Anti-Patterns)
covers the specific ways. The general advice: do not let
published benchmarks make your format choice.

## A worked decision example

Concretizing the framework on a hypothetical: a team is building
a new event-sourcing system for a financial-services back office.
Events are written to durable storage, replayed for state
reconstruction, and audited periodically. Producers and consumers
are independently deployable services in Java and Python.
Throughput is moderate; latency is not the binding constraint;
audit and replay must work indefinitely.

Question 1: this is primarily a *time boundary* problem. The
events live for years. Schema-with-data is essential. ASN.1,
Avro container files, and Parquet are first-class candidates.
Schema-required formats without in-band schema (Protobuf,
Cap'n Proto) require an explicit schema-archival plan.

Question 2: producer/consumer schema control is split, with
independent deployment. Schema evolution is a real concern.
Avro with a registry and tagged-field formats with discipline
both work.

Question 3: read/write ratio is heavily read-dominated (events
written once, read on every replay). Zero-copy is not relevant
because the events are batched into a stream. Compact encoding
matters because storage cost compounds.

Question 4: determinism is *not* required (events are
identified by ID, not by content hash).

Question 5: language landscape is Java + Python — both well
supported by Avro and Protobuf, less well by some of the others.

Question 6: events are flat-ish records with occasional
nesting. No special structural features.

Question 7: operational appetite for a registry exists (the
audit requirement justifies the investment).

The recommendation falls out: Avro with the Confluent Schema
Registry, stored as Avro Object Container Files. The schema is
in the file (good for the time boundary), evolution is registry-
enforced (good for the deployment topology), the Java and Python
implementations are mature, and the storage cost is competitive.

Note that the recommendation is not "the best format" in the
abstract. It is the format whose answers to the seven questions
match the team's situation. A different team in a different
context (microsecond-latency RPC, single-language deployment,
no registry tooling) would land on a different format.

## On the cost of revisiting the choice

A final observation worth stating explicitly: the cost of
choosing the wrong format is mostly paid at *migration time*.
Production data in a binary format is a substantial commitment;
moving it to a new format requires either dual-writing (both
formats during a transition window), translating (running
producer-side and consumer-side translation code, adding
latency), or freezing (stopping the migration when "good
enough"). All three are expensive, and all three accumulate
their own bugs.

The right time to make the format choice carefully is *before*
production data starts piling up. Once you have a petabyte of
Parquet, you are not switching to Arrow's IPC file format casually,
even though Arrow IPC is structurally similar. Once you have a
fleet of services using MessagePack, you are not switching to
Protobuf without a coordinated effort. Once you have a Hive
metastore full of ORC files, you are not switching to Parquet
without a migration project.

The book's working assumption is that you are reading this
*before* you have made the choice, or *while* you are
considering whether the existing choice is right for the next
phase of your system. The decision framework is meant to be used
in that window. Outside it, the framework is mostly aspirational;
the costs of switching usually outweigh the costs of staying.
