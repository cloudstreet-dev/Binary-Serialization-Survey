# Borsh and SCALE

Borsh and SCALE are the formats blockchain protocols pick when they
need a deterministic binary serialization that produces the same
bytes for the same value across implementations and across versions
of the runtime. Both come from the second-generation blockchain
ecosystem, both emerged in roughly the same time frame (2018-2020),
and both took the same general approach: fixed-width little-endian
encoding, no schema-in-bytes, deterministic by construction. They
differ in their length-encoding strategies and in their schema
languages, but the cell of the design space they occupy is the
same. Reading them together is the right way to understand both,
because the differences highlight choices each format made about
the same questions.

## Origin

Borsh — the name is a backronym for *Binary Object Representation
Serializer for Hashing* — was created by the NEAR Protocol team in
2019 as the canonical serialization format for the NEAR blockchain.
NEAR's smart contract platform needed a serialization that
produced bytes equal across producers (so that contract authors
could hash state and signatures could be verified across nodes)
and that worked from Rust, since contracts on NEAR compile to
WebAssembly from Rust. Borsh was the answer: a serde-compatible
Rust library plus a small spec defining the wire format. The
spec is short, the implementation is small, and the design
choices are conservative.

SCALE — *Simple Concatenated Aggregate Little-Endian Encoding* —
was created by the Parity Technologies team for the Substrate
blockchain framework, which underlies Polkadot, Kusama, and a
broader ecosystem of substrate-based chains. SCALE's motivation
was the same as Borsh's: a deterministic serialization for the
runtime, the wire format, and the on-chain state. Substrate is
also a Rust framework, which means SCALE is also a serde-adjacent
Rust library; the spec is platform-independent, but the
canonical implementation lives in the `parity-scale-codec` crate.

The two formats overlap in their intended use case to a degree
that, in retrospect, suggests one of them might not exist if the
two communities had been more aware of each other early on.
Borsh's design was finalized roughly contemporaneously with
SCALE's, and the choices each made — small but real differences in
how lengths are encoded, how options are tagged, how strings are
handled — diverge enough that the formats are not interchangeable.
Both are now well-established within their respective ecosystems,
and the duplication is permanent.

## The format on its own terms

Borsh's encoding rules are straightforward. Primitive integers are
encoded as their natural width in little-endian byte order: u8 is
one byte, i32 is four bytes little-endian, u64 is eight bytes
little-endian. Booleans are one byte (0 or 1). Floating point uses
IEEE 754 little-endian. Arrays of fixed-known size encode as the
concatenation of their elements with no length prefix. Variable-
length collections (`Vec<T>`, `String`, `HashMap`) encode as a
4-byte little-endian length prefix followed by their elements.
`Option<T>` encodes as a one-byte discriminant (0 for None, 1 for
Some) followed by the value if Some. `Result<T, E>` encodes as a
discriminant byte (0 for Ok, 1 for Err) followed by the variant.
Structs encode as the concatenation of their fields in declaration
order. Enums encode as a one-byte discriminant followed by the
variant's payload.

The schema is the Rust source code; the format relies on serde-
derive-style macros to produce serializers and deserializers from
struct and enum definitions. There is no separate IDL.
Cross-language Borsh implementations exist for JavaScript,
TypeScript, Python, Go, and a few others, but the canonical
schema is the Rust struct.

SCALE's encoding rules are similar with one substantial
difference: lengths use *compact encoding*, a variable-width
integer encoding designed to minimize bytes for small values. A
compact integer's low two bits indicate the encoding mode: mode 0
(low bits `00`) is a 6-bit value packed into the remaining bits
of a single byte (values 0-63); mode 1 (`01`) is a 14-bit value in
two bytes (values 64-16383); mode 2 (`10`) is a 30-bit value in
four bytes (values 16384-(2^30 - 1)); mode 3 (`11`) is a big-int
mode where the next byte gives the byte count of a little-endian
integer (values larger than 2^30). The compact encoding is used
for the lengths of `Vec<T>` and `String`, for the count of
elements in collections, and in a few other places where the spec
calls for it.

