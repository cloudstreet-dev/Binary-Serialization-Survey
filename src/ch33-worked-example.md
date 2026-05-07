# A Worked Example

This chapter designs a custom binary format end to end, with
the discipline of the previous two chapters applied. The
example is deliberately small enough to fit in a chapter and
large enough to demonstrate the design process. The format we
will design is a *deterministic compact event log format for a
hypothetical embedded telemetry system* — a use case that has,
in this author's observation, occasionally justified custom
formats in the wild.

The exercise is not a recommendation that you should design
this format. It is a recommendation that, *if* you are going to
design a format, you should walk through the steps described
here.

## The use case

Suppose we are designing the on-device storage format for an
embedded telemetry device — a sensor module attached to a piece
of industrial equipment that records measurements once per
second. The device has 32 KB of RAM, a flash-based storage of a
few megabytes, no real-time clock (timestamps are sequence
numbers), and it talks to a central server via a satellite link
that bills per byte and is available a few minutes per day.

The records are: sequence number (uint32), six float32
measurements, an event-type discriminant (enum of 16 values), a
boolean fault flag, and a 1-byte status code.

The constraints derived from the use case:

- **Hard byte budget.** Each record must encode in a small,
  predictable number of bytes; uplink time is metered.
- **Hard CPU budget.** Encoding must complete in a few hundred
  cycles per record on a 50 MHz Cortex-M0.
- **Append-only writes.** Records are written once and never
  updated.
- **Random read access.** Field-service tools must be able to
  read individual records without scanning the whole file.
- **Multi-decade durability.** Records may be read by tools 20
  years after they were written.
- **No allocator.** The decoder runs on a microcontroller that
  cannot afford dynamic allocation.
- **Cryptographic signing of batches.** Each batch of N records
  is hashed and signed; the signature is verified at the server.

Several formats from earlier chapters are candidates. Cap'n
Proto and FlatBuffers are zero-copy, but their alignment and
metadata overhead is too high for the byte budget. Protobuf is
not deterministic (signing requires canonicalization). MessagePack
is not deterministic and does not give predictable per-record
byte sizes. CBOR with deterministic encoding is the closest
fit, but the per-record varint encoding wastes bytes for the
fixed-size fields, and the floats encode to 5 bytes (1-byte
prefix + 4-byte float) rather than 4 directly.

This is a case — not a common one, but a real one — where the
mainstream formats are close to right but not quite. A custom
format can save a few bytes per record, which over the device's
lifetime is meaningful.

## Apply the axes

Before writing any encoding rules, we take an explicit position
on each of the seven axes from chapter 2.

*Schema-required.* The schema is fixed in firmware; the device's
firmware version determines the schema. We do not need a schema
in the bytes.

*Not self-describing.* The records are decoded by tools that
have the schema available. There is no need for type tags.

*Row-oriented.* Each record is independently meaningful. We do
not need columnar layout.

*Parse rather than zero-copy.* The records are small enough that
parsing is cheap. Zero-copy alignment overhead is not worth its
cost here.

*Codegen-first.* The schema is small; the codegen is a script
that emits a C struct and a C decoder. No runtime reflection.

*Fully deterministic.* The signing requirement makes determinism
non-negotiable.

*Strict evolution rules.* Schema changes happen via firmware
upgrade. Multiple firmware versions may be deployed in the
field at once; the format must be able to identify which
version produced a record.

This is the point of the exercise: by going through the axes
explicitly, we have produced a *specification* for what the
format should do, before we have written any wire-format
details. The format is now a target with clear properties.

## The wire format

A record is exactly 32 bytes:

```
Offset  Size  Field                Description
0       1     format_version       0x01 for the format we are designing
1       1     firmware_version     0x00-0xff, matches the firmware that produced
2       4     sequence_number      uint32 little-endian
6       1     event_type           enum 0-15
7       1     status_code          uint8
8       1     fault                0x00 or 0x01
9       1     reserved             must be 0x00
10      4     measurement_0        float32 little-endian
14      4     measurement_1        float32 LE
18      4     measurement_2        float32 LE
22      4     measurement_3        float32 LE
26      4     measurement_4        float32 LE
30      2     measurement_5_low    low 16 bits of float32 LE (split for alignment)
0+30 ...
```

That last field is awkward; let's reconsider. A 32-byte record
with six floats forces some structural compromise. Two options:

Option A: pad to 36 bytes for natural float alignment.

