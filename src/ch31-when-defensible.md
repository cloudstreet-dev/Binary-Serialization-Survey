# When It's Defensible

The rest of this book has been about formats other people built.
This part is about the case where you should consider building
one yourself, and what that costs. Building a serialization
format is unfashionable advice. The fashionable advice — *use
Protobuf, you fool* — is right roughly 95% of the time and
catastrophically wrong the other 5%, and the 5% includes some
of the most important systems people build. The question this
chapter answers is when you are in the 5%.

The short answer is *almost never*. The long answer is that
there are specific properties no off-the-shelf format provides,
and if your system needs one of those properties, an off-the-
shelf format will fight you for the rest of the system's life.
The cost of building your own format is high but bounded. The
cost of fighting an unsuitable format is low per-incident and
unbounded over time.

## The cases that justify rolling your own

A custom format is defensible when *all* of the following are
true: the existing formats fail to provide a property your
system requires, the property is load-bearing for the system's
correctness or performance, you have the engineering budget to
build and maintain the format for the system's lifetime, and the
team is committed enough to the format that they will not
abandon it after the original author leaves.

These conditions are conjunctive. Failing any of them produces
a worse outcome than just choosing an existing format and living
with its limitations. Many of the systems I have seen with
custom formats had only the first two conditions; the third and
fourth were assumed and turned out to be wrong, and the system
ended up either replacing the custom format under duress or
maintaining it badly forever.

The specific cases that have justified rolling your own, in my
observation:

**Hard real-time constraints.** A system with a hard real-time
deadline (a hardware-in-the-loop simulator, a control system, a
microcontroller-bounded protocol) sometimes cannot afford the
overhead of a general-purpose format's encoder/decoder, even
the zero-copy ones. The constraint is specific (this loop must
finish in 50 microseconds; the existing decoder takes 80) and
non-negotiable. A custom format tuned to the exact data shape
and access pattern can sometimes save the necessary cycles.

**A truly unusual data shape.** Most data is records-with-
fields. Some data is not. Sparse matrices, k-d trees, custom
graph structures, time-series with irregular sampling, voxel
grids — these have natural representations that the row-and-
column orientation of mainstream formats does not capture
gracefully. Forcing the data into a Protobuf or a Parquet
schema usually produces a format that is correct but has 5×
the natural representation's overhead. If the data is large
enough that the 5× compounds, a custom format saves real money.

**Specific cryptographic requirements.** Some cryptographic
protocols have format requirements that go beyond
"deterministic encoding." They require specific byte layouts to
match a published specification, particular ordering of
authenticated and encrypted fields, framing that lets a
verifier check signatures without parsing every byte. Building
the format from scratch lets the requirements drive the design.
Trying to retrofit those requirements onto a general-purpose
format often produces a format that is slightly wrong in ways
that take cryptographic auditing to discover.

**Hard latency at extreme scale.** When the deployment is large
enough, the per-request CPU cost of a serialization format
becomes a budget item. A 10% improvement at a billion requests
per second is real money in datacenter capacity. The case for a
custom format here is harder to make in 2026 than it was in
2010 — modern formats have closed most of the gap — but the
case is sometimes still real, and the systems that built custom
formats here in the past tend to keep them.

**Single-vendor formats with no expected interop.** A vendor's
internal format that will only ever be read by their own code,
on their own hardware, can sometimes justify a custom design
that maximizes ergonomics for the vendor's specific stack.
Apple's PLIST is one example of this; certain game console
formats are another. The lack of any need for interop removes
most of the cost of being non-standard.

**Embedded or constrained-device protocols.** When the device
has 2 KB of RAM and the protocol must fit in 32 bytes, the
existing formats' fixed overhead is sometimes too much. Some
LWM2M deployments, some smart-card protocols, and a few similar
contexts have justified custom formats that no other system
will ever need.

## The cases that look defensible and aren't

Several cases sound like they justify a custom format and don't.

**"We need it to be small."** Almost never justifies a custom
format. The compression layer underneath the format usually
closes the size gap; the wire-format choices that matter for
size are well-understood and present in the modern formats.
If you need bytes that are dramatically smaller than what
Protobuf or CBOR produce, you need either a domain-specific
encoding (varint with prior knowledge of the value range) or a
better compressor, not a custom format.

