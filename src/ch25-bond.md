# Bond

Bond is the format Microsoft built when it had concluded — like
Google before it and Facebook contemporaneously — that it needed a
typed schema-first wire format for inter-service traffic. Bond
arrived after Protobuf and Thrift were already established, and
its design choices reflect that arrival. Bond's authors had the
benefit of seeing what worked and what did not in its predecessors
and made small but considered refinements: richer schema
expressiveness, multiple wire encodings selectable per use case, a
"bonded" type for pass-through serialization that lets a service
forward a message without knowing its full schema. The format has
been used at substantial scale inside Microsoft for over a decade.
The format's gradual displacement by Protobuf inside Microsoft, and
its near-invisibility outside, are the reasons it is in this book
rather than fighting Protobuf for the chapter on schema-first wire
formats.

## Origin

Bond was developed at Microsoft and open-sourced in 2015. Its
internal use predated the public release by several years; Bing,
the search engine, used Bond for inter-service traffic, as did
the Cosmos data warehouse (the internal Microsoft system, not the
Azure offering of similar name) and several other large
properties. The team behind Bond was led by Adam Sapek and
included engineers with substantial experience in distributed
systems serialization.

The design goals of Bond were a slight reframing of Protobuf's:
support typed schemas with formal evolution rules, support
multiple wire encodings (because different use cases want
different tradeoffs between size and parse speed), support
forward-compatible "bonded" fields that let services route or
forward messages without parsing them fully, and support a richer
type system than Protobuf's, including non-nullable references,
nullable wrappers, and various container types.

The format has been stable since the public release. Microsoft's
internal use continued through 2024 in the systems that adopted
it early, but new services inside Microsoft increasingly chose
Protobuf, and the Bond GitHub repository's commit history shows
the slow tempo of a project transitioning from active development
to maintenance. The reference implementations (C++, C#, Python,
Java) are still maintained, but the broader Microsoft engineering
center of gravity has moved to gRPC and Protobuf.

Outside Microsoft, Bond has had minimal adoption. A handful of
external projects use it, mostly in cases where a Bond-using
Microsoft codebase was open-sourced and the dependency came along.
The format is not on the radar of any organization choosing a
serialization format from scratch in 2026.

## The format on its own terms

Bond's schema language is similar to Protobuf's and Thrift's, with
some additions:

```bond
namespace example;

struct Person {
    0: required uint64 id;
    1: required string name;
    2: nullable<string> email;
    3: required int32 birth_year;
    4: required vector<string> tags;
    5: required bool active;
}
```