```
Offset  Size  Field
0       1     format_version
1       1     firmware_version
2       4     sequence_number
6       4     measurement_0
10      4     measurement_1
14      4     measurement_2
18      4     measurement_3
22      4     measurement_4
26      4     measurement_5
30      1     event_type
31      1     status_code
32      1     fault
33      1     reserved (must be 0)
34      2     reserved (must be 0)
36              total
```

Option B: pack the measurements as float16 (8 bytes for six
measurements; precision sufficient for our sensor) and stay at
24 bytes:

```
Offset  Size  Field
0       1     format_version
1       1     firmware_version
2       4     sequence_number
6       2     measurement_0 (float16)
8       2     measurement_1
10      2     measurement_2
12      2     measurement_3
14      2     measurement_4
16      2     measurement_5
18      1     event_type
19      1     status_code
20      1     fault
21      3     reserved
24              total
```

The choice depends on the actual precision of the sensor.
Suppose float16 is sufficient. Option B saves 12 bytes per
record over Option A and lets the device store 50% more records
in the same flash budget. Option B it is.

The alignment is intentional: every multi-byte field starts at
a natural boundary (the float16s at 2-byte alignment, the uint32
at 4-byte alignment). The decoder on the microcontroller can
read fields with native loads.

## Batches and signatures

Records are grouped into batches. A batch header is 16 bytes:

```
Offset  Size  Field
0       4     magic (ASCII "TEL1")
4       1     format_version
5       1     batch_format_version
6       2     record_count
8       8     batch_sequence_number
16              records start here
```

After the records, a signature trailer:

```
Offset  Size  Field
0       64    Ed25519 signature over the batch header and records
64              total
```

The whole batch is therefore: 16 bytes of header + (24 bytes ×
record count) + 64 bytes of signature. For a typical batch of
600 records (10 minutes at 1 Hz), that is 16 + 14400 + 64 =
14480 bytes, or about 24.13 bytes per record amortized. The
satellite link cost per record is reduced from a CBOR-based
implementation by about 20%, which over a multi-year deployment
is real money.

## The specification

A real spec for this format would document:

1. *Magic number.* The four bytes `0x54 0x45 0x4c 0x31` (`TEL1`)
   identify a batch.
2. *Format version.* The single byte at offset 4 of the batch
   header. Currently `0x01`. Future versions bump this byte.
3. *Batch format version.* Reserved for batch-level layout
   changes. Currently `0x01`.
4. *Record format.* Exactly 24 bytes, layout as described above.
5. *Field semantics.* Each field's meaning, valid range, and
   encoding rules.
6. *Float16 encoding.* IEEE 754 half-precision, little-endian.
7. *Reserved fields.* Must be zero on encode; on decode, must be
   verified zero (otherwise reject the batch as corrupted).
8. *Endianness.* Little-endian throughout.
9. *Signature.* Ed25519 signature over (batch header || records),
   computed by the device's signing key, verified against the
   device's public key registered at the server.
10. *Determinism.* The format is fully deterministic. Given the
    field values, exactly one byte sequence is produced.
11. *Schema evolution rules.* Format version `0x01` is the
    initial version. To add a field, bump the format version
    and document the new layout. Firmware that does not
    recognize a format version must reject the batch and report
    the version mismatch.
12. *Decoder requirements.* Decoders must validate the magic,
    the format version, the reserved zero bytes, and the
    signature before exposing data to consumers.

The specification, written out, is several pages of plain
prose. It should live in version control, alongside the firmware
code.

## The conformance suite

The conformance suite for this format is small. Sample test
cases:

- Empty batch: header + zero records + signature. Decoder must
  accept; reader sees no records.
- Single record: known field values, known bytes. Encoder must
  produce these bytes; decoder must reconstruct these values.
- Wrong magic: bytes `XEL1...` instead of `TEL1...`. Decoder must
  reject.
- Wrong format version: bytes valid except offset 4 is `0x02`.
  Decoder for version 1 must reject.
- Non-zero reserved: bytes valid except a reserved byte is
  `0x01`. Decoder must reject.
- Bad signature: bytes valid except the signature trailer is
  random. Decoder must reject.
- Truncated batch: the bytes end mid-record. Decoder must reject.
- Specific field encodings: each field encoded with extreme
  values (zero, max, NaN for floats), with the expected bytes
  documented.

The suite is checked in as a JSON file: an array of test cases,
each with a description, input bytes (in hex), expected outcome
(value reconstruction or specific error code), and rationale.
Any new implementation of the format must pass every case.

