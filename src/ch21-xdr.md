# XDR

XDR is the format that runs the Network File System and, less
expectedly, the Stellar blockchain network. It is one of the
oldest binary serialization standards still in active use,
predating Protobuf by a decade and a half, and its design choices
— fixed-width big-endian fields with 4-byte alignment, positional
encoding without field tags, no built-in schema evolution
mechanism — feel austere by modern standards. The austerity is
deliberate: XDR was built for a world where the cost of CPU
cycles to encode and decode was the bottleneck, not the cost of
bytes on the wire. That world is mostly gone, and yet XDR
persists in the places where it was good enough.

## Origin

XDR was specified by Sun Microsystems in RFC 1014 in 1987,
revised as RFC 4506 in 2006, and has been stable since. The
motivation was Sun's own RPC system (later called ONC RPC), which
needed a wire format for the arguments and return values of remote
procedure calls. The constraints were straightforward: fast to
encode and decode on the workstation hardware of the late 1980s
(SPARC, MC68000, VAX), portable across the architectures Sun and
its customers used, and simple enough that engineers could
implement it correctly.

XDR fell out of those constraints with surprisingly little
friction. Every value is fixed-width, big-endian, and aligned to
a 4-byte boundary. The schema language describes types in C-like
syntax. The encoder is a few hundred lines of C. The decoder
similarly. The wire format is exactly what you would get if you
wrote out C struct fields one at a time with explicit padding —
which, in fact, is approximately what most XDR encoders do.

ONC RPC and XDR became the foundation of NFS, Sun's network file
system, and through NFS XDR achieved a deployment scale that was
substantial throughout the 1990s and remains substantial today.
NFSv3 and NFSv4 both use XDR for their wire formats; every NFS
server and client in the world parses XDR. The format also became
the basis for several other Sun-era protocols (NIS, the Network
Information Service; rwall; rusers; a half-dozen others) that
have mostly faded.

The unexpected modern revival of XDR came from the Stellar
network and its smart-contract platform Soroban. The Stellar
team chose XDR for their wire format in 2014, on the reasoning
that it was simple, deterministic, well-specified, and had
mature implementations in the languages they cared about. The
choice was contrarian — the rest of the blockchain world was
gravitating toward Protobuf or custom binary formats — and it has
held up. Stellar's transactions, ledger entries, and contract
calls are all XDR-encoded.

## The format on its own terms

XDR's data model has primitive types and constructed types, and
the rules for each are short.

The primitive types: integer (32-bit signed, big-endian),
unsigned integer (32-bit unsigned, big-endian), hyper (64-bit
signed), unsigned hyper (64-bit unsigned), float (IEEE 754
single, big-endian), double (IEEE 754 double, big-endian),
quadruple (IEEE 754 quad, used rarely), and boolean (encoded as
a 32-bit integer with value 0 or 1). Every primitive is
fixed-width; there is no varint encoding, no zigzag, no compact
form. A 32-bit integer always takes 4 bytes; a 64-bit integer
always takes 8.

