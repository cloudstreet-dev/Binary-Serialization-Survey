# Schema Evolution Compared

The three preceding chapters covered Protobuf, Thrift, and Avro
each on its own terms. The choice between them is rarely about
wire-level efficiency, which is approximately the same across all
three for typical payloads. The choice is almost always about how
the format handles the case where the schema changes and the
deployment cannot upgrade producers and consumers in lockstep.
This chapter sets up that question directly: a small fixed list of
schema-change scenarios, applied to all three formats in turn,
with the rules and consequences laid out side by side.

The reason for treating this comparison as its own chapter rather
than scattering it through the format chapters is that the
comparison itself is a useful artifact. The right way to choose a
schema-first format for a new system is to walk through the changes
you expect to make to the schema over the next five years and ask
which format makes the easiest of them, which makes the hardest of
them, and where the format's hard cases line up with the changes
you actually need to make. Skipping this exercise produces formats
chosen for the wrong reasons; doing it produces choices that hold up.

The list of scenarios is deliberately small. Real-world schema
evolution is more varied than seven scenarios can capture, but
seven is enough to surface the differences between the formats.
Each scenario describes the change, the constraints on producers
and consumers, and the result for each format.

## Scenario 1: Add a new optional field with no default

The cleanest case. We have a `Person` schema. We want to add a
`country` field that may or may not be present.

In **Protobuf**, the change is a one-line schema edit: `optional
string country = 7;`. Field number 7 is new. Old producers do not
emit it. New producers may emit it. Old consumers ignore it (the
unknown-field code path skips it cleanly). New consumers receive
either an absent field (if the producer is old) or a present field
(if new). Nothing breaks. No coordination is required between
producer and consumer deployment. This is the case Protobuf was
designed for, and it works.

In **Thrift**, the change is a one-line schema edit:
`7: optional string country;`. The behavior is identical to
Protobuf's: old producers omit, new producers emit, consumers
gracefully handle either. Field IDs are the wire-level identity.
Nothing breaks.

In **Avro**, the change requires a default value: `{"name":
"country", "type": ["null", "string"], "default": null}`. The
default is mandatory because Avro reader-writer schema resolution
requires every field in the reader's schema that isn't in the
writer's schema to have a default. Without the default, an old-
producer/new-consumer combination fails resolution at decode time.
With the default, the change is forward-compatible (old reader
sees bytes from new writer and ignores the new field) and
backward-compatible (new reader sees bytes from old writer and
fills in the default).

In the **schemaless self-describing formats** (MessagePack, CBOR,
BSON), the change is: emit the new key when the application has a
value, don't emit it when it doesn't. There is no schema to update
because there is no schema. Old consumers ignore the unknown key;
new consumers handle the missing key as the application sees fit.
The format is uninvolved.

## Scenario 2: Add a new required field

The dangerous case. We want to add a `country` field that must be
present in every record.

In **Protobuf 3**, the `required` keyword is gone, so this scenario
is technically impossible at the schema level. The closest you can
do is add an optional field and enforce required-ness in
application code. This is the answer the Protobuf team prefers,
and the cost is real: the schema does not document the invariant,
and consumers must remember to check.

In **Thrift**, the change is `7: required string country;`. The
behavior at first glance looks fine: new producers emit, new
consumers expect, both sides updated together. The trap is the
deployment sequence. If the producer ships first, old consumers
receive a record with an unknown field, which they skip — but old
consumers do not know to expect the field, so they are unaffected.
Fine. If the consumer ships first, it expects the field to be
present, but old producers do not emit it, and the decoder fails
loudly. The required field cannot be deployed without a strict
deployment order: producers must update before consumers, every
time.

In **Avro**, this scenario is *also* technically impossible without
a default. Adding a field with no default and reading old data
fails at resolution time. The pattern that approximates "required"
in Avro is: add the field with a default in the schema, and have
application code reject records where the value is the default.
This is the same pattern Protobuf 3 uses, with the same costs.

In the **schemaless formats**, the scenario is purely an
application concern. The format will not help.

The lesson is that *adding a required field is a coordinated
deployment in any format*. Thrift will make the failure loud.
Protobuf and Avro will make it quiet. Neither is automatically
safer; both require the same operational care.

## Scenario 3: Remove a field

We want to remove the `email` field. The field is currently optional.

In **Protobuf**, the change is `reserved 3;` (or `reserved
"email";`) plus removal of the field declaration. The `reserved`
keyword prevents future schema versions from reusing field number
3 for a new field of a different type. Old producers may continue
to emit the field; new consumers see the field number as unknown
and skip it. New producers do not emit the field; old consumers
see no field with that number and treat it as absent (its default).
The change is safe in both directions.

