# SBE

SBE — *Simple Binary Encoding* — is the format for the case where
microseconds matter and the cost of the rest of the format ecosystem
is worth paying. It is the dominant format in low-latency financial
trading, and most of the systems that use it would describe their
choice as obvious. To the rest of the world, SBE looks austere,
overspecified, and mildly hostile to ergonomics. Both reactions are
correct. SBE is what you get when you optimize for one thing —
nanosecond-scale encode and decode — at the explicit expense of
everything else, and the systems for which that one thing is the
binding constraint are happy to pay.

## Origin

SBE was designed by Martin Thompson and the team at Real Logic
around 2014, after Thompson had spent several years working on the
LMAX Disruptor and adjacent low-latency systems. The format was
explicitly built for the FIX Trading Community, the standards body
behind the FIX protocol that powers most of the world's electronic
trading. FIX is a text-based protocol designed in the early 1990s,
and by 2014 the latency cost of parsing FIX messages had become a
limiting factor for the fastest market participants. SBE was
proposed as the binary successor — *FIX/SP1*, in the standard's
nomenclature — and was adopted by the major exchanges and trading
venues over the following years.

The design constraints were specific and unusual. The format had to
be fast enough that encoding and decoding were essentially free in
the hot path: a few CPU cycles per field, with no allocations, no
branches that could mispredict, and no cache misses outside the
loaded message buffer. It had to be backward-compatible across
versions in the way that financial-data formats need to be (a
trade booked on Monday must be readable on Friday after the schema
has changed twice). It had to be implementable in any of the
languages traders use (C++, Java, Rust, with C# and Python for
the analytics tier), with consistent wire-level behavior. And it
had to be auditable, because every byte of every message is
potentially regulator-relevant.

The result is a format that looks more like a memory layout
specification than a serialization spec. Reading and writing SBE
messages, in C++ or Java, is equivalent to indexing into a fixed
struct, and that is the point.

## The format on its own terms

SBE messages have three layout regions, in order: a *fixed-size
block* of scalar fields, *variable-length data* fields, and
*repeating groups*. The fixed block is laid out at compile time
according to the schema, with each field at a known byte offset,
and accessed with literal struct-style memory references. The
variable-length data fields and repeating groups follow the
fixed block, in declaration order, with their own framing.

The schema language is XML — a choice that looks dated but is
defensible given that XML is the lingua franca of the FIX
community. The schema declares the types (composite types,
enums, sets), the messages (each with a fixed-block layout and
optional variable-length and group sections), and the metadata
(schema ID, version, byte order, header types). The schema
compiler generates accessor classes that wrap a raw byte buffer
and provide typed methods: `getId()` reads the eight bytes at the
declared offset and reinterprets them as a uint64; `setId(value)`
writes the value at the same offset.

Fields in the fixed block have explicit byte offsets, which can
be implicit (the compiler computes them in declaration order with
appropriate alignment) or explicit (the schema overrides). Fields
have *presence* attributes (`required`, `optional`, `constant`).
Optional fields are encoded by sentinel values — a special "null
value" defined for each type, indistinguishable on the wire from
a value happening to equal the sentinel — which is a constraint
inherited from the financial-data world's preference for
fixed-width records.

Variable-length data fields are encoded as a length prefix
(typically `uint16` or `uint32`, defined by a composite type in
the schema) followed by the bytes. They live after the fixed
block, in the order they were declared.

Repeating groups are SBE's mechanism for arrays of records. A
group has a *dimension* prefix (typically a composite of
`blockLength` and `numInGroup`) and then `numInGroup` entries,
each consisting of the group's own fixed block followed by the
group's variable-length data. Groups can nest, but the nesting
discipline is strict.

The message itself is preceded by a *message header*, which is a
small composite type (defined in the schema, but commonly 8 bytes)
giving the block length, template ID (which message type), schema
ID, and schema version. The header is what lets a generic
dispatcher route a buffer to the correct decoder.

The byte order of every field is specified at the schema level —
typically little-endian for x86 deployments, big-endian for some
exchange-specific deployments. The schema's byte order is part of
the format's identity; bytes are not portable across schemas with
different byte orders without an explicit conversion.

## Wire tour

Schema (abbreviated):