**"We need it to be fast."** Rarely justifies a custom format
in 2026. The mainstream formats have been tuned aggressively
over the past decade. A custom format you build over a
reasonable time budget will be slower than `prost` or
`fxamacker/cbor`. Speed claims for new custom formats almost
always come from comparing against poorly-tuned implementations
of the alternatives.

**"We have a unique set of constraints."** Often the constraints
are not as unique as the team believes. The unique constraint is
typically expressible as "I want format A's evolution model with
format B's speed and format C's tooling." This is wishing for a
format that does not exist, but the wish is not justification
for building one; the right answer is to pick the format whose
trade-offs are least painful for you.

**"We don't want a build-time dependency."** Sometimes
justifies postcard or bincode (Rust-only) over Protobuf
(needs codegen). Almost never justifies a *new* format. The
build-time dependency on `protoc` or its equivalent is a
known cost; trading it for the cost of maintaining a custom
format is usually a bad trade.

**"We want full control."** Usually a sign that the team has
not understood what off-the-shelf formats provide. Full control
also means full responsibility: every bug, every interop
problem, every migration is yours to handle alone. The
mainstream formats provide a community that handles many of
these issues, and giving up that community costs more than the
control is worth.

## The cost ledger

Building a serialization format has costs that are easy to
underestimate. A partial list:

*The encoder.* A few hundred to a few thousand lines of code
per language, plus tests, plus documentation. Initial cost: a
few engineer-weeks per language.

*The decoder.* The same shape of cost, perhaps slightly higher
because the decoder has to handle malformed input gracefully.

*The schema language and compiler.* If the format is schema-
required, the schema language must be specified, the compiler
must be built, the build-system integration must be designed.
This is months of work for a complete implementation.

*The test suite.* Including round-trip tests, malformed input
tests, schema-evolution tests, performance tests, and
cross-language interop tests if the format is multi-language.
A real test suite for a mature format is thousands of test
cases, accumulated over years.

*The documentation.* Specification, tutorials, examples, FAQ.
The level of polish needed for a format other engineers will
adopt is significant. Internal-only formats can have lighter
documentation, but even those need enough detail that a new
team member can be productive.

*The tooling.* Hex viewers, schema linters, breaking-change
detectors, language bindings beyond the initial set, IDE
plugins. Mature ecosystems have all of these. Custom formats
typically have none and will need them as the deployment grows.

*The maintenance.* Bug reports from users, edge cases that
weren't tested, language-binding contributions that need
review, schema-evolution rules that need clarification. The
maintenance cost is open-ended and continues for as long as the
format is in production.

*The opportunity cost of not investing in something else.*
Every hour spent on the custom format is an hour not spent on
the system's core problem. Custom formats are rarely the system's
core problem.

The cost ledger is not meant to be discouraging. It is meant to
be honest. Some systems pay this cost willingly because the
benefit is worth it. Most do not, and they pay it anyway, because
the cost was not estimated correctly at the start.

## A small case study

A team I observed (anonymized) built a custom binary format for
their internal RPC system around 2014. The justification was
"none of the existing formats are fast enough." The format was
a careful design with several legitimate cleverness: bit-packed
field tags, a specialized encoding for sparse maps, in-place
deduplication of strings via a per-message dictionary. The
encoder and decoder were a few thousand lines of C++; the
performance was indeed about 30% faster than the contemporaneous
Protobuf implementation in their workload.

By 2020, the format had become the team's biggest liability.
The original author had left. The schema language had not been
formalized; new fields were added by hand-editing wire-format
constants. The cross-language story (Python, Go, JavaScript)
had been delegated to teams who lacked the original author's
expertise and produced subtly different implementations.
Performance was no longer competitive with modern Protobuf
implementations because the custom format had not been tuned
for newer hardware.

The team migrated to Protobuf in 2022. The migration took six
months of focused effort and was expensive in calendar time.
The 30% speed advantage from 2014 was, in retrospect, a
multi-million-dollar liability when amortized over the
maintenance and migration costs.

This is the typical trajectory of custom formats that were
defensible at the time of their creation. The conditions that
justified the format change; the format does not adapt; the
team eventually pays to migrate. The systems that have *not*
gone through this trajectory tend to be ones with truly
unusual constraints (specific cryptographic requirements;
hard real-time bounds) where the alternative formats remain
unsuitable.

