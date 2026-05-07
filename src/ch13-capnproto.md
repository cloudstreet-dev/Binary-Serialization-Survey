# Cap'n Proto

Cap'n Proto and FlatBuffers are siblings. Both are zero-copy. Both
were built by people with substantial prior experience designing wire
formats. Both occupy the same approximate cell in the format design
space. The differences between them are small in cardinality and
considered in detail; they are exactly the kind of differences that
illustrate why design choices that look similar from far away
diverge sharply when you have to live with them. Cap'n Proto is the
zero-copy format with a more thoroughgoing theory and a smaller
ecosystem. Both facts are consequences of the same person making the
same design choices.

## Origin

Cap'n Proto was created by Kenton Varda, who had spent five years on
the Protobuf team at Google before leaving to build Sandstorm, a
self-hostable web application platform. Cap'n Proto's origin
narrative is unusually candid: Varda's blog post announcing the
format in 2013 framed it as "Protocol Buffers, in less time," with
"less time" referring not to development effort but to wall-clock
time spent encoding and decoding. The argument was that Protobuf's
design choices, made before the rise of mmap-able persistent storage
and high-bandwidth in-memory data exchange, paid for serialization
overhead the modern systems they were being deployed in did not
need to pay. The argument was substantive and correct.

The format has been stable since 2014 in its 1.x line, with a small
number of additive changes (capability tables, more streaming
operations, the `pack`-encoding mode that re-introduces a
post-process step in exchange for size). The reference
implementation is in C++, with a high-quality Rust binding
(`capnp` and `capnpc-rust`) and a Java port. Other languages have
ports of varying maturity. Sandstorm itself, while now a smaller
project than it was at its peak, was the original primary user and
shipped the format to enough independent developers to seed the
ecosystem.

Outside Sandstorm, Cap'n Proto's most visible deployment is at
Cloudflare, which uses it as the control-plane format for the
Cloudflare Workers product and several internal systems. The Cap'n
Proto RPC protocol — distinct from the serialization format,
covered briefly below — underpins much of Cloudflare's distributed
systems work. The Sandstorm-era concept of *capability-based RPC*,
where references to remote objects can be passed as first-class
values, was adopted by Cloudflare's architecture and has become a
significant differentiator between Cap'n Proto's RPC and gRPC's.

## The format on its own terms

A Cap'n Proto message is a sequence of *segments*, each of which is
a contiguous block of *words*. A word is exactly eight bytes; this
is fundamental to the format and is non-negotiable. Everything is
word-aligned, and most data structures are word-sized or
word-multiple. The frame at the front of a message specifies the
segment count (minus one) and the size of each segment in words,
and the segments follow.

The single most important data structure in Cap'n Proto is the
*pointer*, which is a one-word value that references another
location in the message. There are four pointer types: struct
pointers (referring to a struct), list pointers (referring to a
list), capability pointers (referring to a capability table entry,
used by the RPC layer), and far pointers (referring to an entry in
another segment). The low bits of a pointer encode the type; the
remaining bits encode an offset (relative to the location of the
pointer itself) and metadata describing the pointed-to value's
shape.

A *struct* in Cap'n Proto is a fixed-layout record with two
sections: a *data section* of scalars and a *pointer section* of
references to out-of-line data. The data section is laid out by
field size, with 8-byte fields first, then 4-byte, 2-byte, 1-byte,
and finally bit-packed booleans. The pointer section follows,
with one pointer per pointer-typed field, in declaration order.
The struct's layout is determined by its schema; the wire format
has no per-instance metadata about which fields are present.

This is the central design difference between Cap'n Proto and
FlatBuffers. FlatBuffers stores per-instance metadata (the vtable)
that records which fields are actually present in each instance.
Cap'n Proto stores no such metadata: every instance has all fields,
and absence is encoded by zero values (a zero pointer means the
referenced field is null/empty; a zero scalar means the scalar is
its default). The consequence is that Cap'n Proto's structs are
denser than FlatBuffers' tables on average — there is no vtable
overhead per instance — but absence and default values are
indistinguishable for scalar fields, mirroring (and predating, in
fact) the Protobuf 3 design choice that produced years of pain.

A *list* is a length-prefixed sequence of values. The header for a
list (encoded in the pointer that references it) specifies the
element count and the element size. Elements can be 0-bit
(empty/void), 1-bit (booleans), 1-byte, 2-byte, 4-byte, 8-byte,
pointer, or composite (variable-size, with a tag word at the front
specifying the element layout). Lists of structs use the composite
encoding; the tag word at the start of the list lets readers know
the per-element data and pointer section sizes.

Text and Data are special cases of List(UInt8). Text is required to
be valid UTF-8 and includes a null terminator (which is not counted
in the length); Data has no encoding restrictions. Both are stored
as pointers from their referencing struct to a separately-located
list elsewhere in the same segment.