In **Thrift**, the change is the removal of the field declaration.
Thrift does not have a `reserved` keyword in mainstream syntax, but
the rule against reusing field IDs is identical: never reassign a
removed ID. Old producers continue to emit; new consumers skip.
New producers don't emit; old consumers see absence. If the field
was `required`, the rules are the same as Scenario 2 in reverse:
old consumers expecting the field will fail when new producers
omit it. Removal of a required field is a coordinated deployment.

In **Avro**, removing a field requires the field to have had a
default in the writer's schema (so that old readers reading new
bytes can decode the absent field). The Avro registry will reject
the schema change if the consumer-side compatibility mode is
"backward" or "full" and the field has no default. This is one of
the cases where Avro's resolution-based model is more restrictive
than Protobuf's tag-based model: in Protobuf you can remove a
field without consequences (modulo `reserved`), and old data still
decodes because the field number is just unknown; in Avro you have
to have planned for the removal at the time of the field's
introduction by giving it a default.

In the **schemaless formats**, removal means stop emitting the key.
Consumers handle the missing key. There is no policy to enforce.

## Scenario 4: Rename a field

We want to rename `birth_year` to `year_of_birth`.

In **Protobuf**, this is a non-event at the wire level: field
number 4 still encodes the value, regardless of the field's source-
level name. The bytes are unchanged. The cost is in the source code:
every reference to the field name needs to be updated, generated
code regenerated, and any code that uses reflection-by-name has to
be migrated. The wire is fine; the source is the work.

In **Thrift**, the same: field IDs are wire-level identity, names
are decorative. Rename the field, regenerate, redeploy.

In **Avro**, the wire encoding is positional and does not carry the
field name. But the resolution algorithm matches reader-schema
fields to writer-schema fields *by name* (with aliases as the
explicit override). To rename, declare the new name and add the
old name as an alias: `{"name": "year_of_birth", "type": "int",
"aliases": ["birth_year"]}`. Without the alias, an old writer
schema and new reader schema will fail to resolve the renamed
field, and the value will be missing in decoded records.

In the **schemaless formats**, rename means start emitting the new
key, optionally keep emitting the old one for compatibility, and
have consumers handle both. There is no formal mechanism. The
operational discipline is identical to Protobuf's, but spread
across application code instead of schema files.

## Scenario 5: Change a field's type

We want to change `birth_year` from `int32` to `int64`. We also
want to consider the harder case of changing it from `int32` to
`uint32`.

In **Protobuf**, `int32` and `int64` are wire-compatible: the
varint encoding of small positive values is identical, and the
decoder for int64 accepts the int32 wire bytes. `int32` and
`uint32` are wire-compatible *for non-negative values*; for
negative values the encodings differ in sign-extension behavior,
which is why this change is documented as "compatible only if all
values are non-negative." `int32` to `sint32` is *not* compatible
because the encodings differ (zigzag vs. straight varint), and
this is a common mistake. The compatibility table for Protobuf
type changes is well-known and is the kind of thing breaking-change
detectors like Buf check automatically.

In **Thrift**, the equivalent table exists but is shorter: `i32`
and `i64` are wire-compatible because Thrift Compact zigzag-encodes
both, `i32` and `i64` are wire-incompatible with `string` and
`binary` because they use different wire types. There is no
distinction between zigzag and straight varint at the schema level,
which means Thrift does not have the int32-to-sint32 trap.

In **Avro**, type changes go through the resolution algorithm's
*type promotion* rules: int → long → float → double, in that order.
Changing `birth_year` from `int` to `long` is a forward and
backward compatible change. Going the other direction (long to
int) is *not*, because old data may include values larger than
fit in an int, and the resolution will fail for those. There is no
support for unsigned integer types in Avro, which is itself a small
schema-language difference: schemas that need unsigned values
encode them as long and have the application enforce the range.

In the **schemaless formats**, type changes are an application
concern. The wire bytes for an integer carry just enough
information to decode the integer; the application interprets the
result.

## Scenario 6: Reorder fields

We want to declare `name` before `id` in the schema.

In **Protobuf**, this is purely cosmetic. The wire format is keyed
by field number, and field numbers are unchanged. Reordering the
declarations affects nothing. The bytes are identical.

In **Thrift**, same as Protobuf. Field IDs are what matter.

In **Avro**, this is *load-bearing*. Avro encodes records in the
order their fields appear in the schema. Reordering the
declarations changes the wire format. A producer with the new
schema and a consumer with the old schema will produce a
catastrophic mismatch unless schema resolution is in play, which
matches by name. Avro's resolution does match by name, and so a
field-reorder is *technically* compatible — but only if both
schemas are available to the consumer, and only if the resolution
engine matches the names correctly. The wire bytes are different.

The conclusion is that in Avro, the *schema* is what travels, not
just the bytes, and field order in the schema is significant.
Treating the schema as plain JSON and pretty-printing it with a
reorderer can produce wire incompatibility.

In the **schemaless formats**, reordering keys is permitted by
spec (map ordering is unspecified) and tolerated by typical
consumers. Deterministic-encoding requirements may impose a
canonical key order, but the format itself does not.