Field ordinals (the `0:`, `1:`, etc.) play the same wire-level role
as Protobuf's field numbers and Thrift's field IDs: they are the
stable identifier for the field across schema versions. Field
declarations have presence modifiers — `required`, `optional`, or
implicit (which is similar to Protobuf 3's "default" semantics) —
and can use Bond's richer type modifiers: `nullable<T>` for an
explicitly nullable wrapper, `bonded<T>` for the pass-through
type, `vector<T>` and `list<T>` for ordered collections, `set<T>`
for unordered, `map<K, V>` for keyed lookups.

Bond supports a handful of types beyond what Protobuf and Thrift
offer in their core specs: `decimal` (a fixed-point decimal type),
`blob` (binary blobs as a distinct type from byte vectors), and
`bonded<T>` (described below).

The `bonded<T>` type is Bond's most distinctive contribution. A
`bonded<T>` field carries a serialized T as opaque bytes, with
just enough metadata to identify its type and version. A service
can read the bonded field's metadata, decide whether to fully
parse it, forward it as bytes to another service, or store it
without ever decoding it. This is useful for routing services
that need to look at a few fields of a message and pass the rest
through; with a bonded field, the routing service does not need
to depend on the schema for the inner type. Protobuf has
something similar in its `Any` type, but `bonded<T>` is more
explicit about what the inner type is and is generally easier to
work with.

Bond defines several wire encodings:

**Compact Binary** is the default: tag-and-length-prefixed,
varint-encoded integers, similar in shape to Thrift Compact and
Protobuf. This is the encoding used most often.

**Fast Binary** is a faster-to-encode-and-decode variant that
trades some size for performance: explicit width fields, less
varint encoding. The "fast" name is relative; Compact Binary on
modern hardware is rarely the bottleneck.

**Simple Binary** is a minimal encoding that omits some metadata
and is suitable for fixed-schema scenarios where evolution is not
required.

**Simple JSON** is a JSON-shaped encoding for debugging and
interop with non-Bond systems. Bond also has an XML encoding for
the same role.

The encoding is selected at runtime by the service; a single
schema can be deserialized from any of the encodings, and a
service can mix encodings on different topics if appropriate.

## Wire tour

Encoding our Person record with Compact Binary:

```
3a 00                                        struct header (compact binary)
   c0 2a                                     field 0 (id), uint64, value 42
   c1 0c 41 64 61 20 4c 6f 76 65 6c 61 63 65 field 1 (name), string len 12
   c2 15 61 64 61 40 61 6e 61 6c 79 74 69 63
        61 6c 2e 65 6e 67 69 6e 65            field 2 (email), nullable string,
                                              has-value flag plus string len 21
   c3 ae 1c                                  field 3 (birth_year), int32, zigzag(1815)
   c4 02                                       field 4 (tags), vector<string>, count 2
        0d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e
        0a 70 72 6f 67 72 61 6d 6d 65 72
   c5 01                                     field 5 (active), bool
00                                           struct terminator
```

Total approximately 75 bytes. Comparable to Thrift Compact (which
made similar tag-encoding choices) and Avro for this payload. The
breakdown is comparable to Thrift Compact's: field tags take 1-2
bytes, varint integers take their natural width, length-prefixed
strings dominate the size.

The exact bytes here are approximate; Bond's Compact Binary has
specific framing details (the leading struct header, the encoding
of the field tag with type information, the terminator) that
differ slightly from Thrift Compact, but the sizes work out
within a few bytes of each other. Bond's field-tag encoding packs
the field ordinal and the type information into a single byte for
small ordinals and types, with a multi-byte fallback for larger
values.

If `email` were the null branch of `nullable<string>`, the
encoding would emit a single byte indicating absence (0x00 in the
nullable wrapper), saving the 22 bytes of the string. Field 2's
tag would still be present.

## Evolution and compatibility

Bond's evolution rules are similar to Protobuf's. Adding a field
with a new ordinal is forward and backward compatible. Removing a
field requires the ordinal to be retired (Bond does not have
`reserved` syntactically, but the convention is to never reuse
ordinals). Changing a field's type is mostly unsafe, with similar
exceptions to Protobuf's (numeric promotions, nullable to
non-nullable when the value is always present). Renaming is
source-only; ordinals are wire-level identity.

Bond's `required` keyword has the same operational hazard
Thrift's has: removing a required field is a coordinated
deployment, and required fields are conventionally avoided
unless absence is genuinely a fatal error.

Bond's `bonded<T>` type adds an interesting evolution wrinkle. A
service that holds bonded data does not need the schema for the
inner type; it can forward the bonded bytes to another service
that does have the schema. This means a schema change to the
inner type only affects services that fully parse it; routing or
forwarding services are unaffected. This is a real operational
benefit at scale, and it is one of the reasons Bing's
architecture used Bond for as long as it did.

The deterministic-encoding question for Bond is the same as for
Protobuf: not specified at the wire level, achievable with care.
Bond does not have a canonical encoding subset. Map ordering,
varint widths, and a few other choices are encoder-specific.
Applications that need byte-equality canonicalize separately.

## Ecosystem reality

Bond's ecosystem is small. The reference implementations are in
C++, C#, Python, and Java; all four are mature. The Compact
Binary encoding is what production deployments use; the other
encodings are debugging or interop conveniences.

Internal Microsoft systems that use Bond include Bing's search
infrastructure, the Cosmos data warehouse, and several Azure
services that predate the gRPC migration. Outside Microsoft, Bond
appears in the open source projects Microsoft has released that
came with Bond dependencies — a few research codebases, some
distributed systems experiments — and in a handful of independent
projects that adopted it deliberately. None of these are large.

The ecosystem gotcha worth noting is *the gradual displacement*.
Microsoft's recommendation for new services has shifted to
Protobuf and gRPC, and the Bond ecosystem's tooling has not kept
up with Protobuf's. Buf-equivalent tools for Bond do not exist;
the breaking-change detection that makes large-scale Protobuf
deployments manageable is mostly missing for Bond. New services
choosing Bond in 2026 inherit this gap and have to build the
operational discipline themselves.

A second gotcha is the multi-encoding architecture. Bond's
flexibility in supporting Compact Binary, Fast Binary, Simple
Binary, and JSON variants is theoretically useful but
operationally cumbersome: the encoding has to be configured per
service, the choice of encoding is part of the deployment
contract, and tooling that snoops on Bond traffic has to handle
all of them. This is the same operational cost Thrift's
multi-protocol design imposes, and it is one of the reasons
Protobuf's single wire format is, on balance, easier to live with.

