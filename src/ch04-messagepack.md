# MessagePack

MessagePack is the format you would design if someone asked you to make
JSON smaller without changing what JSON is. That is approximately what
Sadayuki Furuhashi did in 2008, and the result has had the kind of
quiet, unglamorous success that the better engineering choices usually
have: shipped widely, criticized rarely, replaced by nothing.

## Origin

MessagePack came out of fluentd, the log aggregation daemon Furuhashi
was building at the time. Fluentd's wire format needed to be smaller
and faster to parse than JSON — log volumes were already large enough
that JSON's overhead mattered — but the data model needed to remain
compatible with JSON, because the upstream and downstream tools all
spoke JSON and were not going to change. The design constraint was
therefore not *what is the best binary format* but *what is the
smallest binary format that round-trips through JSON without losing
anything*. The answer, after some iteration, was MessagePack.

The format was published under an MIT-style license and a website at
msgpack.org collected implementations contributed by the community.
Implementations now exist in every language that anyone has bothered
to write a serializer in: C, C++, Java, JavaScript, Python, Ruby,
Go, Rust, Swift, Erlang, Elixir, Crystal, Zig, and roughly thirty
others. The format is unchanged in any way that matters since the
2013 specification revision; the small flurry of activity around the
"new spec vs. old spec" disagreement that year is now a footnote.

## The format on its own terms

MessagePack has nine families of value, corresponding closely to JSON's
six (null, bool, number, string, array, object) plus three additions
(integer separated from float, raw binary, and an extension mechanism
for application-defined types like timestamps). Every value on the
wire is preceded by a single tag byte that identifies its type and,
for short values, encodes the value or its length directly into the
low bits of the tag.

The high-density tag space is the trick. Small integers from 0 to 127
encode as a single byte where the byte is the integer (positive
fixint). Negative integers from -32 to -1 encode as a single byte
where the low five bits are the value (negative fixint). Strings up
to 31 bytes encode their length in the low five bits of a single
prefix byte (fixstr). Arrays up to 15 elements use a single prefix
byte for the count (fixarray). Maps up to 15 keys use a single prefix
byte for the count (fixmap). For anything larger than these inline
forms, MessagePack escalates to a multi-byte prefix that explicitly
declares the type and width: 0xcc through 0xcf for unsigned integers
of 8, 16, 32, 64 bits; 0xd0 through 0xd3 for signed; 0xd9 through
0xdb for strings whose length needs 8, 16, or 32 bits; 0xc4 through
0xc6 for raw binary; 0xdc and 0xdd for arrays; 0xde and 0xdf for
maps; 0xca and 0xcb for IEEE 754 floats and doubles.

The result is that small values pay almost no overhead — a byte for
the type and the value combined — and large values pay a few bytes
of length prefix. A typical mixed payload encodes to around half the
size of the equivalent JSON, sometimes less, depending on how
white-space-heavy the JSON was. Compression on top of MessagePack
yields further gains, but the format alone closes most of the gap.

The data model deliberately does not include Avro-style records,
Protobuf-style fields-by-tag, or anything that requires a schema. A
MessagePack map is a JSON object: keys are values (almost always
strings, though MessagePack permits any value as a key), the order is
not significant, and no field is "missing" — it is either present in
the map or it is not. This is the same data model as JSON. The
format is intentionally not richer than JSON.

There is one exception: the Extension type. An extension is a
type-tagged binary blob, where the type code (0 to 127 for
application-defined, -128 to -1 for spec-defined) tells the receiver
how to interpret the bytes. The spec defines a single extension type
itself, the timestamp (type code -1), which encodes 32 to 96 bits of
seconds and nanoseconds. Everything else is left to the application.
This is the seam through which higher-level type systems are bolted
on, and for the most part the seam holds.

## Wire tour

Encoding our Person record:

```
86                                           map of 6 entries
  a2 69 64                                   key "id"
  2a                                           value 42 (positive fixint)
  a4 6e 61 6d 65                             key "name"
  ac 41 64 61 20 4c 6f 76 65 6c 61 63 65     value "Ada Lovelace" (fixstr 12)
  a5 65 6d 61 69 6c                          key "email"
  b5 61 64 61 40 61 6e 61 6c 79 74 69 63
     61 6c 2e 65 6e 67 69 6e 65              value "ada@analytical.engine" (fixstr 21)
  aa 62 69 72 74 68 5f 79 65 61 72           key "birth_year"
  cd 07 17                                   value 1815 (uint16, big-endian)
  a4 74 61 67 73                             key "tags"
  92                                           array of 2 elements
    ad 6d 61 74 68 65 6d 61 74 69 63 69 61 6e   "mathematician" (fixstr 13)
    aa 70 72 6f 67 72 61 6d 6d 65 72            "programmer" (fixstr 10)
  a6 61 63 74 69 76 65                       key "active"
  c3                                           value true
```