```xml
<sbe:messageSchema id="1" version="1" byteOrder="littleEndian">
  <types>
    <composite name="messageHeader">
      <type name="blockLength" primitiveType="uint16"/>
      <type name="templateId" primitiveType="uint16"/>
      <type name="schemaId" primitiveType="uint16"/>
      <type name="version" primitiveType="uint16"/>
    </composite>
    <composite name="varStringEncoding">
      <type name="length" primitiveType="uint16"/>
      <type name="varData" primitiveType="uint8" length="0"/>
    </composite>
    <composite name="groupSizeEncoding">
      <type name="blockLength" primitiveType="uint16"/>
      <type name="numInGroup" primitiveType="uint16"/>
    </composite>
  </types>

  <sbe:message name="Person" id="1" blockLength="16">
    <field name="id"         id="1" type="uint64" offset="0"/>
    <field name="birthYear"  id="2" type="int32"  offset="8"/>
    <field name="active"     id="3" type="uint8"  offset="12"/>
    <data  name="name"       id="10" type="varStringEncoding"/>
    <data  name="email"      id="11" type="varStringEncoding"
                                     presence="optional"/>
    <group name="tags"       id="20" dimensionType="groupSizeEncoding">
      <data name="value"     id="21" type="varStringEncoding"/>
    </group>
  </sbe:message>
</sbe:messageSchema>
```

Encoded:

```
10 00 01 00 01 00 01 00                 message header: blockLength=16, templateId=1, schemaId=1, version=1
2a 00 00 00 00 00 00 00                 id = 42 (LE u64)
17 07 00 00                             birthYear = 1815 (LE i32)
01                                      active = 1 (u8)
00 00 00                                3 bytes padding (block is 16 bytes)
0c 00                                   name length = 12 (LE u16)
41 64 61 20 4c 6f 76 65 6c 61 63 65     "Ada Lovelace"
15 00                                   email length = 21 (LE u16)
61 64 61 40 61 6e 61 6c 79 74 69 63
   61 6c 2e 65 6e 67 69 6e 65           "ada@analytical.engine"
00 00 02 00                             tags group dim: blockLength=0, numInGroup=2 (LE u16 pair)
0d 00                                   tag[0] length = 13
6d 61 74 68 65 6d 61 74 69 63 69 61 6e  "mathematician"
0a 00                                   tag[1] length = 10
70 72 6f 67 72 61 6d 6d 65 72           "programmer"
```

92 bytes. Slightly larger than Protobuf or Avro for this payload,
slightly smaller than FlatBuffers and Cap'n Proto. The size is not
the headline; the access pattern is. To read `id` from this buffer,
the C++ code generated by the SBE compiler does, approximately:

```c++
return *reinterpret_cast<const uint64_t*>(buffer + 8);
```

Eight is the offset of `id` after the 8-byte message header. The
read is a single load. There is no parsing, no length check, no
dispatch on type. The same is true of `birthYear` (offset 16) and
`active` (offset 20). Variable-length and group fields require
walking past the fixed block, but the walk is also straightforward:
read the length prefix, advance, repeat.

The optional `email` field encodes the empty string when absent
(length 0, no bytes). Distinguishing "email is empty" from "email is
absent" requires a side channel; SBE's `presence="optional"` on a
data field is not a presence flag in the bytes, only a schema-level
hint that consumers may treat the empty case specially.

If `email` were absent (encoded as zero-length), the bytes would
shrink by 23 bytes, and the encoded total would be about 69 bytes.

## Evolution and compatibility

SBE's evolution rules are the strictest in this book and the
strictest by design. The format makes a deliberate, sharp
distinction between *backward-compatible* changes (which can be
made to a schema in place) and *breaking* changes (which require
a new schema version and a coordinated rollout).

The backward-compatible changes are:

- Adding a new field at the *end* of the fixed block, provided the
  new schema's `blockLength` is updated. Old consumers see the
  same `blockLength` they expect and read only the fields they
  know about; new consumers see the larger `blockLength` and read
  the new field at the new offset.
- Adding a new variable-length data field at the end of the
  variable-length section.
- Adding a new repeating group at the end of the groups section.
- Adding new symbols to an enum (with care; the new symbols won't
  appear in old data).
- Adding fields *to* a repeating group's block, with the same
  rules as for the message-level block.

The breaking changes are: anything that changes the byte offsets of
existing fields, anything that changes the blockLength implicitly
(without an explicit version bump), anything that reorders fields,
anything that changes a field's type, and anything that removes a
field. Breaking changes require a new template ID or a new schema
version; consumers must check the message header and route to the
appropriate decoder.

The strictness is deliberate. SBE was designed for systems where
schema changes are rare, audited, and carefully coordinated. The
format does not try to make schema evolution graceful; it tries to
make sure that schema changes break loudly when they break, so
that no producer or consumer is silently misinterpreting bytes.

The deterministic-encoding question for SBE is trivial: the format
is fully deterministic. Given a schema and a value, there is
exactly one byte sequence that encodes it. This is a consequence of
the fixed-offset layout, the absence of variable-width integer
encoding, and the lack of optional padding. SBE bytes are
hashable, signable, and comparable byte-for-byte without
canonicalization.

## Ecosystem reality