The constructed types: enumerations (encoded as their underlying
integer); structures (encoded as the concatenation of their
fields); fixed-length arrays (encoded as the concatenation of
their elements); variable-length arrays (encoded as a 4-byte
length followed by the elements, padded to a 4-byte boundary at
the end); fixed-length opaque (raw bytes, padded to a 4-byte
boundary); variable-length opaque (4-byte length plus bytes plus
padding); strings (variable-length opaque, with UTF-8 by
convention but technically uninterpreted); optional data (a
4-byte boolean indicating presence, followed by the value if
present); discriminated unions (a tag value followed by the
variant's encoding).

Every variable-length value pays for alignment to a 4-byte
boundary. A string of 5 bytes encodes as 4 bytes of length, 5
bytes of content, and 3 bytes of zero padding — 12 bytes for a
5-byte string. Strings of length 4n encode in 4n+4 bytes (length
plus content with no padding). The padding is bandwidth that is
spent on alignment, but the alignment makes encoding and
decoding trivial: every field starts at a known offset, and the
decoder advances by exactly the field's encoded size.

There is no schema-in-bytes representation. The schema is the
XDR file (a text artifact, similar to a Protobuf `.proto`), and
both ends are expected to have it. The wire bytes are
uninterpretable without the schema.

## Wire tour

Schema:

```xdr
struct Person {
    unsigned hyper id;
    string name<>;
    string *email;
    int birth_year;
    string tags<>[];
    bool active;
};
```

The `string<>` notation declares a variable-length string. The
`*` declares an optional. The `string<>[]` declares a variable-
length array of variable-length strings. Encoded:

```
00 00 00 00 00 00 00 2a                     id: 8 bytes (BE u64) = 42
00 00 00 0c                                  name length: 12
41 64 61 20 4c 6f 76 65 6c 61 63 65          "Ada Lovelace" (no padding needed, 12 % 4 == 0)
00 00 00 01                                  email present (boolean true)
00 00 00 15                                  email length: 21
61 64 61 40 61 6e 61 6c 79 74 69 63
   61 6c 2e 65 6e 67 69 6e 65 00 00 00      "ada@analytical.engine" + 3 bytes padding
00 00 07 17                                  birth_year: 4 bytes (BE i32) = 1815
00 00 00 02                                  tags count: 2
00 00 00 0d                                  tags[0] length: 13
6d 61 74 68 65 6d 61 74 69 63 69 61 6e
   00 00 00                                  "mathematician" + 3 bytes padding
00 00 00 0a                                  tags[1] length: 10
70 72 6f 67 72 61 6d 6d 65 72 00 00         "programmer" + 2 bytes padding
00 00 00 01                                  active: 4 bytes (BE u32) = 1 (true)
```

104 bytes. Larger than every modern variable-length encoding —
about 50% larger than Protobuf, 60% larger than Avro — and the
overhead is essentially all alignment padding plus fixed-width
integers. The `id` field alone takes 8 bytes; in Protobuf it took
2; in Avro it took 1. The `active` boolean takes 4 bytes; in
MessagePack it took 1.

What XDR pays for the bytes is read latency. Every field is at a
known offset within the buffer once the variable-length fields
before it have been walked. There is no varint decoding, no
big-endian-to-host-byte-swap on most architectures (modern x86
and ARM both have native byteswap instructions, but the ones from
1987 didn't, which is why XDR's choice of big-endian was a real
cost on Sun's own SPARC hardware), no length-prefix scanning for
fixed-width fields. The decoder is trivial. The encoder is
trivial. The format is trivial. *Trivial* is the design.

If `email` were absent, the encoding would be:

```
... (id, name as before)
00 00 00 00                                  email present: false
... (birth_year, tags, active)
```

The optional field is a 4-byte presence flag plus, if present,
the value. Saving 28 bytes when absent (the 4 bytes of length plus
21 bytes of UTF-8 plus 3 bytes of padding equals 28 bytes that
disappear when email is null). The presence flag itself remains.

## Evolution and compatibility

XDR has no formal evolution mechanism. The schema is fixed; the
wire bytes are fixed; changes to the schema break compatibility
with old bytes unless the change is "additive at the end of a
struct" — and even then, the format does not require old
decoders to skip unknown trailing bytes, so the convention of
adding-at-the-end is enforced by social agreement, not by the
spec.

The conventional approaches to XDR schema evolution are:

- *Add fields by versioning the entire message.* Define a new
  type, and have the producer emit either the old or the new type
  based on what the consumer expects. The consumer dispatches on
  some discriminator (often part of the protocol envelope, not
  XDR itself).
- *Use discriminated unions.* Define a union type whose variants
  represent different versions of the message. New versions add
  new variants; old consumers ignore variants they don't
  recognize (after some discriminator parsing).
- *Tolerate trailing bytes.* Some consumers are written to read
  exactly the fields they expect and to ignore anything after; new
  versions can append fields and continue to work. This is
  fragile and is not part of the spec.

In practice, XDR-using systems handle evolution at the protocol
layer, not at the format layer. NFSv3 and NFSv4 are different
protocols with different XDR schemas; they do not share wire
format compatibility, and both ends of a connection negotiate
which protocol to use. The Stellar network does similar versioning:
the protocol version is part of the envelope, and major changes
mean a new protocol version with a new XDR schema.

The deterministic-encoding question for XDR is unambiguous: the
format is fully deterministic. Given a schema and a value, there
is exactly one byte sequence that encodes it. The fixed-width
representation, the absence of length-encoding choices, and the
mandated padding all produce a unique encoding. XDR bytes are
hashable and signable without canonicalization.

## Ecosystem reality

XDR's ecosystem is bimodal: extremely mature in the Sun-RPC and
NFS lineage, freshly active in the Stellar lineage, and almost
nonexistent elsewhere.

The Sun-RPC tooling includes `rpcgen`, the canonical XDR-to-C
compiler, which has been distributed with every Unix-like operating
system since the late 1980s. ONC RPC implementations exist for
every language that has been used for systems programming in the
last 35 years. NFS implementations on every operating system
include XDR. The format is wire-stable across decades; an NFS
client from 2005 talks to an NFS server from 2025 (assuming both
agree on protocol version), with the XDR encoding being the
unproblematic layer.

The Stellar tooling is more modern: TypeScript, Rust, and Go
implementations of XDR maintained by the Stellar Foundation, with
codegen tools that produce idiomatic types in each language. The
Stellar XDR schema is a substantial document — hundreds of types
defining the network's data model — and the codegen pipelines are
heavy enough to be a routine concern for Stellar developers. The
choice of XDR has held up well; Stellar's transactions are
deterministic, the wire format is stable across protocol versions,
and the implementations are interoperable.

Outside these two communities, XDR is rare. A few legacy
enterprise systems still use it. Some scientific computing
systems use it for data exchange. None of these communities are
growing.

The most consequential ecosystem gotcha is *the gap between XDR
and modern protocol design.* New protocols built today rarely
benefit from XDR's strengths and are penalized by its weaknesses.
The lack of schema evolution, the wire-size overhead, and the
lack of modern tooling like Buf's breaking-change detection make
XDR a high-friction choice. The Stellar team's decision to use it
was made before some of the modern alternatives existed and has
been preserved by inertia and by the value of stability in a
financial network.

## When to reach for it

XDR is the right choice when interoperating with NFS, Sun RPC, or
the Stellar/Soroban ecosystem. It is the right choice for
deterministic encoding requirements where the alternative would
be a custom hand-rolled format and where a mature spec is preferable.

It is a defensible choice when extreme stability across decades is
the binding constraint and the schema-evolution constraints are
manageable.

## When not to

XDR is the wrong choice for new general-purpose typed binary
serialization. Protobuf, Avro, and CBOR all do the job with
better wire density, better evolution stories, and better tooling.

It is the wrong choice when bytes-on-the-wire matter; XDR's
fixed-width fields and 4-byte alignment cost real bandwidth on
realistic payloads.

## Position on the seven axes

Schema-required. Not self-describing. Row-oriented. Parse rather
than zero-copy, although the fixed-width discipline makes parsing
nearly trivial. Codegen-first via rpcgen and similar tools.
Fully deterministic by spec. Evolution by social convention; no
in-format mechanism.

XDR's stance on the axes is the simplest of any format in this
book: the spec is short, the choices are uniform, and the
trade-offs are transparent. The price of the simplicity is that
modern formats have improved on every axis where XDR is paying a
cost — except determinism, where XDR remains as good as any.

## A note on Sun RPC's adjacent influence

It is worth a brief detour into Sun RPC, because the format
inherits much of XDR's character and because Sun RPC is the
ancestor of every RPC framework that came after.

Sun RPC was designed alongside XDR — they are sibling specifications,
both from 1987 — and defined a complete RPC protocol stack: program
numbers, procedure numbers, version numbers, authentication
flavors, and a wire format that wrapped XDR with framing and
metadata. The protocol was simple and effective. Every Unix
operating system shipped a `portmap` daemon to track which RPC
services were running on which ports, and `rpcinfo` was a routine
tool for service discovery.

The lineage of modern RPC frameworks runs through Sun RPC. CORBA
(designed in the early 1990s) was a more elaborate response to
the same problem, with object references and type system
features that Sun RPC lacked. DCE RPC, Java RMI, and eventually
gRPC and Thrift all owe direct architectural debts to Sun RPC.
The fact that gRPC's designers studied Sun RPC carefully is not
an academic detail; the choice to make program identifiers stable
across versions, to support multiple authentication flavors, and
to keep the wire framing simple all trace back through XDR to
Sun's original design.

The reason Sun RPC didn't itself become the dominant RPC
framework is mostly the security model: Sun's authentication
flavors were rudimentary, and the protocol shipped with `AUTH_NONE`
as a viable option. NFSv3, in particular, had famously weak
authentication, which mostly worked because the deployments were
behind LAN firewalls. As networks became more open, Sun RPC's
security insufficiency became disqualifying for new deployments,
and the protocols that succeeded it (CORBA's IIOP, gRPC over
HTTP/2 with TLS) all centered authentication and transport
security in ways Sun RPC did not.

XDR survives Sun RPC's decline because it can be used outside the
RPC framework. XDR-as-a-format is a viable independent choice,
even when XDR-as-part-of-an-RPC-protocol has been displaced. The
Stellar example is one such use; the NFSv4 example, where XDR is
still in production but as part of a more modern protocol stack,
is another.

## Epitaph

XDR is the format that ran NFS and now runs Stellar; austere,
fully deterministic, and exactly as evolved as it needs to be —
which is to say, not very.