## The implementation

The encoder and decoder, in C, are small. The encoder writes
fields at known offsets in a 24-byte buffer; the decoder reads
fields at known offsets. Both are a few dozen lines of code.

The signing logic uses Ed25519 from a vetted library
(libsodium, or an embedded equivalent). Key management is the
operational responsibility of the deployment, not the format.

A Python decoder, for the field-service tools, is similarly a
few dozen lines. A Rust decoder, for the central server, is
similar. All three implementations are validated against the
same conformance suite.

## What the team owes the future

The team designing this format owes its future self:

- The specification, written, in version control.
- The conformance suite, in a language-neutral format.
- A tool that takes a hex dump of a batch and prints the
  decoded fields, for debugging.
- A schema-versioning policy that documents how new format
  versions are introduced, deployed, and consumed.
- A migration plan for the inevitable case where the next
  generation of the device requires a slightly different schema.
- An owner with budgeted time to maintain the format.

The cost of all of this, for a format this small, is real but
manageable: maybe two engineer-weeks of initial investment, a
few hours per quarter of maintenance, a defined fallback to a
different format if the current design hits a hard limit. The
benefit is a format that does its job for the device's
deployment lifetime without becoming a maintenance burden.

## Why this exercise was worth doing

The walkthrough above looks small, and the format is small.
The exercise was worth doing for a reason that is easy to miss:
*every step of the design corresponds to a chapter or section
of this book.* The use-case analysis came from chapter 1's
boundary discussion. The axis-by-axis position came from
chapter 2. The byte-level layout was informed by the wire
tours in the format chapters. The evolution rules came from
chapters 11 and 30. The discipline of the spec, conformance
suite, and ownership came from chapter 32.

A team that designs a custom format without doing this exercise
typically ends up with a format that is correct but lacks one
or more of these pieces, and the missing pieces are what later
become liabilities. The book has been a long argument for
treating serialization formats as designs that deserve
deliberate attention, and this chapter is the demonstration
that the deliberate attention is, in fact, possible.

## A final note

The format we designed is hypothetical. The hypothetical use
case is one I have seen real teams build versions of, with
varying success. The teams that succeeded did the steps above.
The teams that failed skipped them, usually because the format
felt small and they thought they could do it from memory.

Format design is one of those activities where the discipline
is the work. The technical decisions are usually small; the
operational follow-through is what determines whether the
format survives. If you take one thing from this part of the
book, take that.

The next section — the appendices — collects the wire-tour
material from the format chapters into a single side-by-side
reference, so that you can see the same Person record encoded
across every format in this book at a glance.

## A note on what this exercise deliberately did not include

The example above did not encode our Person record. That was a
deliberate choice. Person is the exemplar for the existing
formats — a record with optional fields, variable-length
strings, a list of strings, a boolean. The hypothetical
telemetry format we designed has none of those shapes. Person
in this format would require expanding the format substantially
(variable-length string encoding, optional fields, list
encoding), at which point the format would be reinventing CBOR
or MessagePack badly.

This is a useful reminder: *custom formats earn their place by
being the right answer to a specific question*. The telemetry
format is the right answer for fixed-record-shape, byte-budget-
constrained, deterministic-signed embedded telemetry. It is the
wrong answer for variable-shaped business records like Person.
Trying to use it for Person would produce a worse format than
either MessagePack or Avro.

If you find yourself extending your custom format to handle a
shape it wasn't designed for, the right move is usually to
declare that shape outside the format's domain and use a
different format for it. The custom format covers its niche;
the rest of the system uses something else.

## A note on the relationship between this book and your judgment

Thirty-three chapters and four appendices is a lot of material
about serialization formats. The book has tried to be honest
about what each format is for, where it succeeds, where it
fails, and how to choose between them. None of the chapters tell
you what to do; they describe what is possible and what the
trade-offs are. The decision is, and remains, yours.

The book's principal claim is that *the decision is worth making
deliberately*. The format choice will outlast most technical
decisions you make in your career. The systems built on a format
inherit its constraints, and the constraints are not always the
ones the format's marketing materials advertise. Reading the
spec, walking the bytes, mapping the format onto the seven
axes: these are activities that pay back many times over the
modest time they take.

If you do that, the book has done its job. The next time
someone in your organization says "we use Protobuf" or "we use
Parquet," you will know what is being claimed, what is being
omitted, and what questions to ask before assuming the
sentence ended the conversation.
