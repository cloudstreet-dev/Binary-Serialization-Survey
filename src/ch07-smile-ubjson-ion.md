# Smile, UBJSON, Amazon Ion

The previous three chapters covered the canonical members of the
self-describing binary family: MessagePack, CBOR, BSON. There are
several others. They have not won, but they exist for reasons, and
each one illuminates a corner of the design space the canonical three
do not. This chapter covers Smile, UBJSON, and Amazon Ion together,
not because they are equivalent but because none of them is
sufficiently common to deserve its own chapter and the design choices
they make are interesting precisely as variations on the same theme.

## Smile

Smile is the binary JSON format that ships with Jackson, the JVM's
canonical JSON library. Tatu Saloranta, Jackson's primary author,
designed Smile in 2010 to occupy the same niche MessagePack does:
a schemaless binary format that is smaller and faster than JSON,
data-model-equivalent, and conveniently embedded in an existing
serialization library that most JVM applications already depended on.

Smile's central design choice is *key sharing*. JSON-shaped data is
typically dominated by repeated keys: every record in an array of
records repeats the same field names, and the bytes of those names
are most of the encoding. Smile maintains a table of recently-seen
keys during encoding and emits a back-reference when a key reappears.
For a sequence of fifty user records, the keys `id`, `name`,
`email`, `birth_year`, `tags`, and `active` are emitted in full
once and then referenced as one-byte tokens for the next forty-nine
records. The savings are substantial, and proportional to the
repetition. Single-record payloads see no benefit, and Smile's
encoded size for our Person record is approximately the same as
MessagePack's — but for the 50-record case, Smile is roughly 60% of
MessagePack's size.

A Smile stream begins with a four-byte header: `:)` followed by a
version byte and a feature flags byte. The header announces the
format and lets a reader detect a Smile stream without further
context. After the header come ordinary tokens. The token encoding
is similar to MessagePack's in spirit — variable-length tags, some
inline-value forms, length-prefixed strings — but with several
JVM-specific touches (a back-reference space for short shared
strings as well as keys, a distinct VLQ-style integer encoding,
explicit tokens for `true`, `false`, and `null` rather than packing
them into a fixed table).

Encoding our Person record:

```
3a 29 0a 03                                  Smile header ":)\x0a\x03"
fa                                           start object
80 69 64                                       key "id" (1-byte length-encoded short string)
24 54                                          value 42 (small integer encoding)
... (remaining fields encode similarly)
fb                                           end object
```

The full byte tour is around 90 bytes for a single record, with the
key-sharing benefits not yet engaged. For a stream of records the
tour would diverge from MessagePack's exactly at the second record,
where the keys would compress to back-references.

Smile has not spread beyond the JVM. Jackson uses it in
`jackson-dataformat-smile` and a handful of JVM-native systems
(Elasticsearch internally serializes some of its on-disk
representations as Smile, or did historically; some Hadoop adjacent
tools use it) consume it, but no major non-JVM ecosystem has a
production-quality Smile implementation. The format is a perfectly
reasonable design and a perfectly reasonable choice for JVM-only
systems with high payload repetition; outside that context it is
hard to recommend.

