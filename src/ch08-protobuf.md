# Protobuf

Protobuf is the schema-first wire format that won. *Won* in the sense
that more services exchange more bytes per second in Protobuf than in
any other format whose name appears in this book; *won* in the sense
that it is the default reach for any new system that needs a typed
binary protocol; *won* in the sense that the surrounding ecosystem
(gRPC, Buf, the Confluent Schema Registry's Protobuf support, the
half-dozen high-quality codegen tools) has so much momentum that even
its deficiencies are not enough to dislodge it. Understanding Protobuf
is mandatory background for anyone working in modern distributed
systems, and the parts of it that are interesting are not always the
parts that get the most attention.

## Origin

Protocol Buffers was built at Google around 2001 to replace an older
internal serialization format whose author had moved on and whose
maintenance had become a liability. The new format had three goals:
to support schema evolution, because Google's index-build pipeline
involved hundreds of binaries running on different release schedules
and the format had to tolerate version skew; to be fast and small,
because the bytes it carried were the inter-service traffic of a
search engine; and to be language-agnostic, because Google had at
least three production languages at the time (C++, Java, Python)
and needed all three to read and write the same bytes.

Protobuf 1 and Protobuf 2 were internal to Google. Protobuf 3 was
the first version made widely available outside Google, in 2008,
after several years of internal use refined the format and its
codegen pipeline. The 2 → 3 transition was incompatible at the
schema level — Protobuf 3 removed the `required` keyword, changed
the default value semantics for scalars, and dropped support for
extensions in favor of a more constrained `Any` type — but the wire
format itself was unchanged, which meant Protobuf 2 binaries and
Protobuf 3 binaries could continue to talk to each other if their
schemas remained compatible.

The wire format has been frozen since 2008. The schema language
(`.proto` files) has gained features, the runtime libraries have
churned, the surrounding ecosystem has accreted layers, but the
bytes have not changed. This is a remarkable property and one of
the principal reasons Protobuf has held up under decades of use.

Protobuf's open-source release in 2008 coincided with the rise of
microservices, the broader movement away from REST-with-JSON for
internal traffic, and the appetite for a typed schema language that
could be checked at build time. gRPC, released in 2015, made
Protobuf the default wire format for an HTTP/2-based RPC framework
that was easy to consume. Once gRPC took hold, Protobuf's market
position became roughly unassailable.

## The format on its own terms

A Protobuf message is a sequence of *fields*. Each field has a
*number* (assigned in the schema, never reused), a *wire type* (one
of six: varint, fixed-64, length-delimited, start-group, end-group,
fixed-32), and a *value*. The wire encoding of a field is a *tag*
byte (or bytes) — a varint that packs the field number and wire type
together — followed by the value, encoded according to its wire type.

The wire type and field number are packed into a single varint, with
the wire type in the low three bits and the field number in the
remaining bits. For field numbers 1 through 15 this is a single byte;
for field numbers 16 through 2047 it is two bytes; and so on. The
guideline of "use field numbers 1-15 for the most common fields" is
not stylistic; it is a real two-byte savings per occurrence.

The varint encoding deserves explanation. A varint is a sequence of
seven-bit groups, little-endian, with the high bit of each byte
indicating whether more bytes follow. Values 0-127 take one byte;
128-16383 take two; and so on. Negative values, when encoded as
varints without zigzag transformation, take ten bytes — the
high-bit-padding eats most of the encoding. For signed integer
fields, Protobuf offers a *sint32* and *sint64* type that uses
zigzag encoding (mapping signed integers onto unsigned ones such
that small absolute values produce small encodings, regardless of
sign), which is dramatically smaller for negative values.

There are six wire types but only four widely used. Wire type 0 is
varint, used for integers, enums, and booleans. Wire type 2 is
length-delimited, used for strings, byte arrays, embedded messages,
and packed repeated fields. Wire types 1 and 5 are fixed-64 and
fixed-32, used for the `fixed64`, `fixed32`, `double`, and `float`
schema types where the schema explicitly opts into a fixed-width
encoding. Wire types 3 and 4 (start-group, end-group) were used by
Protobuf 1 and 2's group syntax, which Protobuf 3 deprecated and
which is now effectively dead.

The structural property to internalize is that *every field is
self-describing on the wire*. Every field's bytes begin with the
tag, which identifies the field number and tells the reader how
many bytes the value occupies (directly, for fixed-width types;
via the length prefix, for length-delimited types; via the
high-bit-terminated varint, for varint types). A reader that does
not recognize a field number can still skip the field cleanly. This
is the wire-level mechanism that makes Protobuf's schema evolution
work.

Optional fields, repeated fields, and message-typed fields all use
existing wire types — there is no special "optional" or "repeated"
marker. An optional field is simply a field that may or may not
appear. A repeated field is one that appears zero or more times.
A message-typed field is a length-delimited field whose bytes are
themselves a Protobuf-encoded message. The recursion is uniform.

