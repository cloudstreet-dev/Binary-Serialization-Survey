# Hessian

Hessian is the format that runs Dubbo, and through Dubbo it runs a
substantial fraction of the inter-service traffic at the largest
Chinese internet companies. Outside that context, Hessian is a
historical curiosity from the Java RPC world of the early 2000s,
preserved by inertia and by the specific operational characteristics
of the Java enterprise ecosystem. Reading Hessian is the right way
to understand a particular flavor of binary RPC that the Anglosphere
mostly skipped, and a particular set of design choices that made
sense in their original context but have not aged well.

## Origin

Hessian was created by Scott Ferguson at Caucho Technology around
2000, as part of the Resin Java application server. Caucho was a
significant player in the early-2000s Java application server space
— Resin was a competitor to BEA WebLogic and IBM WebSphere — and
Hessian was Caucho's answer to the question of how Java RPC should
work. The competition at the time was Java RMI (Java's
language-specific RPC protocol with a complex serialization format),
SOAP-over-HTTP (the XML-based protocol with verbose envelopes), and
the early CORBA implementations. Hessian's pitch was that Java RPC
should be smaller than SOAP, simpler than RMI, and cross-language by
design.

Hessian 1 was specified and released; Hessian 2 followed with
significant wire-format improvements (more compact encodings,
better support for object references and cycles, a more orthogonal
type system). Hessian 2 is the version most production systems use.
The format spec is a few pages of plain text, the implementations
are small, and the cross-language story is reasonable in the
languages that have implementations (Java, Python, JavaScript, C++,
PHP, .NET, Ruby).

The format would have remained a footnote in Java RPC history if
not for Apache Dubbo. Dubbo is a Java RPC framework originally
developed at Alibaba in 2008, open-sourced in 2011, and donated to
Apache in 2018. Dubbo uses Hessian as its default serialization
format for inter-service messages, and Dubbo's adoption inside
Alibaba and the broader Chinese internet ecosystem (JD.com,
Pinduoduo, ByteDance, and many others) is enormous. Estimates put
the daily message volume of Dubbo deployments at hundreds of
billions per day. Most of that traffic is Hessian-encoded.

The non-Dubbo deployments of Hessian are smaller and aging. Some
older Spring Java services use Hessian via Spring's HTTP invoker;
some financial systems in the JVM ecosystem use it for legacy
reasons; Caucho's own products continue to support it. None of
these are growing.

## The format on its own terms

Hessian 2's wire format is similar in shape to MessagePack and
CBOR but with a Java-specific flavor: the type system reflects
Java's, the encoding handles object references explicitly (so that
a Java object graph with shared references is preserved on the
wire), and field names are encoded as strings rather than integer
tags.

Every value begins with a tag byte. The tag byte's bits encode
both the type and, for short values, the value or its length.
Hessian's tag space is compact: small integers (-16 through 47)
encode as a single byte where the byte is the value with a
specific offset; longer integers escalate through 1-byte, 2-byte,
3-byte, and 4-byte forms. Strings have a similar tiered encoding:
short strings (up to 31 characters) take a single tag byte plus
the UTF-8 (Hessian uses modified UTF-8, like NBT, which is a
Java-ism); longer strings have a length prefix.

Maps and lists use a fixed-size encoding when the size is known
at encode time and a stream encoding when it is not. The stream
encoding is similar to CBOR's indefinite-length form: a start
marker, the elements, and an end marker.

Object references are Hessian's distinctive feature. When an
object is serialized, Hessian assigns it an integer reference and
emits its fields. If the same object appears later in the stream
— directly, or as a member of another object — Hessian emits a
reference rather than re-serializing. This preserves shared
references and handles cyclic graphs (a tree with a back-pointer,
say) without infinite recursion.

The class definitions in Hessian 2 are emitted at the start of
the stream. A class definition gives the class name and the
ordered list of field names; subsequent objects of that class are
emitted as a reference to the class definition followed by the
field values in order. This is a hybrid between schemaless
self-describing (the class definition is in the stream, so the
stream is self-contained) and schema-required (the field order
is positional within the class definition). The result is
denser than MessagePack for streams with many objects of the
same class — the field names amortize over instances — and more
self-describing than Protobuf since the class definition is in
the bytes.

The data model is broadly Java-shaped: integers (with a 64-bit
long type), doubles, strings (Unicode), booleans, null, lists,
maps, and objects (with class definitions). There are also
specific types for dates (encoded as 64-bit milliseconds since
epoch), binary blobs, and a mechanism for arbitrary user-defined
serializable types via the bonded-value mechanism.