The interesting analytical point about Smile is that its
key-sharing optimization is the inverse of the schema strategy.
Schema-required formats (Protobuf, Avro, Cap'n Proto) eliminate
key bytes entirely by replacing them with field tags or positions.
Smile keeps the keys but compresses them with a context-sensitive
back-reference. The compression is less effective than full
elimination but does not require a schema. This is a genuine point
in the design space, and Smile's small constituency does not mean
the choice was wrong — only that the gains are not large enough to
overcome the operational cost of being a Java-only format in a
polyglot world.

## UBJSON

UBJSON, *Universal Binary JSON*, is the format you would design if
you cared more about being mistakable for JSON than about being
competitive on size. The encoding is a one-to-one binary mapping of
JSON values, with each value preceded by a single ASCII character
that denotes its type: `i` for int8, `U` for uint8, `I` for int16,
`l` for int32, `L` for int64, `d` for float32, `D` for float64,
`S` for string, `T` for true, `F` for false, `Z` for null, `[` for
array, `{` for object, `]` and `}` for the matching closers, and a
few others. The single-character type prefixes mean a UBJSON
encoder is almost trivial to write and a UBJSON dump is almost
human-readable when viewed in a hex editor: you can see the brace
characters, the field name strings, and the type codes without a
parser.

Strings are length-prefixed using a nested type-prefixed integer:
`S i 0c Ada Lovelace` says "string, int8 length, value 12, twelve
bytes of UTF-8". This recursive type-prefixed encoding generalizes
to lengths of any size, but it imposes overhead: every length
spends two bytes (the length type tag plus the length value) where
MessagePack and CBOR pack the length into a single byte. The
uniform syntax simplifies parsers; the cost in bytes is real.

Encoding our Person record (partial):

```
{                                            object open
i 02 i d                                       key "id" (string, int8 len 2, "id")
U 2a                                           value 42 (uint8)
i 04 n a m e                                   key "name"
S i 0c A d a 20 L o v e l a c e                value "Ada Lovelace"
... (remaining fields)
}                                            object close
```

The object has no count prefix — the close brace marks the end —
which differs from MessagePack and CBOR (both of which prefix the
count). UBJSON's syntactic close-brace makes streaming decoders
slightly simpler at the cost of disabling some quick-skip
optimizations. There is an optional "container with count" form
that prefixes the count after the open brace, but it is not
universally supported.

The ecosystem is small: a reference C library, a Python module, a
few JavaScript implementations, and not much else. UBJSON's
constituency is mostly hobby projects and a handful of game-engine
asset pipelines that want a simple, easy-to-debug binary format and
do not mind paying the bytes. There is no significant production
deployment that distinguishes UBJSON from its competitors, and the
size disadvantage is real enough that recommending UBJSON over
MessagePack or CBOR is hard outside of the specific case where
hex-readability matters more than density.

The interesting point UBJSON occupies is that single-character type
codes are an honest design choice when the system is small and
single-team-owned. They are easy to remember, easy to write a
parser for, and easy to read in a dump. The drawbacks compound at
scale: every value pays a byte for its type even when context would
have made the type obvious, every length is a recursive type-prefixed
integer, and there is no compression mechanism to recover the bytes.
UBJSON is a teaching example of the cost of being too literal a
mapping of JSON.

## Amazon Ion

Ion is the binary format Amazon designed to be the long-haul
serialization for systems that need typed values, schema flexibility,
and round-tripping between binary and human-readable forms. It was
released as open source in 2016 after being used internally for
years. The format has both a binary and a textual representation;
the two are equivalent, both are spec-defined, and any conforming
implementation must support both. This is the feature that
distinguishes Ion from MessagePack, CBOR, and Smile: a single
canonical text representation for every value, designed for human
inspection and editing.

Ion's data model is JSON-shaped but extended: integers of arbitrary
precision, decimals (not floats — *decimals*, with exact precision),
timestamps with timezone and arbitrary precision, *symbols* (interned
strings, encoded as integer references to a symbol table), *blobs*
(byte strings), *clobs* (text strings with non-Unicode encoding,
preserved verbatim), and *annotations* (one or more symbols
prepended to a value to label its semantic meaning). The annotation
mechanism is unique among self-describing formats and is used for
versioning, type hints, and schema attachment. Ion calls a value
with annotations a *typed value*; the annotations are part of the
value's identity, but a reader that does not understand them can
ignore them.

The binary encoding uses a one-byte type-and-length descriptor for
each value, where the high four bits encode the type (15 type codes
plus a few reserved) and the low four bits encode the length or a
length descriptor. Lengths beyond 13 are emitted as a separate
varint. Symbol tables are emitted at the start of a binary stream,
mapping symbol IDs to their string values; subsequent encoded
values use symbol IDs in place of the string keys, dramatically
reducing the bytes spent on field names. The symbol table mechanism
is conceptually similar to Smile's key sharing but more explicit:
the table is a first-class object in the stream, and shared symbol
tables can be referenced across multiple streams.

Encoding our Person record:

```
e0 01 00 ea                                  binary version marker (Ion 1.0)
ee 95 81 83 de 91                            local symbol table header
   87 be 8e                                  symbol list of 6
   83 69 64                                  symbol "id"
   84 6e 61 6d 65                            symbol "name"
   ...                                       (remaining symbols)
de 8e                                        struct with 14 bytes of content
   8a 21 2a                                  field 10 ("id"): int 42
   8b 8c 41 64 61 ...                        field 11 ("name"): string "Ada Lovelace"
   ...
```

The binary encoding is in the same density range as MessagePack
when symbol tables are not amortized, and substantially denser when
they are: a stream of a thousand records pays for the field names
once, in the symbol table, and then references them by integer ID
in every subsequent record.

The text representation of the same record is:

```
{
  id: 42,
  name: "Ada Lovelace",
  email: "ada@analytical.engine",
  birth_year: 1815,
  tags: ["mathematician", "programmer"],
  active: true,
}
```

This looks like JSON, with three differences: Ion's text uses
unquoted symbols for field names (quoted strings are still allowed),
Ion's text supports type annotations (`person::{id: 42, ...}`), and
Ion's text supports the extended types directly (timestamps as
`2023-01-01T00:00:00Z`, decimals as `1.5d0`, blobs as `{{base64...}}`).
The textual representation is what most Ion users see; the binary
representation is what gets stored.