## A note on the formats in this book that started as custom

It is worth observing that most of the formats covered in
chapters 4-27 began as custom formats. Some team built them for
their own use, then realized other people had similar needs and
published the spec. Protobuf was internal to Google for years.
Thrift was internal to Facebook. Cap'n Proto came out of
Sandstorm. SBE came out of LMAX. NBT came out of Minecraft.
The line between "custom format" and "publicly-released format"
is not a property of the format; it is a property of the
publication decision.

This is the optimistic reading: if you build a custom format
that is genuinely good, and you are willing to invest in the
documentation and ecosystem, you might end up adding to the
catalog rather than haunting your team's technical-debt log. The
realistic reading is that the formats covered in this book are
the survivors. For every published custom format, there are
many that didn't survive — built, used briefly, abandoned. The
survivor bias is large.

## Summary

The case for rolling your own format is real but narrow.
Specific cryptographic requirements, hard real-time constraints,
genuinely unusual data shapes, and embedded contexts can
justify it. Most claimed justifications — speed, size,
flexibility, control — do not, because the existing formats
have closed the relevant gaps and the cost of the custom format
exceeds the benefit.

If you do decide to build, the next chapter is about what you
owe your future self: the operational discipline that
distinguishes custom formats that survive from those that
become liabilities.

## Position on the seven axes

Custom formats can occupy any cell in the seven-axis space, by
construction. The axes are still useful as design questions: when
designing your format, take a position on each axis explicitly,
document the position, and stick to it. Custom formats that
drift across the axes — adding self-description here, breaking
determinism there, supporting evolution one way and then the
other — produce the worst of all worlds.

The discipline of designing the format with the seven axes in
mind is the same discipline you would apply when reading another
format's specification. The axes are vocabulary; the format is
your specific application of the vocabulary.

## A note on the conjunctive nature of the conditions

Worth re-emphasizing because the conjunction is what most teams
get wrong. A custom format requires *all four* conditions: an
unmet property, a load-bearing requirement, a sustainable
budget, and committed team ownership. Failing any one of these
turns the format from a feature into a liability.

The most commonly missing condition is the fourth: committed
team ownership. The format is designed by a person who is
deeply interested in formats; that person leaves; the new team
inherits the format and treats it as a black box; the format
ossifies and gradually becomes unmaintainable. This pattern is
so common in industry that I have come to consider the fourth
condition the most important to evaluate. *Will the team
maintain this format for ten years* is a harder question than
it sounds, and "yes" usually requires either institutional
investment in format expertise or a format simple enough that
ordinary engineers can keep it running.

The second most commonly missing condition is the second: the
load-bearing requirement. Teams convince themselves that a
property is load-bearing when it is merely preferred. The clear
test: if you took the existing format and added the property
manually at the application layer, would the system be
sufficiently worse? If the answer is "it would be slightly
slower" or "it would be slightly less ergonomic," the property
is preferred, not load-bearing, and the cost of a custom format
exceeds the benefit.

The other two conditions — unmet property and sustainable budget
— are usually easier to evaluate honestly. Teams know whether
they have a property the existing formats lack, and they know
whether they have engineering hours available. The conjunctive
discipline of checking all four is what tends to be missed.

## A note on adopting an existing format with extensions

A common middle ground worth considering before going full
custom is to adopt an existing format and use its extension
mechanism for the parts that don't fit. CBOR's tag system,
Protobuf's `Any`, MessagePack's extension types, ASN.1's
`OCTET STRING`-of-anything: most mainstream formats have a way
to wrap arbitrary bytes inside a standardly-encoded outer
structure. The application layer interprets the wrapped bytes;
the format layer just carries them.

This pattern preserves the ecosystem benefits of the standard
format (libraries, tooling, debugging, schema-evolution discipline)
while letting the application handle the parts that genuinely
need custom encoding. The result is usually better than rolling
a fully custom format and almost always better than fighting a
format whose model does not fit.

## Epitaph

A custom binary format is a commitment to maintain it for as
long as the bytes exist; defensible when the alternative is
fighting an unsuitable format forever, but rare enough that
"build my own" should be the conclusion of an analysis, not its
premise.
