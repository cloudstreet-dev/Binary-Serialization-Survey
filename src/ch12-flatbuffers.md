# FlatBuffers

FlatBuffers is the format you reach for when the parse step is the
bottleneck. The promise is simple and unconventional: the bytes on
disk and the bytes in memory are the same bytes. There is no decode.
You access fields directly out of the buffer, with offset arithmetic,
and the offsets are arranged so that the access patterns common in
the calling language are cheap and the access patterns rare in the
calling language are still possible. This is not a trivial promise to
keep, and the format pays for it in bytes, in code-generation
complexity, and in constraints on what schemas can express. The trade
is sometimes worth it. When it is, FlatBuffers is the cleanest
expression of the idea on offer.

## Origin

FlatBuffers was built at Google by Wouter van Oortmerssen, formerly of
the game-development industry and at the time on Google's Android
games team. The motivating use case was loading game assets at
startup: a typical game ships with megabytes of structured asset
metadata (level layouts, sprite descriptors, animation curves, audio
manifests) that needs to be available immediately, and the parse
step in formats like JSON or Protobuf was a measurable contributor to
load time. Loading on a constrained device (a phone, a console)
without a parse step would let games start faster and use less RAM
during the load phase, which is approximately when devices have the
least memory available.

FlatBuffers was open-sourced in 2014. Its early adoption was
concentrated in games and embedded systems, where the parse-time
constraint was tightest. The format then accreted a second
constituency: ML model serialization. TensorFlow Lite uses
FlatBuffers as its on-disk model format (the `.tflite` extension is
literally a FlatBuffers buffer with a particular schema), partly
because mobile inference is performance-sensitive in the same way
mobile games are, and partly because FlatBuffers' zero-copy access
maps well to the way ML inference engines want to read model
weights. A third constituency is high-throughput RPC: Cocos2d-x
games, internal Google services where Protobuf's parse cost was
identified as a hotspot, and a few financial systems that wanted
zero-copy without going all the way to SBE.

## The format on its own terms

A FlatBuffers buffer is a self-contained byte sequence with the
following high-level structure: a fixed-size *root offset* at the
front pointing to the *root table*; the root table itself somewhere
inside the buffer; and any out-of-line data (strings, vectors,
embedded tables) elsewhere in the buffer, addressed by offsets from
their referencing tables. The buffer is read by following pointers
from the root, and the pointers are 32-bit unsigned offsets relative
to the location of the offset itself.

The central data structure is the *table*. A table corresponds to
what other formats call a struct or message: a named container of
fields. The wire layout of a table has two halves. The *data section*
holds the inline values (booleans, integers, floats, embedded
fixed-size structs) and offsets-to-out-of-line-data (for strings,
vectors, and embedded tables). The *vtable* is a small auxiliary
structure that records, for each declared field, the offset within
the data section where that field lives. Fields not present in a
particular instance simply have a zero offset in the vtable; the
reader checks the vtable, sees the zero, and reports the field as
absent.