## Wire tour

Schema:

```capnp
@0xb59b1b1d4f1c1234;

struct Person {
  id        @0 :UInt64;
  name      @1 :Text;
  email     @2 :Text;
  birthYear @3 :Int32;
  tags      @4 :List(Text);
  active    @5 :Bool;
}
```

A simplified word-by-word view of the encoded message (each line
is one 8-byte word, addresses on the left in word units):

```
00: 00 00 00 00 12 00 00 00         frame: 0 segments-minus-1, 18 words in segment 0
01: 00 00 00 00 02 00 03 00         root pointer: struct, 2 data words, 3 pointer words
02: 2a 00 00 00 00 00 00 00           struct.data[0]: id = 42
03: 17 07 00 00 01 00 00 00           struct.data[1]: birth_year=1815, active=true (low bit), padding
04: 0d 00 00 00 62 00 00 00           struct.ptr[0]: list pointer for "name"
05: 19 00 00 00 b2 00 00 00           struct.ptr[1]: list pointer for "email"
06: 21 00 00 00 17 00 00 00           struct.ptr[2]: list pointer for "tags"
07: 41 64 61 20 4c 6f 76 65            "Ada Lo"
08: 6c 61 63 65 00 00 00 00            "lace\0" plus padding (Text payload)
09: 61 64 61 40 61 6e 61 6c            "ada@anal"
0a: 79 74 69 63 61 6c 2e 65            "ytical.e"
0b: 6e 67 69 6e 65 00 00 00            "ngine\0" plus padding
0c: 02 00 00 00 02 00 00 00            tags list: 2 elements of pointer type
0d: 09 00 00 00 72 00 00 00            tags[0] pointer
0e: 11 00 00 00 5a 00 00 00            tags[1] pointer
0f: 6d 61 74 68 65 6d 61 74            "mathemat"
10: 69 63 69 61 6e 00 00 00            "ician\0" plus padding
11: 70 72 6f 67 72 61 6d 6d            "programm"
12: 65 72 00 00 00 00 00 00            "er\0" plus padding
```

Total: 144 bytes (8 bytes of frame header + 18 words of segment).
This is comparable to FlatBuffers' size for the same record. The
overhead comes from the same place: alignment padding for
zero-copy access. Cap'n Proto's word-aligned discipline means even
small fields like `active` (one bit) consume a full slot in the
data section's bit-packed area.

Two structural points are worth noting in the bytes. First, the
absence of any per-instance metadata about field presence: every
field has a slot, and absent fields have zero values in their
slots. Second, the locality: the struct's data is in the words
immediately following its pointer, and the strings it references
are in the words immediately after the struct. This locality is
intentional and matters for cache performance; encoders that scatter
out-of-line data across the buffer pay measurable performance costs
on real workloads.

If `email` were absent, the encoder would emit a null pointer in
the email slot (a zero word) and would not allocate space for the
email string. The buffer would shrink by the email string's bytes
(24 bytes plus the pointer's slot is unchanged, since the pointer
is part of the fixed struct layout). The total size would drop to
about 120 bytes. Distinguishing absent from empty for a Text field
is straightforward: the pointer is null vs. pointing to a zero-
length list. For scalar fields, however, absence is encoded as the
default value, which is the trap.

## The packed encoding

Cap'n Proto includes a *packed* encoding mode that compresses the
canonical encoding by collapsing runs of zero bytes. The encoding
is simple: for each 8-byte block, emit a tag byte where each bit
indicates whether the corresponding byte is zero, followed by the
non-zero bytes. The result is roughly 60-80% of the canonical size
for typical schemas, at the cost of a per-message decompression
step. The packed encoding is *not* zero-copy — reading it requires
decompression to a normal buffer first — and so it sacrifices the
format's defining feature in exchange for size. The choice between
canonical and packed is made per use case.

## Evolution and compatibility

Cap'n Proto's evolution rules are looser than FlatBuffers' and
closer to Protobuf's, with caveats specific to the wire format.

Adding a new field is supported by appending it to the schema with
the next available `@N` ordinal. The schema compiler computes the
new struct's data and pointer section sizes; new structs have
larger sections, but old buffers (with smaller sections) are
readable because the schema compiler tracks the section sizes per
schema version, and readers know to interpret old buffers using
the old layout. Pointer fields can be added in the same way; old
data has a smaller pointer section, and the new pointer field is
absent (null) when reading.

Removing a field is supported by marking it `removed`; the schema
compiler reserves the slot but exposes nothing to the generated
code. Reusing a slot is forbidden, with the same severity as
reusing a Protobuf field number.

Renaming a field is purely cosmetic. Field IDs (the `@N` ordinals)
are wire-level identity.

Changing a field's type is mostly unsafe. Numeric types of the
same size are wire-compatible; types of different sizes shift
the data section layout and corrupt old buffers. Text and Data are
wire-compatible (both are List(UInt8)). Pointer-typed fields
cannot change to scalar-typed fields and vice versa.