SCALE also differs from Borsh in `Option` encoding for booleans:
`Option<bool>` is encoded as a single byte with three possible
values (0 = None, 1 = Some(false), 2 = Some(true)), where Borsh
would use two bytes (one for the discriminant, one for the bool).
This optimization saves a byte for the common option-of-bool case.

Both formats share the lack of a schema in the bytes. The
producer and consumer must agree on the schema out of band; the
format provides no metadata for type recovery. Both formats are
strict about their declared types: a u32 is always exactly four
bytes, with no varint compression of small values. This rigidity
is what produces the determinism: there are no encoding choices,
no width selection, no padding options.

The schema language for Borsh is documented at borsh.io and is
expressed via Rust's type system. The schema language for SCALE
is documented in the Substrate documentation and uses a slightly
different vocabulary — a "compact" field type, "Vec<T>" and
"BoundedVec", "BTreeMap", and so on — but the underlying
correspondence is the same.

## Wire tour

Borsh schema (Rust):

```rust
#[derive(BorshSerialize, BorshDeserialize)]
struct Person {
    id: u64,
    name: String,
    email: Option<String>,
    birth_year: i32,
    tags: Vec<String>,
    active: bool,
}
```

Encoded:

```
2a 00 00 00 00 00 00 00                     id: u64 LE = 42
0c 00 00 00                                  name length: u32 LE = 12
41 64 61 20 4c 6f 76 65 6c 61 63 65          "Ada Lovelace"
01                                           email Option discriminant: Some
15 00 00 00                                  email length: u32 LE = 21
61 64 61 40 61 6e 61 6c 79 74 69 63
   61 6c 2e 65 6e 67 69 6e 65                "ada@analytical.engine"
17 07 00 00                                  birth_year: i32 LE = 1815
02 00 00 00                                  tags count: u32 LE = 2
0d 00 00 00                                  tags[0] length: 13
6d 61 74 68 65 6d 61 74 69 63 69 61 6e       "mathematician"
0a 00 00 00                                  tags[1] length: 10
70 72 6f 67 72 61 6d 6d 65 72                "programmer"
01                                           active: bool = true
```

90 bytes. The fixed-width lengths (4 bytes each for every variable-
length quantity) account for most of the difference between Borsh
and the varint-using formats. Five lengths (name, email, tags
count, tags[0], tags[1]) take 20 bytes total in Borsh; Protobuf's
varint encoding of the same lengths would take 5 bytes.

SCALE schema (Rust, with `parity-scale-codec` derive):

```rust
#[derive(Encode, Decode)]
struct Person {
    id: u64,
    name: String,
    email: Option<String>,
    birth_year: i32,
    tags: Vec<String>,
    active: bool,
}
```

Encoded:

```
2a 00 00 00 00 00 00 00                     id: u64 LE = 42
30                                           name compact length: 12 (mode 0, value << 2 | 0 = 48 = 0x30)
41 64 61 20 4c 6f 76 65 6c 61 63 65          "Ada Lovelace"
01                                           email Option: Some
54                                           email compact length: 21 (21 << 2 = 84 = 0x54)
61 64 61 40 61 6e 61 6c 79 74 69 63
   61 6c 2e 65 6e 67 69 6e 65                "ada@analytical.engine"
17 07 00 00                                  birth_year: i32 LE = 1815
08                                           tags compact count: 2
34                                           tags[0] compact length: 13 (13 << 2 = 52 = 0x34)
6d 61 74 68 65 6d 61 74 69 63 69 61 6e       "mathematician"
28                                           tags[1] compact length: 10 (10 << 2 = 40 = 0x28)
70 72 6f 67 72 61 6d 6d 65 72                "programmer"
01                                           active: bool = true
```