There is no map type at the wire level. Map fields in `.proto` files
are syntactic sugar for `repeated` of an embedded message with key
and value fields. The map type is a feature of the schema language
and the generated code, not the wire format.

## Wire tour

Schema:

```protobuf
syntax = "proto3";

message Person {
  uint64 id = 1;
  string name = 2;
  optional string email = 3;
  int32 birth_year = 4;
  repeated string tags = 5;
  bool active = 6;
}
```

Encoded:

```
08 2a                                        field 1 (id), varint, value 42
12 0c 41 64 61 20 4c 6f 76 65 6c 61 63 65    field 2 (name), len 12, "Ada Lovelace"
1a 15 61 64 61 40 61 6e 61 6c 79 74 69 63
   61 6c 2e 65 6e 67 69 6e 65                field 3 (email), len 21, "ada@analytical.engine"
20 97 0e                                     field 4 (birth_year), varint, value 1815
2a 0d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e field 5 (tags), len 13, "mathematician"
2a 0a 70 72 6f 67 72 61 6d 6d 65 72          field 5 (tags), len 10, "programmer"
30 01                                        field 6 (active), varint, value 1
```

71 bytes — about a third smaller than MessagePack and more than half
the size of BSON. The wins come from three places. First, field
numbers replace field names; `id` (3 bytes in MessagePack) becomes
the tag byte 0x08, which packs both the field number 1 and the wire
type 0 (varint) into a single byte. Second, repeated string fields
share the same tag — `2a` appears twice for the two tags entries,
not once with an array prefix; this loses to MessagePack for very
short repeated strings but wins as soon as the strings are long
enough that the per-element tag byte amortizes. Third, integers use
varint encoding; 42 is one byte, 1815 is two bytes.

The bytes show several Protobuf-specific design choices. The
order of fields in the wire format matches the order they were
emitted by the encoder, not the order they were declared in the
schema. The schema-declared field number is what matters; field
order is not significant on the wire and decoders must accept fields
in any order. The repeated string field for tags is *not* packed —
packed encoding is permitted only for repeated fields of primitive
numeric types, where the elements are concatenated within a single
length-delimited field. Strings cannot be packed, which is why the
two tag entries each get their own tag byte.

If `email` were absent, the encoding would simply omit the
`1a 15 ...` portion, dropping 23 bytes. There is no marker for
absence; the field is either in the bytes or it is not. The
generated decoder reports `has_email() == false` for the absent
case, distinguishing it from `email == ""`. This is the optionality
mechanism Protobuf 3 restored after Protobuf 3.0 originally removed
it; for several years Protobuf 3 conflated absent with
default-valued for scalar fields, and the resulting bugs were so
widespread that the `optional` keyword was added back in Protobuf
3.15.

## Evolution and compatibility

The evolution model is the single feature that justifies most of
Protobuf's design choices. The model in plain terms:

Adding a new field is safe in both directions. New producers emit
the new field; old consumers see an unknown field number and skip
it (the wire type tells them how). Old producers do not emit the
field; new consumers see no field with that number and treat the
field as absent (or, for scalar types in early Protobuf 3, as
defaulted to zero/empty/false).

Removing a field is safe with a procedure. The schema-level rule is
to mark the field as `reserved` so that no future schema reuses
the field number for a different type. Old producers may continue
to emit the field; new consumers will skip it as unknown. There
is no wire-level effect of removal; the bytes are unaffected.

Renaming a field is safe at the wire level (the field number is what
matters; names are decorative). Renaming is unsafe at the source
level if any code references the old name, which all of it usually
does, and so renaming requires a careful migration of generated
code.

Changing a field's type is mostly unsafe but has a small set of
exceptions. int32 and int64 are wire-compatible (varint of the same
value produces the same bytes). int32 and uint32 are wire-compatible
for non-negative values but not for negative. int32 and sint32 are
*not* wire-compatible because sint32 uses zigzag and int32 does not.
String and bytes are wire-compatible because both use length-delimited
wire type and the bytes themselves are arbitrary. Most other type
changes are wire-incompatible and amount to schema-level deletions
followed by additions.

Reusing a field number is the cardinal sin of Protobuf schema
evolution. If field 5 was a `string name` and is now a `repeated
int32 ids`, old producers will emit a length-delimited string
under tag 0x2a, and new consumers will attempt to decode the string
bytes as a packed array of int32s. The result will be silent
garbage, occasional crashes when the bytes happen to be malformed
varints, and hours of debugging. The `reserved` keyword exists to
make this mistake impossible, and using it religiously is the
single most important Protobuf hygiene rule.

The `optional` keyword in Protobuf 3 (since 3.15) restores
field-presence tracking for scalar types. Without `optional`, a
scalar field's default value (0 for numbers, "" for strings, false
for booleans) is indistinguishable from absence on the wire and in
the generated API. With `optional`, the field gets a `has_` accessor
in the generated code, and presence is preserved through encoding
and decoding. New code should use `optional` for any scalar where
"unset" is meaningfully different from "default."