The vtable is shared across instances of the same table type when
the field-presence pattern matches, which is a non-trivial
optimization for structures emitted in batches. The vtable starts
with two `int16` fields (vtable size in bytes; inline size of the
table's data in bytes) followed by an `int16` offset for each
declared field. The data section starts with an `int32` offset back
to its vtable, then the inline fields and the offsets-to-out-of-line.

Strings are stored elsewhere in the buffer. A string consists of a
4-byte length prefix, the UTF-8 bytes themselves, and a single null
terminator (the null is not counted in the length but is included
for C-string compatibility). The length prefix is at the offset
pointed to by the string's containing field; the bytes follow. A
string can be referenced from multiple places in the buffer if the
encoder chooses to deduplicate, though most encoders do not bother.

Vectors are length-prefixed arrays of values. A vector of scalars
holds the values inline; a vector of strings or tables holds offsets
to the elements. Vectors of structs (which are inline fixed-size
records, distinct from tables) hold the structs directly. The
length is a 4-byte prefix; the elements follow.

A *struct* in FlatBuffers terminology is *not* a table. A struct is
a fixed-layout, fixed-size, non-extensible record whose fields are
inlined directly wherever the struct is used. Structs are denser
than tables but cannot evolve: the layout is frozen at schema
write time, and any change to a struct's fields breaks every buffer
that contains it. Tables are the right choice for almost everything;
structs are an optimization for known-stable, performance-critical
fields where the per-field vtable lookup overhead matters.

Alignment is enforced throughout. Every value is aligned to its
natural boundary (4-byte fields on 4-byte boundaries, 8-byte fields
on 8-byte boundaries) within the buffer. The encoder inserts padding
bytes wherever necessary to maintain alignment. This is why
FlatBuffers buffers are larger than the equivalent Protobuf
encodings: the format pays in padding for the privilege of
direct-load-without-byteswap on aligned reads.

## Wire tour

Schema:

```fbs
table Person {
  id:uint64;
  name:string;
  email:string;
  birth_year:int32;
  tags:[string];
  active:bool;
}
root_type Person;
```

The buffer for our Person is harder to walk byte-by-byte than the
preceding formats because the layout depends on the encoder's
choices about ordering and alignment. The actual byte count is
approximately 132 bytes when the encoder is a typical implementation
(`flatc`'s output, or an equivalent runtime encoder). The structure,
described from the front of the buffer:

```
[ root offset (4 bytes): points to the root table ]
[ ... out-of-line data: strings, the tags vector ]
[ vtable for Person: 16 bytes ]
[ Person table: 32 bytes ]
[ end of buffer ]
```

The root offset is the first four bytes; reading those tells the
caller where the Person table is. Following that offset lands on
the Person table's first 4 bytes, which are an `int32` offset
backward to the vtable (the value is negative because the vtable is
typically before the table in the buffer). The vtable is read to
find the offsets of each field within the table.

Within the Person table, the field offsets resolve as follows: `id`
is an inlined uint64 (8 bytes); `name` is a 4-byte offset to the
"Ada Lovelace" string elsewhere in the buffer; `email` is a 4-byte
offset to the email string; `birth_year` is an inlined int32
(4 bytes); `tags` is a 4-byte offset to the tags vector;
`active` is an inlined byte (1 byte, padded to 4 for alignment).
The total inline size is approximately 32 bytes once padding is
accounted for.

The strings, stored out of line, take 17 bytes for "Ada Lovelace"
(4 bytes length + 12 bytes UTF-8 + 1 null + 0-3 padding) and 26
bytes for the email (4 + 21 + 1 + padding). The tags vector is a
4-byte length prefix and two 4-byte offsets (12 bytes), pointing
to "mathematician" (4 + 13 + 1 = 18 bytes plus padding) and
"programmer" (4 + 10 + 1 = 15 bytes plus padding).

Adding it all up: ~32 bytes for the table plus ~16 bytes for the
vtable plus ~17 + 26 + 12 + 18 + 15 ≈ 88 bytes for the out-of-line
data plus 4 bytes for the root offset and a few bytes of inter-
field padding gives a buffer in the 130-150 byte range, depending
on the encoder.

This is roughly twice the size of Protobuf and Thrift Compact.
The cost is the price for the access pattern. Reading
`person->id()` from a FlatBuffers buffer compiles, in C++, to:

```
*reinterpret_cast<const uint64_t*>(table_ptr + 4)
```

That is one load, no parsing, no allocation. There is no other
format in this book that achieves that property without similar
constraints.

If `email` were absent, the encoder would emit no string for it,
the vtable's email offset would be 0, and the table reader's
`email()` accessor would return null. The buffer would shrink by
the email string's bytes (about 26-28 bytes) without changing the
table's vtable layout or alignment. This is one of FlatBuffers'
genuine wins: optional fields cost nothing when absent, and the
absence is detectable at the cost of a single zero check in the
vtable.

## Evolution and compatibility

FlatBuffers' evolution rules are stricter than Protobuf's and
Thrift's, and the strictness is the price of zero-copy access.

The rule for tables is *fields can only be added at the end of the
schema*, and once added, their position is permanent. Adding a
field in the middle would shift the vtable layout and break every
existing buffer. Adding at the end is safe because old vtables
simply won't have the new field's slot, and old data won't have a
non-zero offset there. New consumers reading old data see the new
field as absent; old consumers reading new data ignore the new
slot in the vtable.

Removing a field is supported by marking it `deprecated` in the
schema; the field number stays in the vtable but the generated code
no longer exposes it. Old buffers continue to work; new buffers
omit the field's offset in the vtable.

Renaming a field is purely cosmetic at the buffer level (vtable
positions are what matters). The source code change is parallel to
Protobuf's: regenerate, redeploy.

Changing a field's type is mostly unsafe. The vtable assumes a
specific size for each field (computed from the type), and changing
the type changes the size, which corrupts the table layout. The
schema-language rule is to add a new field with the new type and
deprecate the old one.

Reordering field declarations *is* safe in FlatBuffers, because
the vtable assigns each field a fixed offset based on its declared
ID rather than its source order. This is unlike Avro and unlike
human intuition, but it's part of the FlatBuffers contract.

Structs cannot evolve at all. Any change to a struct's fields breaks
every buffer that contains it. The recommended pattern is to use
structs only for layouts you are confident will never change, and
to use tables for everything else.

The deterministic-encoding question for FlatBuffers is harder than
for the parse-required formats. The encoder has freedom in vtable
layout, in deduplication of strings, in ordering of out-of-line
data, and in padding choices. Most encoders are not deterministic.
A "force defaults" mode and a "minimum sizes" mode exist in the
reference encoder but do not produce canonical output across
languages. If you need byte-equality on FlatBuffers, you have to
either build a canonicalizer or accept that hash-stable bytes are
not on offer.

## Ecosystem reality

The reference implementation, `flatc`, generates code for C++, C#,
Go, Java, JavaScript, TypeScript, Lobster, Lua, PHP, Python, Rust,
Swift, and Dart. The C++ generator is the canonical one; the others
vary in maturity. The Rust crate (`flatbuffers`) is mature and
performance-aware; the Go and Java generators are good; the Python
generator is functional but the runtime is much slower than the
others (Python's lack of cheap pointer arithmetic is the
bottleneck).

TensorFlow Lite is the largest single deployment. Every TFLite model
is a FlatBuffers buffer with the schema defined in `schema.fbs` in
the TensorFlow source tree. Any tool that reads or writes TFLite
models is, implicitly, a FlatBuffers consumer. The choice of
FlatBuffers for TFLite was made for the same reasons it's been
chosen elsewhere — load latency and memory footprint on
constrained devices — and has held up well.

Cocos2d-x and Unity-adjacent toolchains use FlatBuffers for asset
manifests. Several internal Google services use FlatBuffers for
hot-path RPC; the public-facing services tend to use Protobuf for
the ecosystem reasons that argument always falls to. A handful of
financial trading systems use FlatBuffers for market data, sitting
between SBE (which is even faster but harder to use) and Protobuf
(which is easier but slower).

The most common ecosystem gotcha is that FlatBuffers buffers must
be aligned in memory when read. On modern x86 and ARM, unaligned
loads of common sizes are fine, and no one notices. On older
architectures, on certain SIMD code paths, and on some embedded
platforms, unaligned loads either crash or are dramatically slower.
The reference C++ runtime checks alignment in debug builds and
elides the check in release; bugs that bypass the runtime's
allocator (e.g., reading from a memory-mapped file at an unaligned
offset) can produce platform-specific failures that are hard to
diagnose. The FlatBuffers documentation is explicit about
alignment, but the failure mode is severe enough that it bites teams
new to the format.

The second gotcha is the tooling. Reading a FlatBuffers buffer
without the schema is hard. There are dump tools (`flatc` itself
can render a buffer back to JSON given the schema), but the
schemaless inspectability of MessagePack and CBOR is not on offer.
Operational tools have to know the schema, and the schema has to be
versioned alongside the buffer. This is the same operational cost
Protobuf imposes; FlatBuffers does not introduce a new problem
here, but the lack of a JSON-equivalent text format makes
ad-hoc inspection harder than Protobuf's `prototext`.

The third gotcha is build-system integration. `flatc` is a separate
binary that needs to run as part of the build, and the choices
about how (CMake, Bazel, Cargo build scripts, npm scripts) are
left to the user. Mixed-language projects often end up with two or
three flatc invocations producing parallel outputs. This is
manageable but is more friction than Protobuf's mature build
integrations.

## When to reach for it

FlatBuffers is the right choice when reads dominate writes, when
latency-per-read matters, and when the data structure can fit
within the format's constraints (immutable buffers, no cycles,
fields-added-at-end evolution). The canonical cases: game asset
loading, ML model loading, mmap'd file formats with random
access, RPC where deserialization cost is the bottleneck.

It is a defensible choice for any high-volume on-disk format where
parse-time would otherwise be the cost. TensorFlow Lite is the
example; similar choices have been made for graph databases,
spatial indexes, and a few message brokers' on-disk segment
formats.

## When not to

FlatBuffers is the wrong choice when writes are common, when the
data is consumed once and discarded, when wire size matters more
than read latency, when the schema is volatile (the
fields-only-at-end rule is real), or when human-readable text
inspection is required.

It is also the wrong choice when alignment-sensitive deployment is
unwelcome (some embedded platforms) and when the build-system
overhead of `flatc` is too much for the project size.

## Position on the seven axes

Schema-required. Not self-describing. Row-oriented. *Zero-copy*,
which is the format's defining axis position. Codegen, with no
serious runtime alternative. Non-deterministic by spec; canonical
forms achievable with effort. Evolution constrained: only-at-end
for tables, immutable for structs.

The cell FlatBuffers occupies — schema-required, zero-copy,
codegen-only, append-only evolution — is unique among the formats
in this book and is the strongest expression of the
zero-copy idea. Cap'n Proto, the next chapter, occupies a similar
cell with different tradeoffs.

## Epitaph

FlatBuffers is the format that asks "what if the parse step were
not there?" and answers in 130 bytes of aligned padding;
indispensable when reads dominate, expensive otherwise.
