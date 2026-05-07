# Avro

Avro is the schema-first wire format that thinks about schemas
differently from Protobuf and Thrift, and it is worth getting that
difference clear before doing anything else with the format.
Protobuf and Thrift identify fields by stable numeric tags emitted
on the wire; the schema's job is to map those tags to types and
names, and the wire format carries enough information that a
decoder can skip fields it does not recognize. Avro identifies
fields by *position* — the wire format is just the field values
concatenated in schema order, with no tags at all — and relies on a
mechanism called *schema resolution* to handle version skew between
producers and consumers. The two approaches are not subtly different,
and the consequences cascade through everything else about Avro:
the wire format is denser, the schema is mandatory at decode time,
the evolution rules are formal and machine-checkable, and the
typical deployment shape involves a schema registry that the
typical Protobuf deployment does not.

## Origin

Avro was created by Doug Cutting in 2009 as part of the Apache
Hadoop ecosystem. Cutting (who had previously created Lucene,
Nutch, and Hadoop itself) wanted a serialization format that would
be the canonical row representation in Hadoop's MapReduce pipeline
and the canonical record format in HDFS. The constraints Cutting
faced were the constraints of an analytical pipeline: huge volumes
of records, all conforming to a known schema, written by jobs that
knew the schema at write time and read by jobs that knew the schema
at read time, with the two schemas potentially differing because
the pipeline evolved over months or years.

The first design choice that fell out of those constraints was
*schema-with-data*. Hadoop sequence files are large; embedding the
schema once at the head of the file amortizes to nothing per record
and gives every reader of that file the canonical interpretation
of the bytes. The Avro Object Container File format codifies this:
a magic number, a schema in JSON form, a synchronization marker,
and then a series of compressed blocks of records, each block
re-stating the synchronization marker so that a stream-aware
reader can skip ahead. The schema is part of the file. The file is
self-describing. The records inside the file are not.

The second design choice was *positional wire encoding*. Once the
schema is known at decode time — guaranteed by the file format, and
the convention by which Avro is used in Kafka via Confluent's
Schema Registry — there is no need for field tags. Records become
the concatenation of their field values, encoded in schema-declared
order, with each field's encoding determined by its declared type.
The result is the densest of the schema-required wire formats in
this book: no field numbers, no length prefixes on records, no
type tags on values.