## Ecosystem reality

The Protobuf ecosystem is enormous, mature, and not always easy to
navigate. The reference implementation is `protoc`, the Google-
maintained schema compiler that generates code in C++, Java,
Python, Objective-C, C#, Ruby, Dart, PHP, JavaScript, and Kotlin.
Generated code in other languages comes from third-party plugins:
`protoc-gen-go` for Go (now the official choice), `prost` for Rust,
`tonic` for Rust gRPC, `protoc-gen-ts` for TypeScript, and several
others. Quality varies, and the choice of plugin within a language
sometimes matters: Go has had two generations of generated code,
and the older `gogoproto` was meaningfully faster than the modern
`google.golang.org/protobuf` for some workloads, at the cost of
diverging from the canonical generator.

Buf (buf.build) is the most important development of the last
decade in the Protobuf world. Buf provides a schema linter, a
breaking-change detector, a remote schema registry (the *Buf Schema
Registry*), and a build system that wraps `protoc` with sensible
defaults. The breaking-change detector is the killer feature: it
parses your schema diff in CI and rejects pull requests that change
field numbers, change field types incompatibly, remove fields
without reserving them, or do any of the other things that produce
silent wire-level breakage. Adopting Buf is the single largest
upgrade most Protobuf-using teams can make.

gRPC is the dominant RPC framework over Protobuf. The relationship
between the two is a well-defined separation: Protobuf defines the
message format and the service IDL; gRPC defines the request/response
flow, the framing over HTTP/2, the streaming semantics, and the
connection lifecycle. You can use Protobuf without gRPC (many do)
and you can use gRPC without Protobuf (you can supply alternative
codecs), but the canonical pairing is the dominant one.

Confluent Schema Registry has supported Protobuf since 2019. The
Kafka ecosystem now has three first-class schema formats — Avro
(historical default), Protobuf, and JSON Schema — and the choice
between them inside Kafka is mostly aesthetic. Protobuf in Kafka is
common enough that several large companies have moved off Avro
specifically to align Kafka with their service-RPC schemas.

The text format (`prototext`) is the canonical human-readable
representation of a Protobuf message. It is not JSON. It is also
not stable across versions in the strict sense — the bytes
round-trip, but the formatting may not. For configuration files,
`prototext` is the right choice. For interchange with non-Protobuf
systems, the JSON encoding (Protobuf 3 has a defined JSON mapping)
is more portable.

Two ecosystem gotchas worth noting. First, the Python implementation
is famously slow; the C++ extension (which most production code
uses) is much faster than the pure-Python fallback, and on
constrained environments where the extension is unavailable the
performance gap is severe. Second, the `Any` type — Protobuf's
mechanism for embedding arbitrary serialized messages — requires
the consumer to know the embedded type's schema separately, which
in practice means `Any` fields are usually the wrong choice and a
`oneof` is what you wanted.

## When to reach for it

Protobuf is the right default for new typed binary protocols
between services you control. It is the right choice for
gRPC-based microservices, period; the alternatives are weaker on
some axis. It is the right choice for Kafka topics where the
producer and consumer are in different languages and the schema
must be enforced. It is a reasonable choice for typed configuration
formats, though the ergonomics of `prototext` are dated.

It is also the right choice for any system where forward and
backward compatibility under heterogeneous deployment is a hard
requirement. The tagged-field model, combined with Buf's
breaking-change detector, gives you the strongest static guarantee
of compatibility that any wire format provides.

## When not to

Protobuf is the wrong choice when the schema cannot be enforced
between producer and consumer (public APIs to undifferentiated
clients), when zero-copy access is required (FlatBuffers or Cap'n
Proto), when columnar layout is required (Parquet, Arrow), when
deterministic encoding is required (the Protobuf "deterministic"
mode is a best-effort and is not canonical across versions), or
when human-editable text is the primary representation (use a YAML
or JSON-based config language and convert).

It is also the wrong choice when the operational cost of schema
distribution exceeds the savings: small projects, prototypes,
public-facing endpoints. JSON is operationally cheaper for those.

## Position on the seven axes

Schema-required. Not self-describing (the bytes are tag-and-value
without semantic field names; the schema is mandatory to interpret).
Row-oriented. Parse rather than zero-copy. Codegen, with `DynamicMessage`
available as a runtime escape hatch. Non-deterministic by spec; the
deterministic mode is a best-effort opt-in. Evolution by tagged
fields, with `reserved` as the explicit hygiene tool.

The cell Protobuf occupies — schema-required, tagged-field evolution,
varint-dense, codegen-first — is the cell most new typed binary
formats are compared against, and the comparison is hard to win.

## Epitaph

Protobuf is the typed binary protocol that the rest of the industry
spent twenty years failing to displace, and the only meaningful way
to lose to it is to choose a format that is better at exactly one
thing you care about more than tagged-field evolution.