75 bytes. The compact encoding shrinks the length prefixes from 4
bytes each to 1 byte each (since all our lengths fit in 6 bits).
Five 1-byte compacts replace five 4-byte fixed lengths, saving 15
bytes overall.

If `email` were absent, Borsh would replace the `01` discriminant
with `00` and skip the value entirely, saving 26 bytes (the
1-byte length-of-length-of-email plus the email itself). SCALE
would do the same, saving 23 bytes.

## Evolution and compatibility

Both formats have the same answer to the schema-evolution question,
which is essentially: don't. The wire format is positional and
rigid; the schema cannot change without breaking every consumer
that has the old schema; new fields cannot be added without a
coordinated upgrade.

In practice, both ecosystems handle evolution at the protocol level
rather than the format level. NEAR's smart contract upgrades are
versioned by the runtime; SCALE-using chains version their state
schema by spec_version (a runtime metadata field) and migrate state
at upgrade time. The format is not asked to handle skew; the
runtime is.

This is a clean separation of concerns and is appropriate for the
deployment context. Blockchains have *atomic* upgrade points (block
heights at which the runtime changes); they do not have the
heterogeneous-deployment problem that Protobuf and Avro were
designed for. The lack of in-format evolution is therefore not the
cost it would be in a microservices context.

The deterministic-encoding question is the entire reason these
formats exist. Both are fully deterministic by spec: given a
schema and a value, exactly one byte sequence is produced. Floats
encode their bit representation directly (NaN bit patterns are
preserved). Maps that have unstable iteration order in the source
language must be encoded with sorted keys (Borsh's spec mandates
this for `HashMap`; SCALE's `BTreeMap` is sorted by Rust's
construction). The bytes are hashable, signable, and comparable
byte-for-byte without canonicalization.

## Ecosystem reality

Borsh's ecosystem is concentrated in NEAR and the broader Solana
adjacent universe; Solana itself uses Borsh for many of its
SPL programs and tooling, despite its own native serialization being
something different. The reference Rust implementation is the
canonical one; JavaScript and TypeScript implementations are
mature; Python and Go implementations exist and are used by
off-chain tools. The format spec is short and stable.

SCALE's ecosystem is concentrated in Substrate and Polkadot. The
canonical implementation is `parity-scale-codec` in Rust; the
JavaScript implementation `@polkadot/types` is used by the Polkadot
JS API and is feature-complete. Implementations in Python, Go, C++,
and Java exist for various Substrate-adjacent uses. The format
spec is part of the Substrate documentation and is also stable.

Outside their respective blockchain ecosystems, neither format has
significant adoption. There is no good general-purpose case for
choosing Borsh or SCALE over Protobuf or CBOR; the determinism
guarantee is real but is not unique (CBOR's deterministic encoding
is comparably strong), and the lack of evolution support is a
disadvantage in the contexts where most binary serialization
happens.

The most consequential ecosystem gotcha is *the schema-source
question*. In a Rust-only deployment, the schema is the source
code, and changes to a struct's fields propagate through the type
system. In a multi-language deployment (Rust contracts plus
JavaScript clients, say), the schema lives in two places and must
be kept in sync. Both ecosystems have tooling to mitigate this:
Borsh has a schema-export tool that generates a JSON description
of a Rust type's serialization layout, which client libraries can
consume; SCALE relies on Substrate's metadata system to publish a
runtime's type definitions for JavaScript clients to deserialize.
Both tools work; both are points of friction.

## When to reach for them

Borsh is the right choice for NEAR contracts and Solana SPL
programs. SCALE is the right choice for Substrate-based chains.
Both are reasonable choices for any system that needs deterministic
binary serialization with a well-specified format and a Rust-first
implementation, and that does not need in-format schema evolution.

For new general-purpose use cases, either is a defensible choice
when the alternative would be a hand-rolled binary format and the
team values the existing libraries. CBOR with deterministic
encoding is the more obvious general-purpose alternative, but
Borsh and SCALE are simpler in some ways: smaller spec, less
optionality, faster to implement.

