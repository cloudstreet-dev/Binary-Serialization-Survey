# Anti-Patterns

This chapter is a catalog of the ways teams misuse binary
serialization formats in production, drawn from observation of
many systems and from the consistent shapes of the resulting
incidents. The catalog is not exhaustive; new anti-patterns
emerge as systems evolve. But the patterns described here recur
often enough that recognizing them in your own systems is
disproportionately valuable. Each entry describes the pattern,
explains why it is tempting, and names the failure mode it
produces.

The patterns are presented roughly in order of frequency, with
the most common first. They are not equally severe; some produce
gradual technical debt, while others produce specific incidents
that page on-call. Both kinds matter.

## Reusing a field number

The single most common Protobuf anti-pattern is reusing a field
number that was previously assigned to a different field. The
pattern: a field is added with number 5, deployed, used for a
year, then removed when its purpose is obsolete. Some time later,
a developer adds a new field, looks at the schema, sees that 5
is "available" (not currently used), and assigns the new field
number 5.

Why it is tempting: the old field is gone, the schema looks like
5 is free, the developer wants the next sequential number rather
than skipping ahead. The schema *looks* fine.

The failure mode: production data written under the old schema
still has field-5 bytes in some buffers. Consumers under the new
schema will decode those bytes as the new field's type. If the
types are wire-compatible (string and bytes are both
length-delimited; varint and fixed64 are not), the decode
succeeds and produces garbage. If the types are wire-incompatible,
the decode fails with a parse error that sometimes looks like a
network corruption. Either way, the bug appears intermittently
and is hard to localize because the byte history of the field is
not visible from the schema.

The fix: the `reserved` keyword in Protobuf, the equivalent
discipline in Thrift, the schema registry's compatibility check
in Avro. Use Buf or its equivalent to detect this at PR time. The
problem is not in the format; it is in the lack of enforcement
infrastructure.

## Treating optional fields as required

The pattern: a field is declared optional in the schema (or, in
Protobuf 3 before the `optional` keyword was restored, scalar
fields are implicitly optional with default values), but the
application code reads the field and assumes it is always
present. Tests pass because the test data always has the field.
Production breaks when a producer omits the field.

Why it is tempting: defining a field as required makes the
schema document the invariant. But Protobuf 3 dropped `required`
for good reasons (Chapter 11), so there is no schema-level way
to express "this must be present"; the application is supposed to
enforce it. Many do not.

The failure mode: consumers crash, return wrong defaults, or
silently skip records when the field is absent. The producer
that omitted the field may be operating correctly per the
schema; the consumer is the one with the bug. But the bug
manifests at the consumer, and the diagnosis often blames the
producer.

The fix: enforce required-ness in application code at the
boundary. Document it in the schema's comments. If you are
crossing a schema boundary where required-ness is contractual,
use a format (Thrift, Avro with explicit non-nullable types) that
expresses required-ness at the schema level.

## Hashing non-canonical encodings

The pattern: an application needs to hash or sign serialized
bytes for caching, deduplication, or signature purposes. The
chosen format does not specify a canonical encoding, but the
application hashes the bytes anyway, on the assumption that
"the bytes are the bytes."

Why it is tempting: hashing bytes is cheap and obvious. The
nuance about canonical encoding is not always documented at the
top of the format's spec.

The failure mode: two producers in different languages (or
different versions of the same library) produce different bytes
for the same logical value. The hashes differ. Cache hit rates
collapse, deduplication breaks, signatures fail to verify across
producer changes. The bug typically appears after a library
upgrade or a producer migration, often months after the original
deployment.

The fix: use a format that specifies canonical encoding (CBOR
canonical, ASN.1 DER, Borsh, SCALE, SBE). If you must use a
non-canonical format, hash the *value*, not the bytes — typically
by canonicalizing first (sorting map keys, normalizing
representations) before serialization, then hashing.

## Choosing format based on benchmarks

The pattern: a team chooses a format based on a published
benchmark. The benchmark says format X is fastest. The team
adopts X. Production reveals that X is not fastest in their
specific workload, or that the speed advantage is dwarfed by
other factors.

Why it is tempting: benchmarks are concrete. They produce numbers.
Numbers feel objective. A 30% speed advantage in a published
benchmark is hard to argue with.

