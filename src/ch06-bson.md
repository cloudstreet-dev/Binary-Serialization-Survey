# BSON

BSON is the easiest format in this book to misunderstand. The name
suggests *binary JSON*, the spec lives at bsonspec.org, and the wire
format encodes JSON-shaped values into bytes that are denser than JSON
and smaller than nothing else in particular. Read at face value, BSON
sounds like a competitor to MessagePack and CBOR. It is not. BSON was
designed to solve a different problem, and judging it as a binary JSON
on size alone produces the entirely correct conclusion that it is
inferior to MessagePack and CBOR at the job. Understanding BSON
requires understanding what it was actually built to do, which is to
serve as a storage and indexing format for a document database whose
queries needed to skip through large documents quickly.

## Origin

BSON was designed at 10gen, the company that became MongoDB Inc., in
2009. MongoDB was built around the idea that the database's storage
format and the application's data format should be the same: documents
are JSON-shaped objects, on disk and in memory, and queries operate on
those documents directly without translation through a relational
schema. This required a binary serialization for the documents on
disk. The serialization had to be writable, readable, and traversable
by query engines that wanted to skip past fields they did not care
about.

The constraints that emerged from those requirements turned out to be
quite different from MessagePack's. MessagePack optimizes for size:
small values get small encodings, length prefixes are minimal, integer
widths shrink to fit. BSON optimizes for *traversal*: every value is
preceded by a length-or-equivalent that lets a reader skip the value
without parsing it; integers are fixed-width so a comparison can
happen in place; field names are null-terminated strings so they can
be matched against query predicates byte-for-byte.

The result is a format that is consistently larger than MessagePack
or CBOR, sometimes by a factor of two or more, and consistently
faster to traverse for the access patterns MongoDB's query engine
cares about. If you are not running a document database, you are
not paying for BSON's strengths and are paying full price for its
weaknesses. This is why BSON is not common outside MongoDB.

## The format on its own terms

A BSON document is a contiguous block of bytes with the following
shape: a 4-byte little-endian integer giving the total document size
(including the size field itself); a sequence of elements; a single
trailing byte 0x00 marking the end of the document. The document size
field exists so that any reader, given a pointer to a document, can
allocate exactly the right buffer or skip to the byte after the
document without examining its contents. Embedded documents (which
appear in the wire format as ordinary values) carry their own size
field for the same reason.

An element has three parts: a single type byte, a null-terminated
field name, and a value. The type byte is one of about twenty
defined codes: 0x01 for double, 0x02 for UTF-8 string, 0x03 for
embedded document, 0x04 for array, 0x05 for binary, 0x07 for
ObjectId (a MongoDB-specific 12-byte identifier), 0x08 for bool,
0x09 for UTC datetime (a 64-bit milliseconds-since-epoch),
0x0a for null, 0x0b for regex, 0x10 for int32, 0x11 for timestamp
(a MongoDB internal type, not a generic timestamp), 0x12 for int64,
0x13 for the 128-bit decimal type added later, and a handful of
deprecated codes that earlier versions used and that current
encoders avoid.

The field name is a C-style null-terminated string. The value's
encoding depends on the type byte. Strings are length-prefixed
(little-endian 32-bit length, including the null terminator),
followed by UTF-8 bytes, followed by an explicit null. Arrays are
encoded as embedded documents whose field names are the strings
"0", "1", "2", and so on; this means an array of one million
elements has one million field names, all of which are integers
formatted as decimal strings, all of which are null-terminated.
This is one of the most striking design choices in the format and
the one that produces most of BSON's size overhead. Integers are
fixed-width little-endian. Doubles are IEEE 754 little-endian.
Booleans are a single byte: 0x00 for false, 0x01 for true.

Several of the type codes encode types that have no analogue in JSON
at all. ObjectId is twelve bytes structured as four bytes of
seconds-since-epoch, five bytes of a per-process random value, and
three bytes of a sequential counter; the structure is documented
because tools want to extract the timestamp portion without
consulting MongoDB. UTC datetime is a single 64-bit integer giving
milliseconds since the epoch; it is signed, which means negative
values are valid and represent dates before 1970. Decimal128 is the
IEEE 754-2008 128-bit decimal type, designed for financial data
where binary floats lose pennies. The type set is wider than
MessagePack's and CBOR's because BSON is the storage format for a
database that has to preserve types its applications care about
even when JSON does not.

Every element is implicitly tagged with both type and length. The
length is implicit for fixed-width values and explicit for
variable-width ones. The trailing null on strings is redundant given
the length prefix; the spec keeps it for compatibility with code
that wants to treat strings as null-terminated C strings, which is
in fact what some of MongoDB's older internals did.

