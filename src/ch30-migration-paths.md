# Migration Paths

The chapters before this have been mostly forward-looking: what to
choose, what to avoid, why one format suits a workload better than
another. This chapter is backward-looking. You have a system that
already uses some serialization format. The format is wrong for
where the system is going, or the format has accumulated technical
debt that has reached its boiling point, or the team has decided
to consolidate around a different format. You need to migrate. The
chapter describes what migration looks like in practice, the
patterns that work, and the patterns that look like they should
work and do not.

The single most important fact to internalize about format
migrations is that they are mostly an *operational* problem, not
a technical one. The wire-level conversion between any two
sensible formats is straightforward; tools exist for most pairs;
the encoding logic is, in raw lines of code, small. What is hard
is doing the conversion *while production is running*, against
existing data, with consumers and producers that cannot all be
upgraded simultaneously, and without breaking the availability or
correctness of the system during the transition.

## The four migration patterns

There are essentially four patterns for migrating from format A
to format B. Most real migrations are combinations of these.

**The rewrite.** Stop writing format A. Convert all existing data
to format B in a batch job. Restart writing as format B. This is
the simplest pattern but requires either downtime or a system
that can tolerate brief unavailability. Rewrites work for small
datasets, internal systems, and cases where a maintenance window
is acceptable.

**The dual-write.** Write to both format A and format B during
the transition. Consumers continue reading format A. New
consumers read format B. Once all consumers are migrated, stop
writing format A and delete the old data. This pattern requires
storage for two copies of the data during the transition and
disciplined consumer migration but is otherwise low-risk.

**The translate-on-read.** Continue writing format A. Add a
translation layer that reads format A and serves format B to
consumers that want it. Migrate consumers one at a time. Once
all consumers are migrated to expect format B, switch the
producer to write format B and remove the translation layer.
This pattern adds latency and operational complexity to the
read path during the transition but does not require dual
storage.

**The translate-on-write.** Switch producers to write format B.
Add a translation layer that converts format B back to format A
for consumers that have not yet migrated. Migrate consumers one
at a time. Once all are migrated, remove the translation layer.
This is the inverse of translate-on-read and has the inverse
trade-offs: migration cost is on the write path during transition.

The four patterns can be combined. A common combined pattern is
*dual-write with translate-on-read*: produce both formats during
the transition, but also expose a translation endpoint so that
emergency consumer needs can be served without waiting for the
official migration to complete.

## Migration scenarios

A handful of specific migrations are worth walking through, both
because they are common and because they illustrate the patterns
in concrete form.

### JSON to Protobuf

The most common migration. The team has been using JSON over
HTTP, has hit scale or cost limits, and wants to switch to
Protobuf over gRPC.

The pattern: dual-write. The producer continues to expose a
JSON HTTP endpoint while adding a Protobuf gRPC endpoint. The
schema for both is generated from the same Protobuf source (the
JSON encoding of Protobuf is well-defined; tools generate the
JSON-shaped REST API from the same `.proto` file). Consumers
migrate one at a time from HTTP/JSON to gRPC/Protobuf. The
JSON endpoint is removed once all consumers have migrated.

The traps: the JSON-encoding rules for Protobuf differ from
ad-hoc JSON conventions in subtle ways. A Protobuf int64 field
serializes to a JSON string by spec (because JSON cannot
represent 64-bit integers exactly), but most ad-hoc JSON
conventions emit them as numbers. If consumers were written
against the ad-hoc encoding, they break when migrated to the
spec-compliant Protobuf-JSON. The fix is to pick the JSON
encoding rules deliberately and document them; tools like
`protojson` make this manageable.

The other trap: consumers who depended on JSON's flexibility
(adding ad-hoc fields, sending strings where numbers were
expected, mixing types) discover the rigidity of Protobuf the
hard way. Each consumer migration becomes a small project
rather than a one-liner.

### Protobuf v2 to v3

The second most common. The team has Protobuf 2 schemas with
required fields, custom default values, extensions, and other
features that Protobuf 3 does not support directly.

The pattern: rewrite the schemas, with mechanical translation
where possible. Required fields become optional fields with
application-level enforcement. Default values are removed (or
expressed via the application layer). Extensions become explicit
embedded messages. The wire format is unchanged for compatible
fields, so old binaries continue to work; the schema-level
changes do not break existing data.

The traps: code that depended on Protobuf 2's required-field
enforcement breaks silently when the field is no longer
required. The fix is to add explicit checks in the application
layer at the same time as the schema migration.

The Protobuf 3.15 `optional` keyword for scalar fields changes
the migration story. Schemas that need explicit absence
semantics for scalars can use `optional` after the migration,
which restores the Protobuf 2 has-bit behavior.