The failure mode: the published benchmark's workload differs from
yours. The format authors' implementations are tuned in ways the
benchmark exercises and your code does not. The compression
codec, the buffer sizes, the message shapes all matter, and the
benchmark fixed them to numbers that are not yours. The result
is a format choice based on a number that does not predict
production behavior, with the operational costs of the chosen
format paid in full while the speed benefit is partial or absent.

The fix: benchmark with your own workload, on your own hardware,
with your own access patterns, before making a decision.
Benchmark also after the fact, to verify the choice was right.
Treat published benchmarks as input to your hypothesis, not as
the conclusion.

## Re-encoding through JSON

The pattern: an application has data in some binary format. To
work with it, the application decodes to language-native objects,
encodes to JSON, parses the JSON, and uses the parsed result. The
JSON step is in the middle for no good reason, often as a
side-effect of using a JSON-centric library that happens to also
support the binary format.

Why it is tempting: JSON is what most application code expects.
Working in JSON-shape internally is comfortable.

The failure mode: every JSON round-trip loses something —
integer precision (above 2^53), byte strings (which JSON cannot
represent), distinct types (number vs. string vs. bool), and
floating-point edge cases. The application accumulates bugs at
the JSON boundary, often subtly. The performance is dramatically
worse than direct binary-to-binary work.

The fix: do not re-encode through JSON. Use libraries that
deserialize directly into your domain types from the binary
format, and serialize directly back out without an intermediate
text representation. This is more work in the short run and saves
much pain over time.

## Storing config in binary

The pattern: an application uses Protobuf (or any binary format)
for its inter-service messages. The same format is then used for
the application's *configuration files*. The config is now in
prototext or in binary; either way, it is not readable by `cat`,
not greppable by `grep`, and not editable by anyone without the
right tooling.

Why it is tempting: the format is already in the codebase. Using
it for config eliminates a dependency on a separate config
language. The schemas can be reused.

The failure mode: operations engineers cannot inspect the config
without specialized tools. Configuration changes are harder to
review (PRs show binary diffs or arcane prototext). Debugging a
production issue at 3 a.m. is harder. Migration to a new tool
chain becomes a project rather than a script.

The fix: use a text-based config language (YAML, TOML, JSON,
HCL) for human-edited configuration. If you want type-safe
configs, use a schema language whose binding generates the
config-validation code without forcing the config bytes to be
binary.

## Premature columnar

The pattern: a team chooses Parquet or Arrow for data that is
not columnar in any meaningful sense. Single-record events,
small JSON-shaped messages, transactional records that get
written and updated, are all stored in Parquet because "Parquet
is the modern standard."

Why it is tempting: Parquet has good ecosystem support. Tools
exist for everything. The team has heard that Parquet is
efficient.

The failure mode: Parquet's overhead — metadata, row group
boundaries, page headers, footer with statistics — dominates the
file size for small or transactional payloads. A million tiny
Parquet files use more storage than a million tiny JSON files
once metadata is counted. Reads are slower because the
metadata-driven access pattern that makes Parquet fast on
analytical queries does nothing for transactional ones. Writes
are awkward because Parquet is batch-oriented and individual
rows are not its unit of work.

The fix: pick the row-oriented format for row-oriented workloads.
Use Parquet for analytical workloads where columnar layout
genuinely matters, and not for everything else.

## The polyglot premature optimization

The pattern: a team is building a single-language application
(say, in Rust). They choose a cross-language format (Protobuf or
Avro) on the assumption that someday the system might need to
talk to a different language. That day does not come. The cost
of the cross-language format — the codegen step, the schema
files, the build complexity — is paid forever for benefits never
realized.

Why it is tempting: future-proofing feels prudent. Choosing a
cross-language format is the conservative choice.

The failure mode: the codegen and IDL machinery slow the team
down on every schema change. The advantages of the host
language's type system (strong types in Rust, classes in Java,
named tuples in Python) are not fully usable because the
generated types stand between the application and the data. The
ergonomics of the system suffer for a feature that is not used.

The fix: if cross-language interoperability is not a current
requirement, use the host language's native serialization
(rkyv or postcard for Rust, Java's serialization or Kryo for
Java, pickle or msgspec for Python). The cost of switching to a
cross-language format if needed is real but bounded; the cost of
using one when not needed is paid forever.

## Failing to archive the schema