Field name ordering in a BSON document is preserved on the wire and
by most consumers, which is a consequential detail: BSON documents
that were emitted in different orders are different bytes, even if
they are equal as JSON objects. Some MongoDB query operations rely
on this ordering, and the canonical encoding rules for digital
signatures over BSON depend on it.

## Wire tour

Encoding our Person record:

```
94 00 00 00                                  document length 148 (LE)
12 69 64 00                                  type 0x12 (int64), key "id"
  2a 00 00 00 00 00 00 00                    value 42 (LE int64)
02 6e 61 6d 65 00                            type 0x02 (string), key "name"
  0d 00 00 00                                  string length 13 (12 + null, LE)
  41 64 61 20 4c 6f 76 65 6c 61 63 65 00       "Ada Lovelace\0"
02 65 6d 61 69 6c 00                         type 0x02 (string), key "email"
  16 00 00 00                                  string length 22 (21 + null, LE)
  61 64 61 40 61 6e 61 6c 79 74 69 63 61
     6c 2e 65 6e 67 69 6e 65 00              "ada@analytical.engine\0"
10 62 69 72 74 68 5f 79 65 61 72 00          type 0x10 (int32), key "birth_year"
  17 07 00 00                                  value 1815 (LE int32)
04 74 61 67 73 00                            type 0x04 (array), key "tags"
  2c 00 00 00                                  inner doc length 44 (LE)
  02 30 00                                     type 0x02 (string), key "0"
    0e 00 00 00                                  length 14
    6d 61 74 68 65 6d 61 74 69 63 69 61 6e 00    "mathematician\0"
  02 31 00                                     type 0x02 (string), key "1"
    0b 00 00 00                                  length 11
    70 72 6f 67 72 61 6d 6d 65 72 00             "programmer\0"
  00                                           inner doc terminator
08 61 63 74 69 76 65 00                      type 0x08 (bool), key "active"
  01                                           value true
00                                           document terminator
```

148 bytes — about 40% larger than MessagePack and CBOR. The overhead
is in the places you would expect: 64-bit integer for `id`, fixed
32-bit length prefixes on every string (even when one byte would
suffice), explicit null terminators on every key and string,
explicit type byte even where the value byte already implies the
type, and the array-as-document encoding that names the elements
"0" and "1". The largest single source of overhead in this
particular record is the pair of array key bytes, which together
take six bytes (`30 00 31 00` plus their preceding type bytes) to
say what MessagePack says in zero.

The `id` field is also worth noting. BSON has both 32-bit and 64-bit
integer types, but most BSON producers default to 64-bit when the
source language's integer is 64-bit, regardless of whether the value
fits in 32 bits. This is the conservative choice; it preserves type
fidelity at the cost of bytes. The MongoDB shell will emit `id` as
int64 by default. To force int32 you have to construct a
NumberInt() value, which is rarely done. The result is that BSON
in practice spends 8 bytes on most numeric IDs, where MessagePack
or CBOR would spend one or two.

The trade is what the format is *for*. In MongoDB, when a query
selects records where `id == 42`, the query engine reads the document
length, jumps to the elements, scans field names looking for `id\0`,
and once found compares 42 against the eight bytes that follow. The
comparison is a single integer load. No parsing, no allocation, no
type-aware traversal. BSON's structure makes this access pattern
fast, and that access pattern is the entire point.

## Evolution and compatibility

BSON has the same evolution story MessagePack and CBOR have at the
field level: keys are strings, the application decides what to do
with unknown keys, removing a field means it is no longer in the
document. There is no formal schema, no version field, no
forward/backward compatibility policy enshrined in the format. The
tooling on top — MongoDB's schema validation, the various ODM
libraries — provides whatever evolution discipline a particular
deployment uses.

The format itself has evolved in three ways since 2009. Decimal128
was added (type 0x13). Several deprecated types from very early
versions were removed from the canonical encoding rules but
remained in the type byte registry to preserve decoder
compatibility; producers no longer emit them, but decoders still
handle them. The canonical-extended-JSON specification was
introduced as a textual representation of BSON values, used in
shell output and tools, distinct from both JSON and BSON. None of
these changes are wire-incompatible.

The deterministic encoding question for BSON is a live one. There
is no spec-defined canonical encoding, and BSON producers do
preserve field order, which makes byte-equality possible if the
producer is careful — but only if the producer is careful. MongoDB's
internal wire protocol does not require canonical encoding because
its uses do not need it; signing-over-BSON schemes (some of which
exist in the audit-log space) define their own canonicalization
rules.

## Ecosystem reality