## When to reach for it

Bond is the right choice when interoperating with an existing
Bond-using Microsoft codebase, where the dependency exists and
the cost of switching is high. It is a defensible choice when
the `bonded<T>` pass-through type is a hard requirement and
Protobuf's `Any` is unsatisfying.

For new general-purpose use cases, Bond is rarely the right
choice. Protobuf is better-supported, better-tooled, and better-
understood; Avro is a more thoughtful answer to a similar
problem; the case for choosing Bond in 2026 has to be specific.

## When not to

Bond is the wrong choice for new microservices in greenfield
environments outside the Microsoft ecosystem. It is the wrong
choice when ecosystem maturity matters; the breaking-change
detection, language-binding-of-the-month, and broad community
support that Protobuf has are all stronger than Bond's
equivalents.

It is also the wrong choice when the multi-encoding flexibility
is unwelcome; new deployments rarely benefit from supporting more
than one wire encoding.

## Position on the seven axes

Schema-required. Not self-describing. Row-oriented. Parse rather
than zero-copy. Codegen-first via the Bond compiler, with runtime
support via reflection-style APIs. Non-deterministic by spec.
Evolution by tagged fields with an explicit `bonded<T>` mechanism
for pass-through.

The cell Bond occupies — schema-required, tagged-field, multi-
encoding, with a forward-compatibility-friendly pass-through
type — is in the same neighborhood as Protobuf and Thrift, with
small distinctive features. The features are real but did not
prove decisive enough to overcome the broader ecosystem
gravity around Protobuf.

## A note on the bonded<T> pattern and the Any / oneof comparison

The `bonded<T>` type deserves more attention than the chapter
gave it, because it represents a design choice that Protobuf has
struggled with and that Bond got right early.

The problem `bonded<T>` solves: a service in a routing or
forwarding role often handles messages whose inner payload is
addressed to a different service. The routing service needs to
look at a few fields (the destination, the priority, perhaps a
correlation ID) but does not need — and arguably should not have
— full schema knowledge of the inner payload. If it had full
schema knowledge, it would have a build-time dependency on every
inner-message schema in the system, which is operationally
expensive and architecturally bad.

Protobuf's answer to this has been the `Any` type, which carries
a type URL and an opaque bytes buffer. The receiver looks up the
type URL and decides whether to unpack. The `Any` type works,
but it has friction: the type URL convention is by string match
rather than by integer ID, the unpacking machinery requires the
schema to be loaded into the runtime separately, and the
identity-of-the-inner-type relies on conventions about URL
resolvers that are not always uniform.

`bonded<T>` declares the inner type explicitly in the schema. A
field of type `bonded<Person>` is known to be a Person, but the
serialized bytes are kept opaque until the field is actually
accessed. The routing service can read the `bonded<Person>`
field's metadata, look at the version, decide whether to forward
or fully parse, and act accordingly. The schema knows the type;
the runtime decides when to materialize it.

This is a small distinction but a meaningful one. `Any` says "the
inner type is whatever the type URL says it is, and we'll figure
it out at runtime." `bonded<T>` says "the inner type is T, and
we'll choose at runtime whether to actually deserialize it." The
latter is more typed and more amenable to static analysis. Bond
got this right; Protobuf has not, and the result is that
forwarding patterns in Protobuf are operationally harder than
they need to be.

## A note on Microsoft's path away from Bond

Several articles and conference talks from Microsoft engineers
over the past few years have addressed the gradual move away
from Bond toward Protobuf. The reasons cited are operational
rather than technical: Protobuf has the better breaking-change
detector (Buf), better cross-team tooling, better language
support outside the languages Bond invested in, better integration
with modern observability stacks, better gRPC integration. The
technical merits of Bond — `bonded<T>`, the richer type system,
the multi-encoding flexibility — have not been compelling enough
to overcome those operational gaps.

The lesson worth absorbing is that in long-running format
choices, *operational maturity outranks technical merit*. Bond
was technically a slight refinement of Protobuf; Protobuf became
operationally a substantial improvement on Bond. The latter
mattered more.

## Epitaph

Bond is Microsoft's contribution to the schema-first wire format
genre; technically thoughtful, operationally outclassed by
Protobuf, gradually displaced inside its own birthplace.