The aspect of evolution where Cap'n Proto is meaningfully more
flexible than FlatBuffers is *struct extension*. A Cap'n Proto
struct can be evolved by appending fields (new ordinals) without
breaking old buffers, because the schema-compiler-tracked section
sizes let readers handle the smaller-old-data case. FlatBuffers'
vtable mechanism does the same, more dynamically. The two formats
arrive at a similar evolution model from different starting points.

The deterministic-encoding question for Cap'n Proto is partially
addressed by the *canonical encoding* mode in the spec, which
defines a unique byte representation for any given message: all
out-of-line data follows its referencing struct in a specific
order, default values are not emitted, and trailing zero data
words are stripped. Most encoders do not produce canonical
encoding by default; the option is opt-in. With it, byte-equality
is achievable. Without it, equivalent messages can produce
different bytes.

## Cap'n Proto RPC

Cap'n Proto includes a full RPC protocol whose key feature is
*promise pipelining*: a client can issue a call against the
return value of an earlier call without waiting for the earlier
call to complete. The first call's response is referenced by a
capability ID, and the second call uses that ID directly. The
result is dramatic latency reduction for chained calls, especially
across high-latency links. This feature has no analogue in gRPC
(though gRPC's streaming covers some of the same use cases) and
is the primary reason Cloudflare's distributed systems
infrastructure uses Cap'n Proto rather than Protobuf.

The RPC layer is a separate concern from the serialization format.
You can use Cap'n Proto's serialization without its RPC, and many
deployments do. The RPC layer's adoption is narrower than the
serialization's because few systems benefit enough from promise
pipelining to justify the operational difference from gRPC.

## Ecosystem reality

Cap'n Proto's ecosystem is smaller than Protobuf's by an order of
magnitude and smaller than FlatBuffers' by a smaller but
significant margin. The C++ implementation is high-quality and
maintained. The Rust crate is mature and idiomatic; capnproto-rust
is the most-used non-C++ binding by some margin. The Java port
exists but lags. Other languages (Python, Go, JavaScript) have
ports that are functional but not first-class.

The user base is concentrated. Sandstorm and Sandstorm-derivative
projects use it. Cloudflare uses it heavily. A few research-grade
distributed systems and a handful of blockchain-adjacent projects
have adopted it. There is not much else in production. This is the
opposite of the Protobuf situation, where every modern stack has
some Protobuf in it; Cap'n Proto is the format you choose
deliberately, with awareness that the language support outside C++
and Rust may be thin.

The most common ecosystem gotcha is the assumption that Cap'n Proto
will replace Protobuf in the same role. It will not. The wire
formats are different, the schema languages are different, and the
ecosystem maturity is different. Cap'n Proto's strengths are
specific (zero-copy, RPC pipelining, a slightly more thoughtful
evolution model); choosing it for general-purpose typed
serialization without those specific needs is choosing a smaller
ecosystem for negligible gain.

## When to reach for it

Cap'n Proto is the right choice when the workload benefits from
zero-copy access *and* you are working in C++ or Rust *and* the
RPC pipelining feature is either useful or irrelevant. The classic
cases: high-throughput RPC on internal systems, mmap-able persistent
formats, distributed systems where latency dominates throughput.

It is the right choice for systems that genuinely benefit from
capability-based RPC, which is a smaller set than commonly
assumed.

## When not to

Cap'n Proto is the wrong choice when language support is required
beyond C++ and Rust; the bindings exist but are thinner. It is the
wrong choice when the target audience for the format is broad and
includes teams unfamiliar with zero-copy formats; the operational
learning curve is real. It is the wrong choice when wire size
matters more than read latency (use Protobuf; or accept the
packed-encoding tradeoff).

It is also the wrong choice when the schema is human-edited
configuration; the lack of a text projection comparable to
Protobuf's `prototext` is a real ergonomic gap.

## Position on the seven axes

Schema-required. Not self-describing (the bytes are
schema-dependent). Row-oriented. *Zero-copy*. Codegen, with no
serious runtime alternative. Canonical encoding mode available;
non-canonical by default. Evolution by ordinal-tagged fields with
schema-tracked section sizes.

Cap'n Proto's stance differs from FlatBuffers' on two axes worth
naming. First, the absence of per-instance vtables: Cap'n Proto
saves bytes per instance but loses the ability to distinguish
absent from default for scalar fields. Second, the canonical
encoding subset, which FlatBuffers does not formally specify; this
makes Cap'n Proto more usable in signing and content-addressing
contexts.

## Epitaph

Cap'n Proto is the format Kenton Varda built when he decided his
old team had taken Protobuf as far as the architecture they
shipped against would allow; word-aligned, capability-aware, and
loved by the people who use it.