### Avro to Protobuf

The opposite-direction analytical migration. A team using Avro
with Confluent Schema Registry decides to move to Protobuf for
better tooling alignment with their non-Kafka services.

The pattern: dual-write at the producer side, with both Avro
and Protobuf schemas registered for the same Kafka topics
(Confluent supports this directly). Consumers migrate one at a
time. The Avro schema is retired once all consumers are
Protobuf.

The traps: Avro's schema resolution semantics (the algorithm
that reconciles reader and writer schemas) does not map directly
onto Protobuf's tagged-field model. Fields with non-trivial
defaults in Avro must have the defaults reproduced in
application code on the Protobuf side. Renames that were handled
via Avro aliases need to become explicit field-rename
coordinations in Protobuf. The migration *can* preserve
compatibility, but the operational discipline differs from
what the team is used to.

### bincode 1.x to 2.x

A Rust-ecosystem migration. The team has on-disk data in
bincode 1.x format and wants to move to 2.x for the smaller
encoding and the better default behavior.

The pattern: translate-on-read with dual-write at write time.
The reader detects the format version (bincode 2.x has a magic
number; bincode 1.x does not) and decodes appropriately. New
data is written in bincode 2.x. Old data is migrated lazily
(rewritten when read, in 2.x format) or eagerly (a batch job
rewrites all existing files).

The traps: bincode 1.x had several configuration modes that
produced wire-incompatible output. Code that produced bincode
1.x with default settings will be readable; code that produced
1.x with VarintEncoding configured will be subtly different. The
migration must handle both.

### Row format to Parquet

A common analytical migration. The team has been writing
JSON-lines or CSV files for their analytical pipeline and wants
to move to Parquet for better query performance.

The pattern: dual-write at the producer if the producer can
support both, or batch-conversion at the storage layer. The new
files are Parquet; the old files are converted in chunks. The
analytical queries are updated to read Parquet, with the row
formats supported as a fallback for old data during the
transition.

The traps: the schema must be inferred or provided. CSV has no
schema; JSON-lines often does not either. Parquet requires one.
Inferring the schema from existing data sometimes produces
surprises (fields that were always integers but turn out to
have one record with a string; nullable fields that were never
null in the sample). Schema validation must happen during
conversion, with the surprises documented and fixed at the
data layer.

A second trap: Parquet's batching expectations. Writing one
Parquet file per record produces enormous metadata overhead.
The conversion job must batch records into reasonable row group
sizes, which sometimes forces architectural changes upstream
(streaming consumers must wait for batches; latency budgets
shift).

### Hessian to Protobuf

A Dubbo-ecosystem migration. The team is on Apache Dubbo with
Hessian and wants to move to Protobuf over Dubbo.

The pattern: Dubbo natively supports both formats. The
configuration is per-service. The migration is service-by-service:
update each service to support both Hessian and Protobuf,
update consumers to expect Protobuf, switch the producer to
emit Protobuf only, decommission the Hessian path.

The traps: Hessian preserves shared object references; Protobuf
does not. Code that depended on object-reference preservation
(typically code with deeply linked domain models) breaks when
migrated. The fix is to denormalize at the application boundary,
which is sometimes a substantial refactor.

### ORC to Parquet

A Hortonworks-lineage migration. The team is on Hive with ORC
and wants to move to Parquet, typically as part of a broader
move from Hive to Spark or to a cloud data warehouse.

The pattern: batch convert. ORC and Parquet both have mature
readers, and tools like `orc-tools` and `parquet-tools` can
convert between them with reasonable fidelity. New data is
written as Parquet; old data is converted on a planned schedule.

The traps: type fidelity at the boundary. ORC's int96 timestamp
encoding (deprecated) and Parquet's int64-timestamp-with-logical-
type are not byte-for-byte equivalent. Decimal precision
representations may differ. Nested type mappings require
attention, especially for unions and complex maps.

The second trap: existing query engines that expected ORC's
specific predicate-pushdown patterns may need tuning when moved
to Parquet. The performance after migration is sometimes worse
than before until the queries are adjusted to the new format's
strengths.

## Common cross-cutting concerns

Across all migration patterns, several concerns recur and are
worth listing explicitly.

*Schema versioning during migration.* If the schema is also
changing during the migration (new fields, removed fields), the
migration becomes substantially harder. Combining a format
migration with a schema migration is generally a mistake. Do one
at a time: format migration first, schema migration after, or
vice versa, but not both at once.

*Test data for both formats.* Production migrations require
test data in both the source and target formats. Generating
this from scratch is laborious; the cleanest source is to record
real production traffic and translate it. The translation logic
is also the migration logic, which is convenient.

