# CBOR

If MessagePack is the format engineers chose, CBOR is the format the IETF
chose. The two are close enough that any serious user of one knows the
other exists; the differences between them are small in cardinality but
large in temperament. CBOR was designed in a standards body, by people
whose day jobs involve writing protocols that must be implemented correctly
by hundreds of independent vendors over decades, and the format reflects
that audience and that lifespan.

## Origin

CBOR was published as RFC 7049 in 2013, authored by Carsten Bormann and
Paul Hoffman. The acronym expands to *Concise Binary Object
Representation*, and the title of the RFC is honest about the design
goals: small enough to be useful in constrained environments, simple
enough to be implemented from scratch, extensible without breaking
existing decoders, and standardized through the IETF process so that
protocol authors could rely on the format being available, frozen, and
fully specified.

The immediate motivator was CoAP, the Constrained Application Protocol,
which was being designed for IoT-scale devices that could not afford
the overhead of HTTP, JSON, or, in some cases, even TCP. CoAP needed
a binary payload format that would survive on devices with kilobytes
of RAM, and JSON was not going to. MessagePack was an obvious starting
point, but its specification at the time was a community artifact, not
an IETF document, and the IETF had a process for producing the latter.
RFC 7049 was the result.

The format was revised in 2020 as RFC 8949, which is the current
authoritative document. RFC 8949 added several clarifications, defined
a deterministic encoding subset, and incorporated the experience of
seven years of deployment. The wire format itself did not change in
any incompatible way; existing CBOR encoders and decoders continue to
work without modification.

CBOR is now an obligate dependency of a small fleet of higher-level
protocols. COSE (CBOR Object Signing and Encryption, RFC 8152) is the
CBOR analogue of JOSE/JWT and is used by FIDO2, WebAuthn, and the
IETF's ACE family of authorization protocols. CWT (CBOR Web Token,
RFC 8392) is the CBOR analogue of JWT and is used wherever the bearer
token needs to be small. The EU Digital COVID Certificate uses CWT
inside QR codes; the bytes of those QR codes are CBOR. Matter, the
smart-home interop protocol led by Apple, Google, Amazon, and Samsung,
uses CBOR throughout. So does the IETF's Software Updates for
Internet of Things working group. CBOR is the format you reach for
when you are designing a protocol that needs to be small, durable,
and fully specified, and you do not control the implementations.

## The format on its own terms

CBOR is built from a single, ruthlessly regular encoding rule. Every
value begins with one byte. The high three bits of that byte encode
the *major type* (one of eight). The low five bits encode either the
value directly (if the value fits) or the size of a follow-on field
that contains the value. The eight major types are: unsigned integer,
negative integer, byte string, text string, array, map, semantic tag,
and a catch-all called *simple values and floats*.

The five-bit *additional information* field follows a uniform
convention. Values 0 through 23 are the value itself (when the major
type is an integer, this means the integer is 0-23 directly; when it
is a string, the length is 0-23 directly). Values 24, 25, 26, and 27
mean "the value or length is in the next 1, 2, 4, or 8 bytes,
respectively, big-endian." Values 28, 29, and 30 are reserved.
Value 31 means "indefinite length," which is CBOR's mechanism for
streaming: the length is unknown at the time of encoding, and a
"break" byte (0xff) terminates the value when it has been fully
emitted.

This regularity is the design's central virtue. A CBOR decoder is
about a hundred lines of C. There is one parsing routine that handles
all eight major types, because the framing is identical across them.
The dispatch on major type happens after the framing is parsed,
which means a partial parser can skip a value it does not understand
without knowing what the value is. Skipping unknown values is exactly
the property that makes a format extensible without breaking decoders,
and CBOR is structurally equipped to do it; the application layer
gets this for free, with no per-format machinery.

The semantic tag major type — major type 6 — is the format's
extensibility hinge. A tag is a small integer that wraps a single
following value, telling the decoder *the value should be interpreted
as*. Tag 0 means "the following text string is an ISO 8601 date";
tag 1 means "the following number is a Unix epoch timestamp"; tag 32
means "the following text string is a URI"; tag 64 means "the
following byte string is an array of unsigned 8-bit integers." The
IANA maintains a registry of assigned tag values; new tags are
allocated through a lightweight first-come-first-served process.
Tags do not change the wire format; a decoder that does not recognize
a tag returns the inner value untagged, and the application either
copes or rejects.

The simple values family (major type 7) is the home of `false`
(0xf4), `true` (0xf5), `null` (0xf6), `undefined` (0xf7), and the
floating-point types: half-precision (0xf9 + 2 bytes), single-precision
(0xfa + 4 bytes), and double-precision (0xfb + 8 bytes). The break
byte (0xff) lives in this family too. The space is otherwise reserved
for spec-defined or application-defined simple values that need to
fit in a single byte.