## Wire tour

Encoding our Person record. The class definition is emitted first,
followed by an instance:

```
43                                           class definition tag
06 50 65 72 73 6f 6e                         class name length 6, "Person"
96                                           field count: 6 (compact)
02 69 64                                       field 0 name: "id"
04 6e 61 6d 65                                 field 1 name: "name"
05 65 6d 61 69 6c                              field 2 name: "email"
0a 62 69 72 74 68 5f 79 65 61 72              field 3 name: "birth_year"
04 74 61 67 73                                 field 4 name: "tags"
06 61 63 74 69 76 65                           field 5 name: "active"

60                                           object reference (class index 0, no fields-as-mods)
ba                                             id: long = 42 (compact range -8..47 → byte 0xba)
0c 41 64 61 20 4c 6f 76 65 6c 61 63 65         name: short string len 12
15 61 64 61 40 61 6e 61 6c 79 74 69 63
   61 6c 2e 65 6e 67 69 6e 65                  email: short string len 21
59 17 07                                       birth_year: int 1815 (3-byte form)
58 96                                          tags: typed list with length 2
   0d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e   "mathematician"
   0a 70 72 6f 67 72 61 6d 6d 65 72            "programmer"
54                                           active: boolean true
```

The class-definition portion is approximately 50 bytes, dominated
by the field name strings. The instance portion is approximately
75 bytes. Total for a single Person: about 125 bytes, somewhat
larger than MessagePack and substantially larger than Protobuf.

The class definition pays for itself when many instances of the
same class are emitted in the same stream. For 1,000 Person
records in a single Hessian stream, the class definition is
emitted once (50 bytes) and the instances each cost about 75
bytes (no per-instance class definition, no per-instance field
names). The amortized per-record cost approaches Protobuf's,
though Protobuf would still be smaller in absolute terms for the
same payload.

The object-reference encoding shines when the data has shared
structure. If two Person records share a `Department` reference,
Hessian emits the Department once and references it twice,
saving the bytes of the duplicate. MessagePack and Protobuf
have no analogous mechanism; they would emit the Department
twice. For data with high sharing, Hessian's wire-size advantage
can be substantial.

If `email` were absent, Hessian would emit a null marker (0x4e)
in the email slot. The class definition still includes the field;
the instance just has null where the value would have been. This
is the same approach Avro takes with `["null", "string"]` unions.

## Evolution and compatibility

Hessian's evolution story is mediated by the class definition
emitted in the stream. Adding a new field requires updating the
class definition; readers that have a different class definition
locally must reconcile somehow.

The conventional reconciliation is by name: the class definition
gives field names, and the consumer matches them to its local
type's fields. Fields in the stream that the consumer does not
know about are skipped; fields in the consumer's local type that
are not in the stream are filled with default values (null for
references, zero for primitives). This is a graceful-evolution
model and has been used effectively in long-running Dubbo
deployments.

Renaming a field is a breaking change unless the consumer side
implements field-name aliasing manually. Reordering fields in the
class definition is permitted (the stream's class definition is
authoritative), but this can cause subtle bugs if a consumer
caches the class definition incorrectly. Type changes follow
similar rules to Protobuf's: numeric promotions are usually
safe, type changes within different categories (int → string)
are not.

The deterministic-encoding question for Hessian is mostly
unresolved. The format does not specify a canonical encoding
subset. Object references can be assigned in any order; map
ordering is unspecified; integer encodings have multiple valid
forms. Two Hessian encoders producing the same logical value can
produce different bytes. Applications that need byte-equality
have to canonicalize separately, as with Protobuf and Thrift.

## Ecosystem reality

The Hessian ecosystem is bimodal. Inside the Dubbo-using
ecosystem (which is to say, inside the Chinese internet at scale),
Hessian is canonical, the implementations are mature, and the
format has been hardened by a decade and a half of high-volume
production. The Java implementation in Dubbo is the canonical
one; bindings exist in many languages but are typically thinner.

Outside Dubbo, Hessian appears in older Java systems (legacy
Spring HTTP invokers, Caucho Resin deployments, older financial
systems), and in a small number of cross-language tools that
adopted it because Hessian was the path of least resistance for
talking to a Dubbo service.

The cross-language story varies by language. Python's `python-hessian`
and `pyhessian` libraries exist and are maintained. JavaScript's
implementations are more or less working. C++ and .NET
implementations exist but are second-class. Ruby's implementation
is limited.