The total is 104 bytes. The equivalent JSON, minified, is 124 bytes.
The difference is not large, and that is the point: MessagePack does
not save dramatic amounts of space on small payloads, because the
overhead of length tags and key strings dominates the encoding of the
values themselves. The wins come at scale, on payloads with many
small numbers or boolean flags, where each value's byte cost is
roughly halved.

Two observations from the bytes. First, the integer tag byte 0x2a is
the value 42 itself; no prefix, no length. This is the positive
fixint family at work. Second, the strings are length-prefixed but
not null-terminated, which is the consistent choice across all
binary formats with length prefixes; null-termination is a feature of
C strings and adds nothing once you have a length.

If `email` were absent, the encoding would simply omit both the key
bytes and the value bytes, and the map prefix would be 0x85 (fixmap
of 5) instead of 0x86. There is no marker for absence. The map
either contains the key or it does not. This is the same model JSON
uses, and it has the same consequence: distinguishing *absent* from
*null* requires the application to encode null explicitly when that
distinction matters.

## Evolution and compatibility

MessagePack inherits JSON's evolution story, which is to say it has
none, and the lack of one is the design. Adding a field means
emitting one more key in the map. Removing a field means not emitting
it. Renaming a field is a breaking change, the same as renaming a
JSON key. There are no field tags, no positions, no schema-resolved
defaults. If you want any of those, you build them at a higher
layer.

In practice this works because the consumers of MessagePack messages
are written with the same casualness JSON consumers are written
with: pull the keys you care about, ignore the rest, treat missing
keys as missing. The cost is that no automated tool can tell you
whether two versions of a producer remain compatible with two
versions of a consumer; you have to read the code. The benefit is
that nothing in the format requires coordination between producer
and consumer beyond the keys themselves.

Schema validation, if you want it, lives in libraries on top:
msgpack-schema for Python, various JSON Schema validators that
support MessagePack as an alternative input format, and the typed
deserializers in language ecosystems that have them (serde for
Rust, encoding/json-style decoders for Go, Jackson with the
msgpack-jackson module for Java). None of this is in the format.

The format itself has had two backward-compatible changes since the
original release. The 2013 spec revision added the str family
(distinct from raw bytes) and the bin family. Some early
implementations conflated strings and bytes; the new tags
disambiguated them. There is a "compat mode" flag in most
implementations that lets you produce either the old or the new
encoding, and this is occasionally still required when interoperating
with very old MessagePack readers. The 2017 timestamp extension
added a standardized timestamp encoding; it is namespaced under the
extension mechanism, so old readers see it as an unknown extension
and either skip it or fail loudly, depending on configuration. No
version of the spec has removed anything.

## Ecosystem reality

MessagePack's ecosystem is sprawling, mostly high-quality, and
occasionally surprising. The reference C implementation (msgpack-c)
is reasonably fast and reasonably featureful, but is not always the
state of the art in any given language. Modern competition includes
ormsgpack in Python (which is dramatically faster than the canonical
msgpack library by leaning on Rust), msgpack-cli in .NET, the
official msgpack-rust crate, and rmpv (a slightly different Rust
implementation focused on dynamic values). In JavaScript,
@msgpack/msgpack is the most common choice and is broadly fine.
In Java, both Jackson's msgpack module and the standalone msgpack-java
package are widely deployed; they produce identical bytes but expose
different APIs.

Notable adoptions: Redis used MessagePack internally for its
script-replication mechanism for years (the bytes of script
arguments crossed a MessagePack-shaped pipe before reaching the
Lua VM). Pinterest's engineering blog described using MessagePack
extensively in their search indexing pipeline. Influx's protocol
options at one point included MessagePack alongside line protocol.
Many internal RPC systems at companies that rejected gRPC for
some reason settled on MessagePack-over-HTTP as the next-most-
reasonable choice. None of these uses are particularly visible
because MessagePack does not advertise itself the way Protobuf does;
it is a piece of plumbing that ends up in stacks because someone
benchmarked it once and was satisfied.

The interoperability story is good but not perfect. A specific
gotcha: integer types. MessagePack's wire format encodes integers
in the smallest representation that fits, which means the same
value can encode in different widths from different producers. A
naive Python decoder will produce a Python int regardless of the
wire width, which is fine. A typed decoder in a language that
distinguishes uint32 from uint64 will sometimes have to decide
which to produce, and the heuristics differ between libraries.
For most workloads this does not matter; for a few it produces
silently wrong types. The fix is to use a typed schema layer on
top, at which point you have built half of Avro and may want to
reconsider your format choice.

A second gotcha: map key ordering. MessagePack does not specify
that maps preserve insertion order, and most libraries do not.
For deterministic output, sort the keys before encoding, or use a
library that supports a deterministic-encoding mode. Several do
not. This is the source of the small but persistent stream of
bug reports against msgpack-related signing and content-addressing
schemes.

