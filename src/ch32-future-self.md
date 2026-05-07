# What You Owe Your Future Self

You have decided to build a custom binary format. The previous
chapter argued you should not, and now we are past that argument.
This chapter is about what you owe your future self, and the
future engineers who will inherit your format, in terms of
operational discipline. The discipline is the difference between
a custom format that survives a decade and a custom format that
becomes the team's biggest liability inside three years.

The advice in this chapter is not technical. The technical
choices have been covered in the format chapters; you now know
the design space well enough to make them. The advice here is
operational: things to do once the format exists, that determine
whether it stays usable.

## Write the spec

The first thing you owe your future self is a written
specification. Not a comment in the source code. Not a wiki page
that drifts. A specification that describes the wire format
precisely enough that someone with no access to your source code
could implement an interoperable encoder and decoder.

A real specification covers, at minimum: the byte-level encoding
of every type your format supports; the framing rules that say
where one value ends and the next begins; the rules for handling
malformed input; the alignment, byte order, and endianness
conventions; the schema language (if any); the versioning and
evolution rules; the canonical-encoding rules (if any); the magic
numbers or signatures that identify your format's bytes.

The specification should be testable. Specifically, you should be
able to write a conformance test suite — a set of test cases
that any conforming implementation must pass — derived from the
specification. If the spec cannot generate a test suite, the spec
is not precise enough.

Specifications drift from implementations. The drift is the
single largest cause of "the spec says X, the code does Y, which
is right" debates that consume engineer-years. The mitigation is
to keep both in version control, in the same repository, with a
review discipline that requires changes to the implementation to
update the spec and vice versa.

The specifications for the formats in this book vary in
quality. Protobuf's spec is excellent. Avro's is good but has
gaps that the implementations have papered over. MessagePack's
is okay. NBT's is essentially community-maintained on the
Minecraft Wiki. The good specs make their formats survive. The
gappy specs produce implementations that disagree on edge
cases, which produces operational pain.

## Magic numbers and version bytes

Every byte sequence you produce should start with bytes that
identify the format. A four-byte magic number is the convention
for files; a single-byte tag is enough for in-memory uses. The
magic should be: short (you do not want to spend many bytes on
identification), distinctive (it should not collide with the
magic of other common formats), and stable (once chosen, it
should never change).

Examples worth emulating: Parquet's `PAR1` (four printable
ASCII bytes, easily greppable in hex dumps); the Cap'n Proto
buffer header (no magic per se, but the segment-count field at
position 0 is structurally distinctive); ASN.1's tag byte
distinctiveness for SEQUENCE (`0x30`, easy to recognize).

Following the magic, you should have a version byte. The version
should not start at 0 (so that someone seeing a zero byte after
the magic immediately knows something is wrong). Modern
conventions start at 1 and bump on every wire-incompatible
change.

A version byte is a contract: it tells future implementations
which decoder to use. Format authors who omit the version byte
because "we won't ever change the format" are wrong, and the
omission is the single regret most format authors voice in
retrospect. Add the version byte. The cost is one byte per
record. The benefit is the ability to evolve.

## The conformance test suite

A conformance test suite is the single most important durability
asset your format can have. It is a set of (input, expected
output) pairs that any conforming implementation must
reproduce exactly.

The pairs cover: encoding test cases (given a value, the bytes
your encoder should produce); decoding test cases (given bytes,
the value your decoder should reconstruct); malformed-input
cases (given bytes that violate the format, the specific error
the decoder should report); evolution cases (given bytes from
an old schema and a new schema, the value a new decoder should
produce); canonical-encoding cases (given a value, the unique
canonical bytes a deterministic encoder should produce).

The conformance suite is what lets you write a second
implementation, in a different language, and verify that it
produces the same bytes as the first. It is what lets you
upgrade the encoder and verify that the new version is
backward-compatible. It is what lets a junior engineer fix a
bug in the format and verify they did not break anything.

A conformance suite is not the same as a unit test suite. Unit
tests verify that individual functions in your encoder behave
correctly. The conformance suite verifies that *the bytes are
right*, by checking against an authoritative set of expected
outputs. The conformance suite should be checked in to source
control, in a format that can be consumed by any language
implementation, and updated only with care.

The formats that have lasted have conformance suites. Protobuf's
test data files are part of the open-source distribution. Avro
has test cases that any new implementation must pass. CBOR's
RFC includes a substantial appendix of test vectors. ASN.1 has
the IETF-maintained test corpus.

## Schema versioning, even if you don't think you need it

If your format has a schema (even an implicit one — every
format has a schema, the question is whether it is in a file or
in your head), version it explicitly. Schemas evolve. Pretending
they do not, and then evolving them anyway without version
markers, produces bytes that no decoder can interpret correctly
because the bytes' meaning depends on the schema version that
produced them.

The schema version can be embedded in every record (cheapest
when records are large; expensive when records are small),
embedded in the file or stream header (cheapest when records
are short and the stream is long), or embedded in the protocol
envelope (when the format is part of an RPC system). Pick one
and stick with it. *Multiple* schema-version mechanisms in the
same system are how you get bugs that take a year to
reproduce.

Document the schema-evolution rules. Specifically: which
changes are backward-compatible (new code reads old data),
which are forward-compatible (old code reads new data), and
which are breaking. Test the rules with the conformance suite.

## Decoder hostility

Your decoder will, at some point, be given input that it does
not expect. The input might be a corrupted file, a mid-flight
network error, an adversarial payload, or a bug in another
decoder that produced wrong bytes. The decoder must handle this
input gracefully — meaning, it must report the failure
explicitly rather than crashing, looping, or producing garbage.