The most consequential ecosystem fact about Hessian is that it
is closely associated with Dubbo, and Dubbo's choice to support
multiple serialization formats has created options. Dubbo can use
Hessian, Protobuf, JSON, or several others as its serialization
layer; the choice is per-service, and there has been a slow
migration in newer Dubbo deployments toward Protobuf for the
ecosystem reasons that Protobuf wins in most other places. New
Dubbo services in 2026 often use Protobuf rather than Hessian,
even though Hessian remains the default.

The ecosystem gotcha worth noting is *the modified-UTF-8 issue*
(same as NBT — the Java-isms produce bytes that don't round-trip
through standard UTF-8 implementations) and the *class
definition caching* (consumers that cache class definitions
across connections can get out of sync if a producer evolves the
class). Both are well-known in the Dubbo community and have
documented workarounds.

## When to reach for it

Hessian is the right choice when interoperating with Dubbo or
with an existing Hessian-using Java system. It is rarely the
right choice for new systems outside that context.

It is a defensible choice when the object-reference preservation
is a hard requirement and the alternative would be to manually
deduplicate references in the application code; few formats handle
shared references the way Hessian does.

## When not to

Hessian is the wrong choice for new microservices in greenfield
environments. Protobuf, gRPC, and the modern alternatives have
better tooling, better cross-language support, better breaking-
change detection, and broader community knowledge.

It is also the wrong choice when ecosystem maturity matters; the
non-Java implementations are functional but not first-class.

## Position on the seven axes

Schemaless-with-class-definitions-in-stream (a hybrid). Self-
describing (the class definitions are in the stream). Row-
oriented. Parse rather than zero-copy. Codegen via class
definitions, with runtime support for dynamic types. Non-
deterministic by spec. Evolution by name-matching across class
definitions, with graceful skipping of unknown fields and
defaulting of missing ones.

The cell Hessian occupies — class-definition-as-in-stream-schema,
object-reference-aware, Java-flavored — is unusual and is the
combination of design choices that distinguish Hessian from
MessagePack, CBOR, and the schema-first formats. The
distinctiveness is real but has not produced enough leverage to
overcome the broader ecosystem gravity.

## A note on the object-reference mechanism

Hessian's handling of object references deserves a longer look,
because it is the design feature that most distinguishes Hessian
from the other formats in this book and the one that maps least
well onto modern serialization patterns.

The mechanism, in plain terms: when an object is serialized, it is
assigned an integer index. The first time the object appears in
the stream, its full encoding (class definition reference plus
field values) is emitted. Each subsequent appearance of the same
object is emitted as a reference to the integer index. The
decoder maintains a table of seen objects and resolves
references on demand.

This works for shared references and cyclic graphs. It also works
for repeated emission of the same string or large blob, as long
as the encoder's identity-tracking treats them as the same
object — Java's String interning is the relevant detail; two
String references to the same intern slot are the same object,
and Hessian preserves that identity on the wire.

The cost of this is operational. The encoder must track every
serialized object in a table, which grows unboundedly with the
size of the stream. For very large streams, the table memory is
a real concern. Decoders similarly must allocate a table for
resolution. Streams that use heavy reference sharing benefit; those
that don't pay the table overhead for nothing.

The other cost is that Hessian's bytes are stateful. A reference
to object index 5 means nothing without knowing what object
indices 0 through 4 were. Streams cannot be trivially partitioned;
a consumer that wants to read the second half of a Hessian stream
must replay the first half to populate the reference table. This
is awkward for several modern access patterns (random access,
parallel processing, lazy decoding) and is one of the reasons
Hessian has not spread to those use cases.

Modern formats handle reference sharing differently. Protobuf
does not preserve references at all; if a producer has two
fields that point to the same object, both fields are serialized
in full, with no de-duplication. The applications either accept
the duplication or de-duplicate at the application layer. This is
less efficient on the wire but simpler operationally.

The trade Hessian made was right for the Java RPC context of
2000-2010, where typical payloads were small enough that the
table overhead was negligible and shared references were common
in domain models. The trade ages less well in 2026, where
streams are bigger, parallel processing is more common, and the
operational cost of stateful encodings is more painful. Reading
Hessian's design today is the right way to understand a feature
that was reasonable in its time and that the rest of the format
landscape has chosen not to replicate.

## Epitaph

Hessian is the Java-flavored binary RPC format that runs Dubbo
and not much else; technically interesting in its handling of
object references and class definitions, ecosystem-bound to a
context most engineers outside Asia rarely encounter.