Ion's adoption inside Amazon is broad. DynamoDB Streams, the Amazon
QLDB ledger, and several internal Amazon services use Ion in their
storage and inter-service formats. Outside Amazon, adoption is
modest: a handful of analytics tools, the occasional financial-data
service, and the obvious alignment with AWS-using teams that want
Ion's typed values without choosing a heavier format like Avro.
The reference implementations live at amzn/ion-* on GitHub and
cover Java, C, C#, Python, JavaScript, and Rust; the implementations
are mature and the wire format is stable.

The aspect of Ion most worth understanding, even if you never use
the format, is the symbol table mechanism. Ion separates *the
universe of strings used as identifiers* from *the encoding of
those identifiers in any particular value*, and that separation is
genuinely useful. Schemas in other formats (Protobuf, Avro) achieve
something similar by mapping names to integer tags or positions,
but they require a schema artifact to interpret. Ion does it within
the stream itself: the symbol table is part of the stream, the
stream is self-contained, and the trade between density and
self-description is settled by the symbol table's presence rather
than by an external schema's presence.

## Comparing the three

The three formats in this chapter share a stance on most of the
seven axes — schemaless, self-describing, row-oriented, parse,
runtime, no formal evolution rules — and differ on the others.
Smile is JVM-only and optimizes for repeated keys via back-references;
its determinism story is ad-hoc. UBJSON optimizes for syntactic
simplicity at the cost of bytes; it has no determinism story. Ion
optimizes for type fidelity and human-text round-tripping, with a
symbol table mechanism that gives it MessagePack-class density when
the table amortizes; it has a deterministic encoding subset.

For a JVM-only system that processes large arrays of similar
records, Smile is a defensible choice but a niche one. For a system
that wants the simplest possible JSON-shaped binary format, UBJSON
is honest but not competitive. For a system that needs typed
values, deterministic encoding, and a human-readable text form for
debugging, Ion is the strongest choice in this group, and arguably
the strongest choice in the schemaless self-describing family
overall — but its weight (the symbol table machinery, the broader
type system, the dual text/binary spec) is enough that for most
deployments MessagePack or CBOR remain easier to live with.

## When to reach for any of them

Smile: if you are JVM-only, Jackson is already a dependency, and
your data has high key repetition. Otherwise no.

UBJSON: hobby projects, learning exercises, deployments where
"a binary format I can read in a hex editor without a parser" is a
hard requirement. Otherwise no.

Ion: when you need rich types (timestamps, decimals, blobs), a
text representation that round-trips, and an annotation mechanism
for in-band type metadata. Strong fit for AWS-using teams and for
financial data systems where decimal exactness matters. Defensible
choice for general-purpose typed serialization where the alternative
would otherwise be Avro or Protobuf and the team does not want to
adopt a schema-required format.

## When not to

Smile, outside the JVM. UBJSON, when bytes matter. Ion, when the
broader ecosystem (libraries in your language, debugging tools,
operational comfort) is not strong enough; in non-Java/Python/Rust
languages, Ion's tooling is real but thin.

## Position on the seven axes

All three: schemaless, self-describing, row-oriented, parse,
runtime, evolution by application convention. They differ on
determinism (Ion has a canonical encoding; Smile and UBJSON do
not specify one) and on whether they have any in-band typing
beyond the JSON model (Ion has rich types and annotations; Smile
and UBJSON are JSON-equivalent).

## A note on the also-rans we left out

Several other formats deserve mention without deserving a chapter.
*BSON-CXX* and *EJSON* exist as MongoDB-adjacent variants and are
covered under BSON. *Concise Binary Encoding* (CBE) was a 2018 effort
by Karl Stenerud to design a successor to MessagePack and CBOR that
preserved their best properties while fixing perceived defects; it
has not gained adoption. *MUMPS-style globals* are a structured
binary representation with their own forty-year history; they are
genuinely interesting and genuinely outside the scope of any
survey-of-modern-formats chapter. *Lasso* and *PSON* exist as more
obscure JVM-adjacent formats with no significant deployment.
*FastJSON's binary mode* exists in some Alibaba-adjacent stacks.
None of these would change the analysis if included, and including
all of them would make the chapter exhausting without making it
more useful.

The point is that the self-describing schemaless binary format space
is crowded. The reason MessagePack and CBOR have absorbed most of
the oxygen is that they are good enough at the job, with ecosystems
that are large enough to mean someone has already debugged your
edge case. The variations covered in this chapter exist because
*good enough* did not satisfy somebody, and the dimension on which
it failed to satisfy is each format's organizing principle. None of
them is wrong. Most of them lost.

## Epitaph

Smile is Jackson's binary mode; UBJSON is JSON traced over with a
hex marker; Ion is the format Amazon uses when it wants typed
values to outlive the system that wrote them.