## When not to

Neither is the right choice when in-format schema evolution is a
hard requirement. Neither is the right choice when bytes-on-the-wire
matter and the alternative is a varint format like Protobuf or
CBOR (Borsh in particular pays substantial bytes for fixed-width
length prefixes). Neither is the right choice when the
cross-language ecosystem outside Rust matters; both have
implementations in other languages, but the implementations are
secondary.

For non-blockchain use cases where the appeal is "deterministic
binary format," CBOR with the deterministic encoding profile is
typically the stronger choice — better tooling, more languages,
broader ecosystem.

## Position on the seven axes

Schema-required. Not self-describing. Row-oriented. Parse rather
than zero-copy. Codegen via Rust's derive macros, with runtime
fallbacks for dynamic schemas. Fully deterministic by spec. No
in-format evolution mechanism.

The cell Borsh and SCALE occupy — schema-required, fixed-width-
little-endian, fully-deterministic, no-evolution — is a coherent
choice for the workloads they target and a poor choice for almost
anything else. The fact that two such similar formats exist is an
ecosystem accident, not a sign that the cell is naturally divided
in two.

## A note on the broader blockchain-format landscape

It is worth situating Borsh and SCALE in the broader landscape of
binary formats in blockchain protocols, because the
determinism-and-no-evolution choice these formats make is shared by
a number of others.

Bitcoin's serialization is its own format — a custom little-endian
binary encoding with VarInt-style length prefixes and explicit
versioning at the protocol level. The bytes are deterministic; the
format is unspecified outside Bitcoin's source code. Ethereum uses
RLP (Recursive Length Prefix), which is simpler than Borsh but
similar in spirit: fixed encoding rules, deterministic output,
positional layout. Cosmos chains use a Protobuf-derived format
called Amino (now mostly replaced by direct Protobuf with strict
encoding rules); Tendermint's wire format is Protobuf-shaped.
Stellar uses XDR (covered in chapter 21).

The pattern across blockchain formats is that *determinism is
non-negotiable* and *evolution happens at the runtime/protocol
level, not the format level*. The formats that fit this pattern
have remarkably similar shapes: fixed-width fields, no in-band
schema, deterministic encoding rules. The handful of formats that
tried to bring richer schema features (Avro, Protobuf with
deterministic mode) into blockchain contexts had to disable or
work around the features that don't survive the determinism
requirement.

Borsh and SCALE are therefore not idiosyncratic; they are
representatives of a coherent format family that emerged from a
specific deployment context. The family's design choices are
driven by the consensus mechanism, not by general serialization
preferences.

## A note on canonical bytes vs. canonical structure

One subtlety worth flagging is the difference between
*canonical bytes* (the format produces the same bytes for the same
logical value) and *canonical structure* (the format normalizes
the value before encoding). Borsh and SCALE produce canonical
bytes given a value; they do not normalize the value first. If
the source-language type is a `HashMap` whose iteration order is
unstable, the encoder must sort keys before emitting, which is
the producer's responsibility, not the format's. SCALE's
documentation is explicit about this; Borsh's is implicit and
relies on the Rust types (specifically, on using `BTreeMap`
rather than `HashMap` when ordering matters).

This is the place most often where determinism-by-spec fails in
practice: the format produces bytes from a value, but the value
itself was constructed with unspecified ordering, and so two
producers that "look" like they produced the same value produce
different bytes. The fix is always at the application layer — sort
inputs, use ordered collections, normalize before encoding. It is
not a flaw of the format; it is a consequence of the format being
honest about what it can and cannot guarantee.

## Epitaph

Borsh and SCALE are blockchain-flavored deterministic binary
formats: rigid by design, fast to implement, perfectly suited to
the on-chain consensus contexts they were built for, and rarely
the right choice anywhere else.