A third gotcha: the str-vs-bin distinction. Some older code, and
some newer code that imitates older code, treats the str and bin
families as interchangeable. They are not, and decoders that
reject the wrong family will break ingestion. The compat-mode flag
is worth knowing about specifically because of this.

## When to reach for it

MessagePack is the right choice when you would otherwise use JSON
and you care about size or parse speed enough that the cost is
worth a binary format. The classic case: high-volume internal RPC
in a polyglot environment where you control both sides, the schema
changes constantly, and the operational ergonomics of "use a typed
schema language" are not worth the cost. MessagePack gives you
JSON's schemaless flexibility, JSON's data model, and roughly half
the bytes.

It is also the right choice for binary blobs in places where you
want the field-level structure to remain inspectable: a Redis
value, a Kafka message body, a queue payload. Tools exist that can
pretty-print MessagePack from any of those, and the inspectability
is approximately as good as JSON's once you have the tool installed.

## When not to

When you have a stable schema that you control on both ends, the
density advantage MessagePack offers over JSON is smaller than the
density advantage Protobuf or Cap'n Proto offer over MessagePack.
At that point you are paying for schemalessness you do not benefit
from. Switch to a schema-first format.

When you need any of: deterministic encoding without library-
specific support, formal schema evolution, language-level type
safety, zero-copy access, columnar layout, or guarantees about
what unknown fields do, MessagePack does not give them to you, and
the workarounds (sort keys yourself, validate at the boundary,
use a typed wrapper, accept full parses, use rows, hope for the
best) are all things you have to remember. The format will not
remind you.

When the consumers are public clients and you do not control them,
MessagePack imposes the parser-distribution problem that all
binary formats impose. Public APIs almost always end up with JSON
for this reason. MessagePack-over-the-public-internet exists, and
some popular APIs (some game backends, some IoT services) use it,
but the choice is an organizational one, not a technical one.

## Position on the seven axes

Schemaless. Self-describing. Row-oriented. Parse rather than zero-copy.
Runtime bindings, with no codegen step (and no codegen tools, because
there is nothing to generate from). Non-deterministic by default,
canonical encoding possible if you sort keys and use minimum-width
integers but no library guarantees this without configuration. No
evolution strategy: keys are strings, types are tags, the application
deals with the rest.

This stance is unusual for a binary format only in the sense that most
binary formats have abandoned the schemaless side of the schema axis.
Picking MessagePack is a deliberate decision to remain on that side
while still spending bytes more efficiently than JSON does. Picking
MessagePack and then trying to bolt schemas on top produces something
worse than picking Avro or Protobuf to begin with — the bolt-on lacks
the formal evolution rules, the wire-level optimizations, and the
codegen ergonomics, while paying most of the operational cost of
having a schema at all.

The corollary is that *the cases where MessagePack is the right
choice are exactly the cases where its stance on the seven axes
matches what your system actually needs*. If you find yourself
wishing for any of schemas, deterministic encoding, zero-copy
access, columnar layout, or compatibility checking, MessagePack is
not what you want, and the version of MessagePack you build by
adding those things one at a time will be worse than picking the
right format from the start. This is, in my experience, the
single most common way to misuse MessagePack: choosing it because
it sounds like "a better JSON," and then discovering, two years in,
that what you actually wanted was Protobuf.

## A note on the timestamp extension

A footnote on the timestamp extension is worth including, because
it is the one place where MessagePack's spec actually defines a
non-trivial type beyond the JSON model. The extension uses extension
type code -1 and supports three widths: 32 bits (seconds since the
epoch, no nanoseconds, valid through 2106), 64 bits (32 bits of
nanoseconds plus 34 bits of seconds, valid past 2514), and 96 bits
(32 bits of nanoseconds plus 64 bits of seconds, valid for any
timestamp representable in twos-complement). The encoding is
straightforward and is supported by most of the major libraries,
but not all of them, and not always by default. If your data
includes timestamps and you want them to round-trip across language
boundaries, check that both sides understand the extension. If they
do not, the fallback is usually to encode timestamps as ISO-8601
strings or as Unix epoch integers — both of which work, both of
which lose some fidelity, and both of which sidestep the extension
mechanism entirely.

The existence of the timestamp extension is the strongest evidence
that MessagePack's authors knew the format would be used for more
than just JSON-equivalent payloads, and the deliberate restraint
in not adding more extensions (no UUID, no decimal, no big-int) is
the strongest evidence that they wanted the format to remain small
in surface area. The trade has held up; no widely-deployed system
has, to my knowledge, complained that MessagePack's type system is
too small to be useful, and the few that need richer types reach
for CBOR or Ion instead.

## Epitaph

MessagePack is JSON's binary doppelgänger: same data model, half
the bytes, none of the schema. Reach for it when the only thing
wrong with JSON is the size.