CBOR is *not* a strict superset of JSON. The data model differs in
small but real ways: CBOR has byte strings (JSON does not, except
through base64), CBOR has integers up to 64 bits (JSON's spec is
silent past 53 bits), CBOR has tagged semantic values (JSON has no
analogue), CBOR distinguishes `null` from `undefined`, and CBOR's
map keys can be any value (JSON's must be strings). For most
payloads the round-trip CBOR → JSON → CBOR is lossy, and the JSON
sequence specification (RFC 8259, in JSON Text Sequences and
related documents) tries to bridge this gap with mixed success.

## Wire tour

Encoding our Person record:

```
a6                                           map of 6 entries
  62 69 64                                   key "id" (text string len 2)
  18 2a                                      value 42 (uint, 1-byte follow)
  64 6e 61 6d 65                             key "name" (text string len 4)
  6c 41 64 61 20 4c 6f 76 65 6c 61 63 65     value "Ada Lovelace" (text string len 12)
  65 65 6d 61 69 6c                          key "email" (text string len 5)
  75 61 64 61 40 61 6e 61 6c 79 74 69 63
     61 6c 2e 65 6e 67 69 6e 65              value "ada@analytical.engine" (text string len 21)
  6a 62 69 72 74 68 5f 79 65 61 72           key "birth_year" (text string len 10)
  19 07 17                                   value 1815 (uint, 2-byte follow, big-endian)
  64 74 61 67 73                             key "tags" (text string len 4)
  82                                           array of 2 elements
    6d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e   "mathematician" (text string len 13)
    6a 70 72 6f 67 72 61 6d 6d 65 72            "programmer" (text string len 10)
  66 61 63 74 69 76 65                       key "active" (text string len 6)
  f5                                           value true
```

Total: 105 bytes — one more than MessagePack. The single byte of
overhead is in the encoding of `id`: 42 is greater than 23 and so
does not fit in CBOR's immediate-value range, requiring the byte
0x18 (additional info 24, meaning "1-byte follow") plus the value
0x2a. MessagePack's positive fixint encoding goes up to 127 in a
single byte and so encodes 42 in one byte total.

This single-byte difference is essentially the entire density gap
between MessagePack and CBOR for typical payloads. It applies to
integers in the range 24 through 127, where MessagePack uses one
byte and CBOR uses two. For larger integers, smaller integers, all
strings, all arrays, all maps, the formats encode in identical
counts. The two formats really are extraordinarily close.

The byte-level difference that does have practical consequences is
the array and map prefix encoding. MessagePack uses fixarray (low
nibble = count) for arrays up to 15 elements; CBOR uses immediate
(low five bits = count) for arrays up to 23 elements. For maps and
strings the same difference applies. In typical telemetry data,
both ranges are large enough that the difference rarely appears.

If `email` were absent, the encoding would shrink by 28 bytes (the
key plus the value), and the map prefix would change to 0xa5 (map
of 5). As with MessagePack, absence is encoded by omission, not by
a sentinel.

## Evolution and compatibility

CBOR's evolution story is the same as MessagePack's at the wire-format
level — keys are strings, the application treats unknowns as it sees
fit — but with one substantial addition: the semantic tag. A tag is
a versioning hook by design. A producer that wants to introduce a
new interpretation of an existing value type can wrap that value in
a new tag; consumers that recognize the tag interpret the value
specially, and consumers that do not recognize the tag fall through
to the underlying value unchanged. This is a forward-compatibility
mechanism baked into the wire format, and it is genuinely useful.

The deterministic encoding subset defined in RFC 8949 is the second
addition. *Deterministic encoding* means: integers use the smallest
representation that fits; floats use the smallest representation
that exactly preserves the value; map keys are sorted by their byte
encoding; indefinite-length encodings are not used; tags use minimal
encoding. A CBOR producer that emits deterministic encoding produces
the same bytes for the same value every time, and two producers
emitting deterministic encoding produce the same bytes as each other.
This makes CBOR usable for content addressing, signing, and
deduplication — all the use cases where MessagePack's lack of a
canonical form forces extra coordination.

The COSE family of protocols depends on deterministic encoding for
its signature schemes. The IPLD ecosystem (the content-addressed
data layer underlying IPFS and Filecoin) uses dag-cbor, a slight
restriction of CBOR that mandates deterministic encoding plus a few
additional rules. ACE/OAuth tokens lean on the same property.
Without deterministic encoding, none of these protocols would work.

The CDDL specification language (RFC 8610, *Concise Data Definition
Language*) gives CBOR a formal schema language and the ability to
validate payloads against schemas. CDDL is not required to use CBOR,
and most uses of CBOR do not involve CDDL, but it exists for the
cases where you want it. CDDL is broadly comparable to JSON Schema
in scope, with type rules better suited to CBOR's richer type system.

## Ecosystem reality

CBOR's ecosystem is structurally different from MessagePack's. The
implementations are fewer in number and more uniformly high-quality,
because the format's primary user base is in standards-driven
ecosystems where wire-level interoperability is non-negotiable. The
canonical C implementation is libcbor, and there are mature
implementations in Rust (ciborium, serde_cbor), Go (fxamacker/cbor —
the de facto Go implementation, with explicit support for COSE
and deterministic encoding), Java (jackson-dataformat-cbor and
co.nstant.in.cbor), Python (cbor2), and JavaScript (the cbor and
cbor-x packages). Most of these libraries support deterministic
encoding via configuration, and the ones that do not are usually
discoverable as the inferior choice.

The deployments that pull CBOR in are concentrated in standards-
driven domains. WebAuthn assertions are CBOR; FIDO2 attestations are
CBOR; the EU's COVID certificate format is CBOR; ACE tokens are CBOR;
Matter messages are CBOR; the IETF's CORE working group's protocols
are CBOR; the COAP-WG's payloads are CBOR; OpenID Connect's CIBA
flow can use CBOR. None of these deployments would have considered
MessagePack, because none of them would have accepted a format
without an RFC. CBOR's existence is the reason none of them had to
invent their own format.

The format also shows up in a few less-standards-driven places.
LightStep used CBOR in some of its tracing libraries. Rust's serde
ecosystem uses CBOR through ciborium and serde_cbor for general
purpose binary serialization. The Cap'n Proto JSON encoding is, in
some configurations, CBOR-shaped. None of these are dominant uses;
the standards-driven cases are.

Two ecosystem gotchas worth noting. First, indefinite-length
encoding is permitted by the spec but disallowed by deterministic
encoding, and a surprising number of CBOR producers use it for
streaming output. Decoders that work in protocols requiring
deterministic encoding must reject indefinite-length values; some
libraries do not by default. Configure carefully if you are
implementing one of the COSE-family protocols.

Second, the half-precision float type is real and is occasionally
emitted by libraries that aggressively shrink. Some decoders do not
handle half-precision floats correctly (or at all). If your data
includes floats, test the round-trip; do not assume.

## When to reach for it

CBOR is the right choice when MessagePack would be a candidate but
the deployment context is one of the following: a public protocol
where the spec must be referenceable and immutable; a deterministic
encoding requirement (signing, content addressing, deduplication);
a constrained-device deployment where the spec stability matters
more than fewer bytes; or a deployment that intersects with one of
the existing CBOR-using protocols (COSE, CWT, FIDO, Matter), where
piggy-backing on CBOR avoids a format-translation layer.

CBOR is also a defensible default for new internal binary protocols,
even when none of the above apply, on the principle that a format
governed by an IETF RFC is harder to accidentally diverge from than
a format governed by a community-maintained website. The handful of
extra bytes per integer is rarely the bottleneck.

## When not to

When you have a stable schema you control on both sides, the
schemaless self-describing nature of CBOR is paying for flexibility
you do not need; switch to a schema-required format. When you need
zero-copy access, CBOR's varint-prefixed strings disqualify it. When
you need columnar layout, CBOR's row-oriented structure disqualifies
it. When the payload is going to be edited by humans, CBOR's binary
encoding disqualifies it (use the JSON sequence equivalent, which is
not the same format). When you simply want the densest possible
schemaless binary encoding, MessagePack will be one byte per integer
shorter on small values, which over a billion records is a real
size; consider both.

## Position on the seven axes

Schemaless. Self-describing. Row-oriented. Parse rather than zero-copy.
Runtime bindings; CDDL is available for those who want a schema, but
the format does not require one. Deterministic encoding subset
specified, used widely in protocol contexts, opt-in elsewhere.
Evolution by application convention, with semantic tags as a
forward-compatibility hook.

CBOR's stance on the seven axes is identical to MessagePack's on
six of them and stronger on the seventh: where MessagePack has no
canonical form by spec, CBOR does. This single difference is the
reason CBOR exists and the reason MessagePack does not show up in
COSE, WebAuthn, FIDO2, or Matter. For protocols where bytes are
signed or hashed or addressed by content, the lack of a canonical
form is a deal-breaker; for protocols where they are not, the
difference is academic. The choice between MessagePack and CBOR is
therefore mostly the choice of which side of that line your system
sits on.

## A note on the JSON-equivalence question

CBOR is sometimes pitched as "JSON in binary" the way MessagePack
sometimes is, and the pitch is approximately as honest in either
case. Most JSON values round-trip through CBOR fine. The exceptions
are: integers larger than 2^53 (CBOR preserves them, JSON does not
specify how), byte strings (CBOR has them, JSON does not), and
date semantics (CBOR has tags for dates, JSON has only strings).
Going the other direction — CBOR to JSON — is lossier still: tagged
values cannot be represented in JSON without a convention, byte
strings have to be base64-encoded, and floats with half-precision
have to be promoted.

Practically, this means the MessagePack and CBOR communities have
settled on similar mental models for their formats — *like JSON, but*
— with the understanding that the *but* clause is doing a lot of
work. The CBOR community has been more honest about this than the
MessagePack community, perhaps because the IETF-style specification
process forces a level of precision that a community-maintained
website does not.

## Epitaph

CBOR is MessagePack with standards-body discipline: same data model,
same density, but with a normative spec, a deterministic encoding
mode, and a thriving graft of higher-level protocols. Reach for it
when the protocol must outlive its authors.