SBE's primary ecosystem is FIX/SP1 and the broader low-latency
trading community. The reference implementation, *Real Logic SBE*,
is open source and maintained on GitHub under the agrona
organization. It generates code for Java, C++, C#, Rust, and Go.
The generated code is high-quality, low-allocation, and has been
audited extensively by the financial firms that use it.

The CME Group, ICE, Eurex, NASDAQ, and most other major
electronic exchanges publish their market-data and order-entry
schemas in SBE. A trading firm wanting to consume those feeds
generates the appropriate code from the published schemas and
links it into their trading system. The wire-level details of how
each exchange uses SBE differ in small ways (header conventions,
optional-field sentinel values, group-prefix sizes), but the
format itself is uniform.

Outside finance, SBE is rare. There are a few uses in hardware
trading systems, a few in low-latency networking research, a few
in academic teaching of binary formats. Aeron, the
high-throughput messaging library by the same Real Logic team,
uses SBE for its internal message types and is the most-visible
non-financial deployment.

The most common ecosystem gotcha is the mismatch between SBE's
mental model and what most engineers expect from a serialization
format. SBE is not meant to be a generic typed binary protocol; it
is meant to be a memory-layout specification. Engineers who try to
use SBE as a Protobuf replacement find the schema language clunky,
the evolution rules austere, and the ergonomics of optional fields
hostile. They are not wrong; SBE is not for them.

The second gotcha is that the schema XML is not always exchanged
between trading partners; sometimes a partner will publish a PDF
specification that you must convert by hand into an SBE schema.
The PDF and the schema are supposed to agree, but the version
control story for that agreement is uneven. New SBE adopters
benefit from running their generated code against published
sample messages before going live.

## When to reach for it

SBE is the right choice when latency dominates every other
concern and the problem domain has fixed-width fields, well-defined
message templates, and a manageable number of message types. The
canonical case is electronic trading. Adjacent cases include
hardware-in-the-loop simulation, high-throughput sensor pipelines,
and any embedded system where the schema is stable and the access
patterns are time-critical.

It is a defensible choice for any system where the access pattern
is overwhelmingly read, the schema is highly stable, and the
benefits of zero-cost field access outweigh the schema-evolution
constraints.

## When not to

SBE is the wrong choice for almost any other workload. Generic
typed binary serialization (use Protobuf). Schema-evolving
systems (use Avro or Protobuf). Systems where latency is not the
binding constraint (any of the others). Systems where ergonomics
matter (anything but SBE).

It is also the wrong choice when the language is one SBE does not
support well; the long tail of language bindings is thin compared
to Protobuf's.

## Position on the seven axes

Schema-required. Not self-describing (the bytes plus the message
header tell a generic dispatcher *which schema*; the schema is
still mandatory to decode the body). Row-oriented. *Zero-copy*,
in the strictest sense available in this book — the bytes
literally are the in-memory layout. Codegen-only. Fully
deterministic by spec. Evolution by strict append-at-the-end with
explicit schema version bumps for anything else.

The cell SBE occupies — schema-required, fixed-offset,
fully-deterministic, append-only evolution — is the strictest
possible expression of the zero-copy idea, and it is the right
expression for the workloads it was designed for.

## A note on the FIX/FAST predecessor

Before SBE, the FIX Trading Community standardized FIX/FAST, a
binary encoding of FIX messages designed to compress the textual
form aggressively at the cost of complex stateful encoders and
decoders. FAST used template-based delta encoding: each field in a
message could be declared as encoded relative to the previous
message's value of the same field, or as a constant, or as a
copy-from-previous, or as several other operators. The result was
extremely small messages — sometimes a quarter the size of FIX
text, and competitive with SBE on size — at the cost of an
encoder/decoder state machine that was hard to implement
correctly and harder to debug.

FAST was deployed at several major exchanges through the 2000s and
into the 2010s. Most have migrated off FAST to SBE. The reasons
were operational: FAST's stateful encoding meant that a dropped
packet on the multicast feed could desynchronize the decoder, with
nontrivial recovery; FAST's complexity meant that a small number
of vendors dominated the implementation space, with the resulting
licensing concerns; and the latency-of-decode advantage of SBE's
zero-copy approach was decisive once 10 GbE networking made the
size advantage of FAST less critical.

The reason this matters for an SBE chapter is that SBE inherited
its constituency directly from FAST, and the design decisions in
SBE — fixed-offset layout, sentinel-encoded optionality, strict
schema-versioning — are partly reactions against the operational
costs of FAST. SBE traded size for simplicity, and in the
particular community SBE was built for, the trade was the right one.

## Epitaph

SBE is the format for nanoseconds-matter, schemas-rarely-change,
hash-the-bytes deployments; austere, audited, and the unspoken
default of electronic trading.