Outside MongoDB, BSON is rare. Inside MongoDB, it is everything.
The driver libraries are mature in every language MongoDB supports
officially: C, C++, Go, Java, JavaScript/TypeScript, Python, Ruby,
Rust, Scala, Swift, .NET, PHP, and a few others. The Go driver's
BSON encoder/decoder is the implementation most often used outside
the MongoDB context, partly because it is well-documented and
partly because Go's encoding/json shape generalizes naturally.
Several non-MongoDB projects use the Go driver's BSON package as
a general binary format on the strength of "we already have a
dependency on it"; this is a defensible choice, but it is the long
way around to MessagePack.

The notable non-MongoDB uses are: GridFS, MongoDB's file-storage
abstraction, which is technically a separate format but uses BSON
as its metadata representation; some logging and audit systems
that wanted BSON's typed values; and a handful of replication and
backup tools that interoperate with MongoDB and chose BSON for
that reason. None of these are general-purpose binary serialization
deployments. None of them suggest that BSON is the right format
for a new system unrelated to MongoDB.

The shell and the various MongoDB tools speak a JSON dialect called
*Extended JSON* that round-trips BSON values, including the types
that have no JSON equivalent. Extended JSON has two modes:
*relaxed* (which uses regular JSON for round-trippable values and
falls back to type-tagged objects for the rest) and *canonical*
(which always uses type-tagged objects, even for values that have
JSON equivalents). The `mongoexport` tool emits relaxed extended
JSON by default; many of the tools downstream of mongoexport have
to handle the type tags, and this is a common source of pipeline
bugs.

## When to reach for it

If you are using MongoDB, you are using BSON, and the question does
not arise. If you are interoperating with MongoDB at the protocol
level — building a tool that reads MongoDB's oplog, implementing a
replica of MongoDB's wire protocol, processing MongoDB backup files
— BSON is the right format because it is the only format that
works.

The question does arise if you are building a document database of
your own, and asking whether to use BSON as the storage format
because MongoDB did. The answer is usually no: the design tradeoffs
that work for MongoDB's specific query engine may not be the right
ones for yours. CouchDB, RethinkDB, and a few others made different
choices for good reasons.

## When not to

Outside the MongoDB context, BSON is the wrong format for almost
any new system. It is larger than MessagePack and CBOR. It has no
schema language. It has no canonical encoding. It has no formal
evolution rules. It has type codes for things you do not need
(ObjectId, MongoDB Timestamp, deprecated types) and lacks features
you might want (efficient integer encoding, tagged extension,
columnar layout). It exists to serve a database, and away from
that database it is an inferior implementation of a problem
several other formats solve better.

The temptation to use BSON arises specifically because some teams
already have it as a dependency, and the temptation should be
resisted. Switching to MessagePack or CBOR is not a large project
in any language with mature support for both, and the bytes-on-disk
savings will pay back the migration in any deployment large enough
to make the question matter.

## Position on the seven axes

Schemaless. Self-describing. Row-oriented. Parse rather than
zero-copy, except in the in-document-traversal sense MongoDB's
query engine uses. Runtime bindings. Non-deterministic by spec, but
field order is preserved on the wire so deterministic-with-care is
achievable. Evolution by application convention.

The unusual cell BSON occupies on this map is *self-describing,
schemaless, and yet structurally optimized for in-place
traversal*. MessagePack and CBOR make the same axis choices but
optimize for size, not traversal. The result is that BSON looks
like the worst of both worlds when judged on size and the best of
both worlds when judged on the access patterns MongoDB cares about.
Outside that context, the access-pattern advantage evaporates and
only the size disadvantage remains.

## A note on the array-as-document encoding

The single most-criticized design choice in BSON is the array
encoding. Arrays are documents whose keys are decimal-formatted
indices: "0", "1", "2", and so on, each null-terminated, each
preceded by its element's type byte. For a thousand-element array
the keys alone occupy approximately three kilobytes, none of which
carry information. The motivation for this encoding is consistency:
arrays are documents, documents have keyed elements, and the
recursive structure means a generic document parser handles arrays
without special cases. The cost is paid at every BSON-encoded
array, in every database, every day.

Several proposals to add a more compact array encoding have been
made over the years and rejected. The reason for rejection is
backward compatibility: every BSON parser ever written assumes
that arrays are documents with stringified-integer keys. Changing
the encoding would either require all parsers to be updated
simultaneously (impossible) or introduce a separate type code for
the new encoding (creating two ways to do the same thing, which is
worse). The cost of fixing the design exceeds the cost of leaving
it alone, and so it has been left alone for fifteen years and
counting. This is the canonical example of a wire format whose
mistakes are permanent.

## Epitaph

BSON is MongoDB's storage format wearing JSON's vocabulary; competent
inside that database, awkward outside it.
