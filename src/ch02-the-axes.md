# The Axes

You cannot make a useful decision between two formats without first
agreeing on the dimensions along which they differ. The mistake almost
everyone makes the first time they have to choose a format is to pick
one or two dimensions — usually speed and size — and try to rank
candidates on those alone. The choice that results is usually
defensible enough to ship, and usually wrong in some way that takes
two years to surface.

This chapter is a map of the dimensions that actually matter. There
are seven of them. Each is more or less independent of the others,
which means the space of meaningfully distinct formats is large, and
also that a format which takes a position on one axis is not
constrained on any other. A schemaless format can be deterministic
or not. A schema-first format can be row-oriented or columnar. A
zero-copy format can use codegen or runtime reflection (though in
practice they all use codegen, for reasons we'll get to). The seven
axes are not orthogonal in the strict mathematical sense — some
combinations are easier to engineer than others — but they are
independent enough that thinking of them separately is more useful
than not.

For each axis, the question to ask of a format is the same: *what
position does this format take, and what are the consequences?*

## Axis 1: schema-required vs. schemaless

A schema is a description, external to the bytes, of what structure
those bytes are supposed to have. *Schemaless* formats do not require
one. The bytes carry enough type and structural information to be
decoded into a generic representation — a map, a list, a number, a
string — without knowing anything in advance.

JSON is the textbook schemaless format, and the binary formats most
similar to it in spirit are MessagePack, CBOR, BSON, and Smile. You
can decode any well-formed payload in any of these into a tree of
generic values, and only then ask: *does this look like the thing I
expected?* The schema, if there is one, lives in your application
code and is enforced at the boundary between the generic tree and
your domain types. The format itself does not care.

*Schema-required* formats are the opposite. The bytes do not contain
enough structural information to be decoded without consulting the
schema. Protobuf, Thrift, FlatBuffers, Cap'n Proto, SBE, and Avro
are all schema-required. Without the `.proto` file, a Protobuf
message is a sequence of integer-tagged fields whose meaning — *is
field 5 a string or a uint32?* — cannot be determined from the
bytes alone. The wire format encodes a hint at the type (`length-
delimited`, `varint`, `fixed64`), but multiple types share each
hint, so the schema is not optional.

Avro is the interesting hybrid. Its wire encoding is so compact
that, taken in isolation, an Avro payload is meaningless. Field
ordering, length, and type are all dictated by the schema. But Avro
is almost always paired with a mechanism for distributing the
schema alongside the bytes — Object Container Files include the
schema in their header, and Confluent Schema Registry indexes the
schema by ID and ships the ID inline with each message. Avro is
therefore schemaless from the *bytes* perspective and schema-required
from the *protocol* perspective. The distinction comes up later when
we discuss the second axis.

The cost of schema-required formats is operational: the schema must
be present everywhere it is needed. The benefit is density and type
fidelity. The cost of schemaless formats is wire size and ambiguity:
every payload re-states its own structure, and the application is
responsible for catching the cases where producer and consumer
disagree about what that structure should mean. The benefit is
flexibility, especially in environments where the schema is hard to
ship in advance — public APIs, polyglot systems, exploratory work.

A common confusion is to claim a particular format can be used
"without a schema" by relying on the runtime's reflection facilities.
Protobuf has `DynamicMessage`; Thrift has its `Protocol` interface;
FlatBuffers has reflection via `.bfbs`. These are not schemaless
modes. They are schema-required modes in which the schema is loaded
at runtime instead of compile time. The schema is still mandatory.
The difference is whether you build it into your binary or ship it
alongside.

## Axis 2: self-describing vs. external schema

This axis is closely related to the first but not identical, and the
distinction is worth pulling apart because confusing them is a
common source of architectural error.

A format is *self-describing* if a payload, taken alone, contains
enough information to be decoded. JSON, MessagePack, CBOR, and BSON
are all self-describing: every value is preceded by a tag declaring
its type. A format requires *external schema* if you need information
that is not in the bytes to decode them. Protobuf, FlatBuffers,
Cap'n Proto, SBE, and naked Avro are all external-schema formats.

The interesting cell of the two-by-two table is *self-describing,
schema-required.* This is what an Avro Object Container File is:
the file requires a schema to interpret, and it includes that
schema in its own header. The bytes of the file are self-contained
in the sense that an Avro reader can decode them without consulting
any external resource. They are schema-required in the sense that
the schema must be present for anything to happen.

This combination is important for the time boundary. A format that
is schema-required *and* not self-describing produces bytes that
are only as durable as the schema's availability. If the schema is
in a registry that goes dark in 2030, the bytes written in 2024
become uninterpretable. If the schema is in a Git repository that
gets deleted, same problem. Avro's container format and Parquet's
embedded schema metadata both exist because the people who designed
those formats understood that *the schema must be archived with the
bytes* if the bytes are to outlive the producer system.

Self-describing formats also pay a cost: the type tags inflate the
wire size, and the receiver has to check at every step whether the
type it got is the type it expected. External-schema formats can
elide all of that, because the schema has already told them what to
expect. The cost is paid in operational complexity instead.

The decision rule worth internalizing: *self-describing formats are
the right default unless you have a specific reason to pay for the
density of an external-schema format.* The reasons are real — mostly
network bandwidth at scale, or zero-copy access — but they are
fewer than people think.

## Axis 3: row-oriented vs. columnar

The values of a record can be laid out in bytes in two fundamentally
different ways. Row-oriented formats keep all of a record's fields
together. To find the next record's `email`, you skip past the
current record's `id`, `name`, `email`, `birth_year`, `tags`, and
`active`. Columnar formats keep all values of one field together.
To read every record's `email`, you scan a contiguous block of
strings; to read every record's `id`, you scan a contiguous block
of integers.

Almost every format in this book is row-oriented. The columnar
exceptions — Apache Arrow IPC, Parquet, ORC, Feather — exist
because for analytical workloads, columnar layout is not slightly
better but dramatically better.

The reason is twofold. First, columnar layouts compress
spectacularly. A column of timestamps has enormous redundancy that
disappears when you delta-encode adjacent values; a column of
booleans is a bitmap; a column of small integers can be
dictionary-encoded down to a few bits per value. Mixing those
columns into rows ruins all of these compression opportunities.
Second, analytical queries usually touch a few columns out of many,
and a columnar layout lets you read only the bytes you care about.
A query that selects `id` and `birth_year` from a billion-record
dataset reads two columns; the row-oriented equivalent reads
the whole table.

Columnar formats lose, badly, on transactional access patterns.
Reading a single record means scattered reads to every column.
Writing a single record means appending to every column buffer.
The CRUD-style access pattern that operational systems live on
is exactly the pattern columnar formats are bad at.

Most systems that mix the two end up with a row-oriented
operational store and a columnar analytical store, with a transcoding
pipeline between them. Arrow's specific contribution is to be the
row-oriented-shaped *in-memory* representation that nonetheless
shares the columnar layout with the analytical store, so the
transcoding can be elided in many cases.

## Axis 4: zero-copy vs. parse

Most formats require a parse step: bytes go in, an in-memory object
graph comes out. The parse copies, allocates, validates, and
constructs. For most workloads this is fine, and the cost is
dominated by other things.

A few formats are designed so that the in-memory representation *is*
the byte representation. The bytes are aligned, padded, and laid out
so that the program can read fields directly from the buffer with
offset arithmetic, without any decode step. FlatBuffers, Cap'n Proto,
and rkyv are the canonical examples. Apache Arrow IPC is in this
family for analytical workloads.

The benefits are substantial. There is no decode time and no
allocation; an mmap'd file becomes an addressable data structure;
random access into a large message reads only the touched bytes.

The costs are also substantial. The format must use fixed-size
fields or pointers, which means padding and alignment requirements
that inflate the encoded size. Variable-length data sits at the
end of the buffer and is referenced by offsets. The schema cannot
freely change the layout of existing types without breaking
compatibility — adding a field is fine, reordering fields is not.
The format constrains how fields can refer to each other (no cycles,
typically). And the abstractions in your language tend to be
generated accessor objects rather than native structs, because the
fields you read are not necessarily where the language would put
them.

Zero-copy is the right answer when reads dominate writes by orders
of magnitude, when latency matters per-read, and when the data is
large enough that copying it would itself be the bottleneck. It is
the wrong answer when the format will be read once and discarded,
when wire size matters more than read latency, or when ergonomics
in the host language matter more than microsecond-scale access time.

## Axis 5: codegen vs. runtime

Schema-required formats need bindings — the code that reads and
writes the schema's types in your language. The bindings can be
generated at build time or constructed at runtime. The choice has
big consequences for ergonomics and build complexity, and it is
often invisible until you try to do something the format authors
didn't anticipate.

Codegen produces source files (or class files, or whatever your
language uses) that are compiled into your program. The generated
types know their schema at compile time, can be checked by the
type system, and are usually fast because the access paths are
specialized. The cost is a build step. Adding a field means
re-running the codegen and recompiling. Cross-language workflows
mean running the codegen for every language. CI pipelines acquire
opinions about which version of the codegen tool is the canonical
one, and disagreements between developers' local installs become
mysterious diff churn.

Runtime bindings parse the schema at startup, build an in-memory
representation of it, and then either expose a generic accessor
(`record.get("name")`) or use the host language's reflection to
build typed wrappers on the fly. The build is simpler. Adding a
field means updating the schema and restarting. But the access path
goes through a hash lookup or a dispatch table, the type system
cannot help you, and certain optimizations the codegen path enjoys
are unreachable.

Many formats support both modes. Protobuf has `DynamicMessage`,
Thrift has dynamic protocols, Avro has `GenericRecord`, JSON Schema
validators are by definition runtime. FlatBuffers and Cap'n Proto
strongly favor codegen because the zero-copy invariants are easier
to enforce when the accessors are generated.

The interesting case is the tooling that lives on top of a format.
A schema registry, a wire-format viewer, or a CDC pipeline almost
always needs the runtime mode, because those tools want to handle
arbitrary schemas they didn't know about at build time. If you are
writing such a tool, the format's runtime story matters more than
its codegen story. If you are writing an application server, the
codegen story usually wins.

## Axis 6: determinism

Given the same value, will the format always produce the same bytes?
Sometimes yes. Often no. Surprisingly often, the answer is *yes for
most values, no for some*, and the exceptions are the cases where
your hash function will quietly start producing different digests
for objects you considered equal.

Three positions exist on this axis.

*Deterministic by spec.* Encoding the same value always produces
the same bytes, and the spec mandates this. ASN.1 DER is the most
prominent example; it was designed for cryptographic uses where
deterministic encoding is essential. Borsh, used in the Solana
ecosystem, is deterministic by construction. SBE is deterministic
because every field has a fixed offset.

*Canonical form available, non-canonical forms also valid.* The
format permits multiple encodings of the same value, but defines a
canonical subset that is deterministic. CBOR has a canonical encoding
(deterministic encoding rules in the spec); Protobuf has a
"deterministic serialization" mode that is a best-effort, not a
guarantee, and explicitly not canonical across implementations or
versions. JSON has a long, sad history of canonicalization proposals.

*Non-deterministic.* The format makes no guarantees, and in practice
the encoding will differ between languages, library versions, and
sometimes between runs of the same program. Map ordering varies,
varint widths can differ, optional padding may or may not appear.
Most schemaless formats fall here unless you opt into a canonical
mode.

The reason determinism matters is that *equality of bytes is a
useful primitive.* Hashing requires it. Digital signatures require
it. Content-addressable storage requires it. Deduplication requires
it. Caching keyed on serialized values requires it. If your system
will ever do any of these things, the format's stance on determinism
is a question you have to answer up front. Deciding to retrofit
canonical encoding onto a system that has been writing
non-canonical bytes for two years is, generally, a project.

## Axis 7: evolution strategy

How does the format handle the situation where the schema changes,
producers and consumers run different versions, and you do not get
to upgrade them all at once? This is the axis that, in my
experience, kills more formats in the wild than any other.

The major strategies are:

**Tagged fields.** Each field has a stable identifier (a number,
sometimes a string) that is independent of its position or name.
Adding a new field uses a new tag. Removing an old field leaves the
tag retired but reserved. Producers and consumers that disagree on
the schema can still parse the bytes, ignoring tags they don't
recognize. Protobuf and Thrift work this way. The cost is a small
per-field overhead and the discipline of never reusing a tag number.

**Schema resolution by reader/writer.** Both the reader and writer
have schemas, and the format defines rules for resolving differences
between them. Avro is the canonical example: a reader's schema and
a writer's schema are reconciled at decode time, with rules for
field promotion, default values for missing fields, and aliases for
renamed fields. The bytes themselves carry no per-field metadata —
they rely on the writer's schema being available — but the resolution
rules let producers and consumers diverge gracefully. The cost is
that the writer's schema must travel with the bytes, or be available
through some side channel like a registry.

**Position-only.** Fields are identified by their position in the
record. Adding a field at the end is safe; adding one in the middle
is not; reordering is a breaking change. XDR works this way. Borsh
works this way. Most "just write the struct's bytes" formats work
this way. The cost is that schema evolution is heavily constrained
and easy to get wrong.

**Hash-based.** Each field is identified by a hash of its name and
type. Renaming a field is a breaking change; changing its type is
a breaking change. This is rare in mainstream formats but appears
in some content-addressed systems.

**None.** Schemaless formats often have no evolution strategy because
they have no schema; the application is responsible for handling
unknown fields, missing fields, and type drift on its own. The
format gives you maps and lists; what they mean is your problem.

Evolution intersects with two further questions: *what kinds of
compatibility does the format guarantee*, and *what does it do
when bytes don't match the expected schema*. The first question
yields a four-way classification — backward compatible, forward
compatible, full compatibility, no compatibility — which Avro
tooling has made standard vocabulary even though the underlying
ideas predate Avro by decades. The second question is the
decode-failure question from Chapter 1, and the answer is again
format-specific: silently drop, surface as an unknown, hard fail.

If a single axis deserves disproportionate attention when choosing
a format, it is this one. Performance differences between mainstream
formats are usually within an order of magnitude, and within an
order of magnitude rarely decides anything. Schema evolution
differences between mainstream formats can be the difference between
a system that survives ten years of organic change and a system
that becomes unmaintainable after eighteen months.

## Reading the rest of the book through this lens

Each format chapter that follows takes a position on each of these
seven axes. Sometimes the position is explicit and well-documented.
Sometimes it is implicit, and the format's authors have not written
it down because to them it was so obvious as to need no statement.
Part of the job of those chapters is to make the position explicit,
so that two formats can be compared on a level field.

When you finish a chapter and don't remember anything else, remember
where on these seven axes the format sits. That is enough to know,
in any new situation, whether the format is even a candidate.