*Monitoring during transition.* Track the percentage of
producers and consumers on each format. Track decode failures
on each side. Track the operational cost of the translation
layer (latency, error rate). The migration is in flight as long
as both formats are being produced or consumed.

*Rollback planning.* Migrations sometimes have to be rolled
back. The dual-write pattern makes rollback easy: stop writing
the new format, switch consumers back. The translate-on-read
pattern makes rollback harder because the producer has already
moved. Pick the pattern with the rollback story you can live
with.

*The long tail.* Migrations are easy at 0% and 100% and hard at
80%. The last consumer or last producer is always somewhere
specific that nobody remembered: a forgotten cron job, a
dormant region's failover replica, a script run quarterly by a
person who has since left. The migration is not done until
that consumer is migrated or formally retired. Plan for the
long tail; it is the largest part of the calendar.

## Migrations that should be avoided

Some migrations are not worth doing. The format choice was
defensible; the operational cost of changing it exceeds the
benefit; the team should redirect its attention to other
problems.

The clearest case is migrating between formats with similar
trade-off profiles. MessagePack to CBOR is rarely worth the
effort; the formats are essentially equivalent in most workloads.
Avro to Protobuf inside an analytical pipeline is rarely worth
it; the schema-evolution mechanisms differ but the wire-level
performance is comparable. ORC to Parquet inside a Hive
deployment is sometimes not worth it; the marginal gains do not
justify the project cost.

The case for migration is strongest when the existing format
has a real, observable cost — a recurring incident class, a
storage bill that compounds, a hiring constraint because nobody
knows the format — and the candidate format eliminates that
cost. The case is weakest when the migration is purely
aesthetic.

## A note on the cost of always-migrating

Some teams convince themselves that migrations are easy and
schedule them recurrently: "we'll migrate to the new format
every two years to stay current." This is a mistake. Migrations
are expensive in calendar time and in attention; recurring
migrations crowd out other work. The right cadence is "migrate
when the existing format has a cost that justifies the project,"
not "migrate on a schedule."

The corollary is that the right cadence for *choosing the
initial format* is "carefully, with an eye to the next decade."
Format choices are sticky. They should be made deliberately,
not casually, and they should be allowed to stay made for as
long as they continue to be defensible.

## A worked migration playbook

To make the patterns concrete, here is a worked playbook for a
hypothetical large migration: a team has a five-year-old Java
service stack using Hessian over Dubbo, and is migrating to
Protobuf over gRPC. The team is around 50 engineers, the data
flows through about a hundred internal services, and the
migration target is six months.

**Month 1: Baseline.** Document the existing Hessian schemas as
Protobuf `.proto` files. Set up the Buf-equivalent tooling for
the new schemas. Stand up gRPC infrastructure (load balancers,
service mesh integration, observability hooks) alongside the
existing Dubbo infrastructure. Identify pilot services that
will migrate first.

**Month 2-3: Dual-write rollout.** Update producer services to
support both Hessian and Protobuf. Configure Dubbo to expose
both protocols on each service. Pick three to five pilot
services for end-to-end migration; migrate their consumers to
gRPC. Monitor for decode failures, performance regressions, and
unexpected schema mismatches.

**Month 4-5: Bulk migration.** Roll the dual-write pattern out
to all services. Migrate consumers in batches, prioritized by
service criticality (most critical last, so failures appear in
less critical services first). Track the percentage of traffic
on each protocol per service.

**Month 6: Decommission.** Once all consumers are migrated, stop
writing Hessian. Remove the Hessian client libraries from the
build. Clean up the dual-protocol Dubbo configuration. Archive
the old Hessian schemas alongside any historical data that
might still need to be readable.

**Forever after: long-tail vigilance.** The forgotten consumer
will appear three months after the migration is "complete." A
quarterly cron job, a disaster-recovery script, an obscure
monitoring system. Each one costs a day or two to migrate. Plan
for this; budget the time as part of the migration.

The playbook is approximately what every successful Hessian-to-
Protobuf migration I have observed has looked like. Variants
exist (translate-on-read instead of dual-write for systems where
producer changes are expensive; faster timelines when the team
is smaller; slower timelines when the data flow is more
complex), but the structure is the same.

## Summary

Migrations are operational problems. Pick a pattern that
matches your tolerance for transition complexity (dual-write,
translate-on-read, translate-on-write, rewrite). Plan for the
long tail. Avoid migrating between similar formats. Avoid
combining format migrations with schema migrations. Monitor
during the transition. Have a rollback plan.

The book has now described the formats, the axes, the common
mistakes, and the migration paths. The remaining chapters cover
what to do when the right answer is to roll your own format,
and the appendices collect the wire-tour material that has been
distributed across the format chapters.