The pattern: a team uses Protobuf, FlatBuffers, or Cap'n Proto
for archival data. The bytes are written to durable storage. The
schema is in a Git repository that gets reorganized, renamed, or
deleted years later. The bytes still exist; the schema does not.

Why it is tempting: the schema lives in the source tree. The
source tree feels permanent.

The failure mode: years later, the bytes are unreadable. The
team that wrote them has moved on. The Git repository has been
restructured. The compiler is several versions newer and
generates different code from the same schema. The data is, for
practical purposes, lost.

The fix: archive the schema *with* the bytes. Avro Object
Container Files do this automatically. Parquet does it via
the file footer. Other formats require explicit operational
discipline: copy the schema to a write-once archive every time
it changes, version the archive, document the recovery
procedure. The format will not remind you.

## Schema in code only

The pattern: a team uses a serde-binary format (bincode,
postcard, rkyv, or Java serialization) for their wire format.
The schema is the source code. There is no separate IDL, no
schema document, no contract artifact.

Why it is tempting: schema-as-code is convenient. The type
system enforces correctness within the language. Why have a
separate schema?

The failure mode: any non-source consumer (a debugging tool, a
data-pipeline migration, a forensic investigation, a different
language client added later) cannot decode the bytes without
having the source code. The source code is sometimes available;
sometimes the developer who wrote it has moved on; sometimes
the source has evolved and the old bytes are incompatible. The
schema is undocumented, and "the schema is the code" is a
sentence that means "the schema is uncomputable from the bytes."

The fix: if you want schema-as-code, accept that the format is
single-language and the bytes are not interpretable without the
source. If that is acceptable, the choice is fine. If it is not
acceptable, use a format with an explicit IDL.

## Map ordering assumed stable

The pattern: an application encodes a map (Java HashMap, Python
dict, Rust HashMap, Go map) into a format that does not specify
map ordering. The application then assumes the bytes are stable
across encodes — for testing, for caching, for hashing. Different
runs produce different bytes for the same data, and the assumption
fails.

Why it is tempting: in some language runtimes the iteration
order of a map is *almost* stable (Go's intentional randomization
notwithstanding); developers extrapolate from "almost stable" to
"stable" without verifying.

The failure mode: tests that compare byte outputs flake. Caches
keyed on hashes miss intermittently. Two replicas produce
different bytes for the same data, and reconciliation logic
trips.

The fix: use ordered map types (Rust's BTreeMap, Java's
LinkedHashMap or TreeMap, Python's dict in 3.7+ which is
insertion-ordered, sorted iteration when emitting) and document
the ordering as a contract. For deterministic encoding, sort
keys before emitting.

## Forgetting that compression sits underneath

The pattern: a team compares formats on uncompressed size and
chooses the smallest. The deployment uses compression at the
storage or transport layer. After compression, the size
differences are smaller or reversed.

Why it is tempting: format size is what marketing materials
publish. Compression is "an implementation detail."

The failure mode: the format choice was driven by uncompressed
density. The actual on-disk or on-wire bytes after gzip/snappy/
zstd are sometimes within a few percent across formats. The
format choice solved the wrong problem; the team paid the
operational cost of the format for a benefit that compression
would have provided anyway.

The fix: measure the post-compression size if compression is in
your stack. The format choice should be based on the bytes that
actually reach storage or wire, not the bytes the format
produces before compression.

## A meta-pattern: the wrong-question format choice

The patterns above are mostly specific. They have one thing in
common: each starts with the team using a format whose answer to
"what should you optimize for" does not match the question the
team is actually asking. Protobuf optimizes for cross-language
typed serialization with tagged-field evolution. If you are
asking how to encode your Rust types densely for a single-machine
cache, Protobuf is the wrong answer to the right question, even
though it would be the right answer to a different question.

The recurring fix is therefore to ask, before choosing or
defending a format, *what question is this format an answer to?*
Then ask whether that is the question your system is actually
asking. If it is not, the format is going to fight you, often
in ways that don't manifest until production data has accumulated.
If it is, the format will mostly cooperate.

This is the meta-pattern that ties the chapter together. Each
specific anti-pattern is a manifestation of choosing a format
whose intended question does not match the team's actual one.
The next chapter, on migration paths, describes what to do once
you have made one of these mistakes and need to walk back from
it.