Specifically, decoders should:

- *Validate every length prefix*: a string declaring 4 GB of
  content must not be allocated; the decoder must reject the
  input.
- *Bound recursion*: nested structures must have a maximum
  depth, configurable, with a sensible default.
- *Bound iteration*: any varint or length-encoded count must
  have a maximum, with a sensible default that depends on the
  field's expected use.
- *Reject extra bytes after a record*: trailing data after a
  fully-parsed record is suspicious; the decoder should report
  it rather than silently dropping it.
- *Handle alignment misses*: if your format requires aligned
  buffers and the buffer is unaligned, the decoder should
  report the misalignment, not produce undefined behavior.

These rules sound like security advice. They are. Custom formats
deployed in untrusted contexts are typically the source of CVE
findings several years after they go to production, and the CVE
is almost always about one of the rules above. Bake the rules in
from the start.

## The migration story

Plan how the format will change before it has changed. Specifically:
how will you add a field? how will you remove a field? how will
you change a field's type? how will you bump the format version?
What does a version-2 decoder do when it encounters version-1
bytes? What does a version-1 decoder do when it encounters
version-2 bytes?

The answers should be in the spec. They should be tested in the
conformance suite. They should be enforced by tooling that
prevents schema changes that violate the rules.

The fundamental discipline: *changes that break old data are
not allowed without a version bump, and version bumps are not
allowed without a coordinated migration plan*. Every change is
either compatible or it is a project. There is no third option.

## A note on tooling investment

Custom formats need tooling. Beyond the encoder and decoder
themselves, you will need: a hex viewer that understands your
format (essential for debugging); a schema linter (if you have a
schema language); a breaking-change detector (the equivalent of
Buf for Protobuf); language bindings for every language your
consumers use; integration with your build system; CI tests that
verify cross-implementation compatibility.

The tooling is not optional. It is the difference between a
format that is debuggable and a format whose users curse it.
Budget for the tooling at the same time you budget for the
format itself, and treat the tooling as part of the format's
specification.

The pattern that works: the canonical implementation is in the
language the team is most comfortable in; the spec is in a
neutral format (Markdown or HTML); the conformance suite is in
a language-neutral format (JSON or YAML, with byte data in
hex); the tooling lives in the same repository as the spec; and
new language bindings are required to pass the conformance
suite before being accepted.

## What you owe people who inherit your format

The team that inherits your format will be smaller, less expert,
and less invested in the format than you were. Your job is to
write enough down that they can be productive without rebuilding
your knowledge from scratch.

This is mostly the spec, the conformance suite, and the tooling.
But it is also the *rationale*. Every nontrivial design decision
should have a written justification: why this byte order, why
this varint encoding, why this evolution rule. The rationale is
what lets the next team change the format intelligently rather
than just preserving it as a black box.

The good open-source formats do this through release notes, RFC-
style design documents, and substantive commit messages. Custom
formats often do not, because the team doesn't think they're
publishing. They are publishing — to the next team — and the
next team will benefit from being treated as a real audience.

## Summary

What you owe your future self, in a list:

- A precise written specification, in version control, kept in
  sync with the implementation.
- A magic number and a version byte at the start of every
  byte sequence.
- A conformance test suite that any implementation must pass,
  in a language-neutral format.
- Explicit schema versioning, even when you think you don't
  need it.
- A hostile decoder that bounds resource use and reports
  malformed input clearly.
- A documented migration story, with rules for compatible
  changes and clear bumps for incompatible ones.
- Tooling investments commensurate with the format's deployment
  scale.
- A written rationale for every nontrivial design decision.

This list is the difference between a format that survives the
loss of its original author and a format that does not. The
investment is real; budget for it.

The next chapter walks through a worked example of designing a
custom format end to end, with the discipline above applied.

## A note on social maintenance

The technical disciplines above are necessary and not sufficient.
Custom formats also have *social* maintenance requirements that
are easy to underestimate.

Format ownership needs to be assigned, formally, to a specific
person or team. The owner is responsible for: triaging
implementation bug reports, reviewing changes to the spec or
conformance suite, mediating disputes about whether a particular
change is compatible, accepting new language bindings, and
documenting the format's release notes. Without a designated
owner, the format drifts: changes get made by whoever is
nearby, the spec falls behind, the conformance suite stops
running, and the format becomes a liability.

Format ownership is a job that needs slack in the team's budget.
It is not a 0% time activity. The minimum sustainable level is
probably 5-10% of one engineer's time, with spikes during
schema-change periods. Teams that nominate an owner without
allocating the time produce owners who are nominally in charge
of a format they cannot actually maintain.

The formats in this book that have survived all have ownership
arrangements that work. The formats that have not survived all
share the absence of such arrangements.

## A note on the path to publication

If you build a custom format that turns out to be useful beyond
your team, there is a question of whether to publish it. The
calculus has changed in the past decade. Open-sourcing a
serialization format used to be a meaningful contribution that
attracted users and validation. It is now a long-tail decision
with mixed outcomes: published formats accumulate users who file
bug reports, request features, and sometimes fork the project;
the maintenance burden grows beyond what one team can sustain;
the format either becomes a real community project or a
maintainer-burnout liability.

If you are going to publish, plan for the maintenance burden
explicitly. Document a clear governance model: who can merge
PRs, how the spec is updated, how breaking changes are decided.
The formats that have published successfully (Cap'n Proto,
Borsh, FlatBuffers) all have governance models that work; the
formats that have published unsuccessfully often had clear
technical merit but inadequate governance.

If you are not going to publish, the discipline in this chapter
is still required for internal use. The audience is just smaller.