## Scenario 7: Change a field from optional to required

We want to make `email` required.

In **Protobuf 3**, this is impossible at the schema level (no
`required` keyword). The application enforces required-ness.
Switching from `optional string email = 3;` to a non-optional
`string email = 3;` is a wire-compatible change, but it changes
the API surface (the `has_email()` accessor disappears in newer
proto3) and the application semantics (default values become
indistinguishable from absence). This is the change Protobuf 3.0
made by accident and Protobuf 3.15 partially undid.

In **Thrift**, changing `optional` to `required` is wire-compatible
but operationally hazardous, as covered in Scenario 2. The change
must roll out producers-first.

In **Avro**, an optional field is a union with null and a default
of null. Making it required means removing null from the union.
This is *not* compatible: old data with the field absent (encoded
as the null branch) cannot be resolved against a reader's schema
where the field is a non-null type. The field has to remain
optional in the schema, and required-ness has to be enforced
elsewhere.

In the **schemaless formats**, the change is purely application-
side. The format is uninvolved.

## A summary table

| Scenario                          | Protobuf            | Thrift              | Avro                 |
|-----------------------------------|---------------------|---------------------|----------------------|
| Add optional field                | Trivial             | Trivial             | Requires default     |
| Add required field                | Not in proto3       | Producers first     | Requires default     |
| Remove field                      | Trivial w/ reserved | Trivial             | Field needs default  |
| Rename field                      | Source-only         | Source-only         | Requires alias       |
| Type widening (int → long)        | Wire-compatible     | Wire-compatible     | Resolution-promotion |
| Reorder declarations              | Cosmetic            | Cosmetic            | Wire-significant     |
| Optional → required               | Discouraged         | Hazardous           | Incompatible         |

## What the table actually means

The table is small enough to read quickly, and the differences are
real, but the right reading is not "which format has more 'trivial'
entries." Every format has roughly the same number of safe and
unsafe scenarios. The differences are in *which scenarios are safe*
and *what kind of failure happens when you make an unsafe change*.

Protobuf's failure mode for unsafe changes is usually silent: the
bytes decode, but the values are wrong. Field-number reuse with a
type change produces a decode that succeeds but yields garbage.
Type-incompatible changes within the same wire type (int32 → sint32)
produce values that look plausible but differ from the originals.
The remedy is `reserved` plus a breaking-change detector like Buf,
which catches these mistakes at schema-merge time.

Thrift's failure mode is mixed. Compatible changes work cleanly.
The `required` keyword turns some failures loud (the decoder
errors on missing required fields), which is helpful in some
deployments and harmful in others. There is no equivalent of Buf
for Thrift in widespread use, which means breaking-change
detection is mostly manual.

Avro's failure mode is *loud and early*. Schema resolution
failures happen at decode time and are explicit, with messages
that name the offending field. The Confluent Schema Registry
catches incompatible schema changes at registration time and
rejects them, which means many failures never reach a decoder.
The cost is rigidity: changes that are "harmless" in Protobuf or
Thrift (renaming a field, reordering declarations) require explicit
metadata in Avro.

The choice between formats is therefore a choice between *what
kind of evolution discipline you want enforced where*. Protobuf
asks for discipline at the human level (use `reserved`, run Buf in
CI). Thrift asks for discipline at the deployment level (sequence
your rollouts). Avro asks for discipline in the schema itself
(declare defaults, declare aliases). Each works; they just shift
the cost to different places.

## What about the schemaless formats?

MessagePack, CBOR, BSON, and the rest of the self-describing
schemaless family have no formal evolution rules. They make every
scenario "trivial" at the wire level, and the cost is paid
downstream: in application code, in operational coordination, in
tests that catch mistakes the schema would have caught.

For small teams, fast iteration, and schemas that change often
without strong deployment-skew constraints, this is fine. For large
organizations, slow rollouts, and schemas that need to stay
compatible across many independent versions, the lack of formal
rules is a chronic source of bugs. The right format is the one
where the operational pattern matches your organization's
deployment topology, and *operational topology* is the part of the
question almost nobody answers honestly when picking a format.

## A practical recommendation

If you are starting a new system and asking which schema-first
format to use, the right question is not which has the best wire
encoding (they are all comparable) but which evolution model you
can credibly enforce. If your organization has the operational
muscle to run a schema registry and check compatibility at
registration, Avro is the strongest choice and has aged
exceptionally well. If your organization runs Buf or an
equivalent breaking-change detector in CI, Protobuf is the
strongest choice and is by far the most common. If neither
infrastructure is in place, the schemaless options will produce
fewer surprises in the short run and more in the long run; budget
accordingly.

The one wrong answer is to choose a format on the assumption that
you will adopt the surrounding evolution infrastructure later.
Nobody adopts it later. The infrastructure ships with the format
or it does not ship at all.