The third design choice was *schema resolution*. When the schema
under which the bytes were written (the *writer's schema*) differs
from the schema the consumer wants to interpret them as (the
*reader's schema*), Avro defines explicit rules for reconciling
the two. Field reordering, type promotion (int → long → float →
double, in that order), default values for missing fields, and
aliases for renamed fields are all handled by the resolution
algorithm, which runs at decode time and produces the resolved
record in the reader's schema.

These three choices — schema-with-data, positional encoding, and
schema resolution — define Avro. Everything else in the format is
plumbing.

## The format on its own terms

Avro schemas are written in JSON. A primitive type is a JSON
string: `"int"`, `"long"`, `"string"`, `"boolean"`. A complex type
is a JSON object with a `type` field and additional properties.
A `record` has a `name`, optional `namespace`, and a `fields` array
where each field is an object with `name`, `type`, and optional
`default`, `doc`, `aliases`, and `order` properties. An `enum` has
a `name` and a `symbols` array. An `array` has an `items` type. A
`map` has a `values` type and string keys. A `union` is a JSON array
of types; the most common union is `["null", T]`, which encodes an
optional T. A `fixed` is a fixed-length byte string with a name.

The wire encoding for each type is small and ruthlessly literal.
Boolean: one byte, 0 or 1. Int and long: zigzag varint, exactly the
same encoding Protobuf calls `sint32`/`sint64`. Float: four bytes,
little-endian IEEE 754. Double: eight bytes, little-endian IEEE 754.
String and bytes: a long (length) followed by that many UTF-8 or
binary bytes. Records: the concatenation of their fields' encodings,
in schema-declared order, with no separator. Enums: a long giving
the symbol's index in the schema's symbols array. Arrays and maps:
a sequence of *blocks*, each consisting of a count (long) and that
many items, terminated by a zero-count block. Negative counts on
array and map blocks indicate that a byte size for the block follows
the count; this enables decoders to skip whole blocks without
parsing their contents, which matters for analytics workloads.

Unions are encoded as a long indicating which branch was selected
(0 for the first member of the union, 1 for the second, etc.),
followed by the value encoded according to that branch's type. For
the common case `["null", "string"]`, encoding null produces the
single byte 0x00, and encoding a string produces 0x02 followed by
the string encoding. This is the optionality mechanism: optional
fields are unions with null.

The order in which fields appear on the wire is the order they
appear in the schema. There is no flexibility, no version
information, no per-record schema identifier in the bytes. The
bytes are uninterpretable without the schema, and the schema must
be known by the decoder through some mechanism the format does not
specify (file header, registry lookup, side-channel).

The Avro Object Container File adds the framing that makes
self-described files practical. The file format is: a four-byte
magic (`Obj\x01`), a header containing the schema in JSON and a
randomly-generated 16-byte synchronization marker, and then a
sequence of data blocks. Each block contains the count of records,
the byte size of the compressed block, the compressed bytes, and
the synchronization marker repeated. Compression is per-block and
configurable (deflate, snappy, bzip2, xz, zstandard).

## Wire tour

Schema:

```json
{
  "type": "record",
  "name": "Person",
  "namespace": "com.example",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null},
    {"name": "birth_year", "type": "int"},
    {"name": "tags", "type": {"type": "array", "items": "string"}},
    {"name": "active", "type": "boolean"}
  ]
}
```

Encoded:

```
54                                           id: zigzag(42) = 84
18 41 64 61 20 4c 6f 76 65 6c 61 63 65       name: len 12, "Ada Lovelace"
02                                           email union branch 1 (string)
   2a 61 64 61 40 61 6e 61 6c 79 74 69 63 61 6c
      2e 65 6e 67 69 6e 65                   email value: len 21, "ada@analytical.engine"
ae 1c                                        birth_year: zigzag(1815) = 3630, varint
04                                             tags array block of 2
   1a 6d 61 74 68 65 6d 61 74 69 63 69 61 6e   "mathematician"
   14 70 72 6f 67 72 61 6d 6d 65 72            "programmer"
00                                             tags array terminator (0-count block)
01                                           active: true
```

67 bytes — the densest schema-first encoding of our Person record
in the book, narrowly beating Protobuf and Thrift. The wins come
from three places. First, no field tags: the record is just the
concatenation of values. Second, the union encoding for email is
a single byte plus the string, where Protobuf and Thrift each
spend a tag byte plus a length prefix. Third, the array encoding
uses a single zigzag-varint count rather than a per-element tag.

The cost is also visible. The bytes alone are uninterpretable.
There is no field number to scan for, no string key to match
against. A reader that does not have the schema cannot tell where
one field ends and the next begins, because the field boundaries
are not marked in the bytes; they are implicit in the schema.
This is the price of positional encoding, and Avro pays it
willingly.

If `email` were absent (the field with default null), the encoding
would emit `00` for the union branch (null), which is a single
byte rather than the 23 bytes of the present case. The schema's
default makes the absence well-defined: encoding a record without
the email value produces the null branch, and decoding the null
branch produces a record where email's deserialized value is
the schema-declared default, which is null.

## Evolution and compatibility

Avro's schema resolution rules are the most formal in this book.
Given a writer's schema and a reader's schema, the resolution
algorithm produces either a successful resolution (the bytes can
be decoded into the reader's schema) or a structural error (the
schemas are incompatible). The rules:

- A field present in both schemas with compatible types is decoded
  through type promotion if the types differ.
- A field present in the writer's schema but not the reader's is
  decoded and its value is discarded.
- A field present in the reader's schema but not the writer's is
  decoded as the schema-declared default. If no default is
  declared, the resolution fails.
- An enum symbol present in the writer's schema but not the
  reader's is decoded as the reader-schema-declared default. If
  no default, the resolution fails.
- A union with members in the writer's schema can be resolved
  against a reader's schema where the union members are a subset,
  with rules for null branches and type promotion.
- Aliases on the reader's schema let a field be matched to a
  differently-named field on the writer's schema.

The four standard compatibility modes are:

- **Backward compatible**: a reader using the new schema can read
  bytes written by the old schema. Field additions with defaults,
  field removals, and union promotions all preserve backward
  compatibility.
- **Forward compatible**: a reader using the old schema can read
  bytes written by the new schema. Field additions are forward-
  compatible if the reader is willing to drop unknown fields
  (which Avro readers are, by spec); field removals require the
  field to have had a default in the old schema.
- **Full compatibility**: both backward and forward compatible.
  Only schema changes that satisfy both rules are allowed.
- **No compatibility**: schemas may change arbitrarily; consumers
  must coordinate with producers.

The Confluent Schema Registry, the canonical schema-distribution
service for Kafka-based Avro deployments, enforces the chosen
compatibility mode at registration time. A producer that tries to
register a new schema that violates the topic's compatibility
policy gets rejected at the registry, before any incompatible
bytes can be written. This pre-deployment enforcement is the
single most valuable feature of Avro-with-registry as an
operational pattern, and it is the reason Avro has held up under
years of organic schema evolution in large Kafka deployments
where Protobuf would have required a lot more discipline.

The deterministic-encoding question for Avro is interesting.
Records are encoded in field order with no flexibility, integer
encodings are zigzag-varint with no width choice, strings are
length-prefixed with no padding option, and the only place
non-determinism creeps in is map field ordering (which Avro maps,
unlike Avro records, do not specify) and the choice of array
block sizes. A canonicalization layer that sorts map keys and
forces a single-block encoding is straightforward to add. Avro is
not deterministic by spec, but it is *closer to deterministic* than
Protobuf or Thrift.

## Ecosystem reality

Avro's primary ecosystem is the Hadoop / Kafka / Spark / Flink
analytics stack. Apache Avro is the canonical format for HDFS
records in Hadoop deployments; it is one of three first-class
formats in Confluent's Schema Registry (alongside Protobuf and
JSON Schema), and historically the default; it is the canonical
record format for Kafka Streams when typed records are needed; it
is supported natively by Spark's `from_avro`/`to_avro` functions
and by Flink's table API.

Outside the analytics stack, Avro's adoption is modest. The Avro
RPC story (Avro IDL plus the Avro RPC spec) exists but is not
widely deployed; gRPC and Thrift have taken that space. The Avro
schema language has a few warts — JSON syntax for schemas is
verbose, the *Avro IDL* alternative syntax exists but does not
have the tooling support — and the runtime libraries vary in
quality across languages. The Java implementation is canonical and
high-quality. The Python implementation (`fastavro`) is mature.
The Go and Rust implementations are functional but less polished
than their Protobuf equivalents.

The single most consequential ecosystem fact about Avro is that the
schema-with-data file format and the schema registry pattern *were
right*. Both have aged remarkably well. Both have proven robust to
the kinds of long-term schema evolution that Protobuf-without-Buf
struggles with. Both have spawned analogues for other formats — the
Confluent Schema Registry now supports Protobuf and JSON Schema
exactly because the operational pattern is more important than the
underlying serialization. If a single Avro contribution outlives the
format itself, it will be the registry pattern.

Two ecosystem gotchas. First, *logical types* — Avro's mechanism
for layering semantic types like decimal, UUID, timestamp on top of
primitive types — are implemented inconsistently across libraries.
A timestamp encoded by the Java client may not round-trip through
the Python client unless both are configured to recognize the same
logical type. This is a known issue and is improving, but it bites.

Second, the JSON encoding of Avro is a separate spec from the
binary encoding, and the two are not the same bytes. JSON
representations of Avro values are useful for debugging and for
interop with non-Avro consumers, but they should not be confused
with Avro itself; the binary encoding is the format, and the JSON
encoding is a debugging tool.

## When to reach for it

Avro is the right choice for analytical workloads on the
JVM-adjacent stack: Hadoop, HDFS, Hive, Spark, Flink. It is the
right choice for Kafka topics where the schema-registry pattern is
in place and the operational discipline of compatibility-checked
schema evolution is desired. It is the right choice for
long-lived data on disk where the schema must travel with the
bytes (Object Container Files solve this perfectly).

It is a defensible choice for inter-service typed serialization
when an operating registry is already in the stack and the team is
comfortable with positional encoding.

## When not to

Avro is the wrong choice for inter-service RPC in greenfield
deployments; gRPC plus Protobuf has won there and the Avro RPC
story does not compete. It is the wrong choice when the schema
cannot be reliably distributed (public APIs, polyglot
multi-organization deployments without registry infrastructure).
It is the wrong choice for tiny payloads where the overhead of
schema lookup or transmission exceeds the bytes saved.

It is the wrong choice when the team's mental model is
schema-as-tagged-fields (Protobuf-flavored) rather than
schema-as-positional-record (Avro-flavored); the conceptual
mismatch produces operational pain.

## Position on the seven axes

Schema-required. Self-describing only via container file framing —
the bytes alone are not. Row-oriented (the columnar Avro variants
discussed under Parquet inherit only the schema language). Parse
rather than zero-copy. Codegen optional; runtime use through
GenericRecord is common and idiomatic. Non-deterministic by spec
but close to it; canonicalization is straightforward. Evolution
by reader/writer schema resolution, with the strongest formal
compatibility model in this book.

The cell Avro occupies — schema-required, positionally-encoded,
resolution-based evolution — is unique among the formats in this
book and is structurally well-suited to the analytical-data
workloads it was designed for.

## Epitaph

Avro is the schema-first format that decided to put the schema in
the file rather than in the bytes, and the operational pattern that
followed (the schema registry) is more durable than any specific
wire format.
