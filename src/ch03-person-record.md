# The Person Record

The exemplar used for the rest of the book is a single record describing
a person. The values are fixed; every chapter that follows encodes
exactly these values. The choice of values is not aesthetic. Each one
was selected to land at a particular point in the encoding space of the
formats under discussion, so that the differences between formats become
visible in the bytes rather than hidden behind padding.

This chapter exists to fix the record once, in format-neutral terms,
so that subsequent chapters can refer back to it without
re-introduction. There is no wire tour here — this is the wire tour's
input.

## The fields

The record has six fields, declared in this order:

```
id          uint64            42
name        string            "Ada Lovelace"
email       optional string   "ada@analytical.engine"
birth_year  int32             1815
tags        list<string>      ["mathematician", "programmer"]
active      bool              true
```

`id` is a 64-bit unsigned integer. The value 42 was chosen because it
is small enough to fit in a single byte under any reasonable variable-
length encoding (Protobuf varint, MessagePack fixint, CBOR immediate),
and yet large enough that the formats which encode integers in fixed
8-byte words (BSON, XDR, Borsh, fixed-width bincode) will pay the full
eight-byte tax. Comparing the encoding of `id` across formats is the
clearest single demonstration that variable-length encoding is a real
trade and not an academic one.

`name` is the twelve-byte UTF-8 string `Ada Lovelace`. Twelve bytes is
short enough that length-prefixed formats can encode the prefix in a
single byte (or smaller), and long enough that the bytes themselves
dominate the per-field overhead. Every byte of the string is in the
ASCII range 0x20–0x7e; there are no multi-byte code points, no
ambiguous normalization questions, no zero-width joiners, no
right-to-left marks. This is deliberate. The book is not about
Unicode pitfalls; if it were, the exemplar would include them.

`email` is an *optional* string. Its value, when present, is the
twenty-one-byte UTF-8 string `ada@analytical.engine`. The optionality
is the load-bearing part of the field. Formats differ enormously in
how they represent "this field is present" versus "this field is
absent" versus "this field has its default value." Some formats
(MessagePack, CBOR, BSON, JSON-shaped formats) make the question
trivial: present means present, absent means the key isn't in the
map. Some (Protobuf 3) make it surprisingly hard, because the wire
format originally collapsed "absent" and "default" into the same
encoding for scalar types, and only later restored the distinction
for strings via the `optional` keyword. Some (Avro) require an
explicit union with `null`. Some (Borsh, SCALE, Postcard) have an
`Option` type with a discriminant byte. Some (NBT, the Apache
Cassandra family) cannot natively express "field is absent" at all
and require an in-band convention (empty string, sentinel value).
Each chapter encodes Person twice when the format's behavior
differs based on `email`'s presence: once with the email and once
without.

`birth_year` is a 32-bit signed integer with the value 1815. The
choice of 1815 over, say, 1985 is not arbitrary. 1815 fits in eleven
bits and therefore in two bytes of a varint encoding, while 1985 also
fits in two bytes. But the more important reason for picking a
historical year is that the variable-length encoders treat it the
same way as any other small positive integer; nothing about being a
date affects the bytes. Date semantics live in a higher layer, and
the formats that have a *date type* (Ion, BSON, CBOR via tag) are
explicit about the choice; for a plain integer, the encoding is just
an integer. Birth year is stored as `int32` rather than `uint32`
because the formats that distinguish signed and unsigned do so in
ways that matter (zigzag versus straight varint, in particular), and
making the canonical type signed exercises the more interesting code
path.

`tags` is an ordered list of two short strings: `mathematician` (13
bytes) and `programmer` (10 bytes). The list has two elements, which
is small enough that all surveyed formats encode the count in a single
byte. The two elements have different lengths, which means the
flat "concatenation of equally-sized values" trick that some formats
support for scalar arrays is not applicable; the format must encode
each element's length individually. Strings rather than integers were
chosen for the list elements because string-of-strings is the case
that exercises the most format machinery: every variable-length
encoding rule applies, the encoder has to decide whether to share
length-prefix overhead, and the columnar formats (Parquet, ORC, Arrow)
have to encode dictionaries, definition levels, and offsets all at
once. An array of integers would have demonstrated less.

`active` is a boolean with the value true. Booleans are deceptively
varied across formats. Some encode them as a single dedicated byte
(MessagePack 0xc3, CBOR 0xf5, BSON 0x01, JSON token `true`). Some
fold them into the tag of an enclosing structure (Thrift Compact uses
type code 1 for true and 2 for false, with no payload byte at all).
Some encode them as a single bit in a packed bitmap (Apache Arrow's
validity bitmap is the obvious case, but boolean arrays use the same
structure). Some require explicit DER-style encoding where 0xff is
canonical for true (ASN.1). Comparing the boolean encoding across
formats is a microcosm of the format's overall philosophy.

## What the record is not

It is worth being explicit about what the exemplar omits, because
several common features have been left out deliberately.

There are no nested records. A Person does not contain an Address,
which does not contain a list of GeoCoordinates. The reason is not
that nested records are uninteresting — they are very interesting,
especially for the columnar formats — but that adding nesting would
make the wire tours twice as long without illuminating axes that the
flat record fails to illuminate. Where a format's nesting story is a
material part of how it works (Cap'n Proto's pointer arithmetic,
Parquet's repetition and definition levels, Arrow's struct columns),
the chapter for that format includes a sidebar with a nested example.

There are no floating point fields. Floating point is its own subject:
NaN bit patterns, denormals, signaling vs. quiet, IEEE 754 vs.
implementation-specific extensions, and the question of whether the
format preserves the exact bit pattern or only the value. Including a
float field in the exemplar would force every chapter to address the
question, and the question is mostly the same across formats (they
all use IEEE 754 double-precision, with minor differences in how NaN
is handled). Where a format does something interesting with floats,
the chapter mentions it.

There are no maps with arbitrary keys. A `tags` *list* exercises
arrays; a `metadata` map with string keys would exercise the map
encoding. Most formats handle maps fine, but a few (Protobuf 2,
FlatBuffers without scaffolding) do not have native maps and require
a list-of-pairs convention. Including a map field would force a
discussion that is mostly orthogonal to the format's core design.
Each chapter notes the format's map story without encoding one.

There are no binary blob fields. Bytes-as-bytes is a fine field type
in most formats, and most formats handle it sensibly. Including a
blob would inflate the wire tours without showing anything not
already shown by the string field.

There are no enums. Enums are interesting precisely because formats
disagree about whether unknown values are forward-compatible (Protobuf
3 says yes; Thrift's older code generators said no), but the
disagreement is small and well-understood. Discussion goes in the
schema evolution chapter, not in every individual chapter.

There are no recursive types. A Person does not contain a list of
Persons. Recursive types are beyond the scope of the exemplar; they
appear in the FlatBuffers and Cap'n Proto chapters in passing.

There is no field marked deprecated, reserved, or removed. The field
list is what it is; the chapter on schema evolution simulates removing
`email` and adding a `country` field, but the canonical encoding shown
in each format chapter uses the current schema only.

## Two encodings, when relevant

For formats whose behavior differs materially based on `email` being
present versus absent, both encodings are shown. The difference is
usually small — a missing tag, an absent map key, a null branch tag —
but the difference is exactly the part of the format that handles
optionality, and it is worth showing. For formats where the
difference is uninteresting (a key that's just not in the map; a
list element omitted), only the present-email case is shown and the
absent case is described in prose.

The full email value is `ada@analytical.engine`. The domain is
spurious; Ada Lovelace did not have an email address. The value was
chosen because it is exactly twenty-one bytes long, which produces
encodings that are short enough to fit on a single line in most
hex viewers and long enough to demonstrate the length-prefix
mechanism in formats that vary their prefix size based on the
encoded length.

## Why this exemplar will sometimes feel cramped

A single Person record is small. Some of the formats in this book —
Parquet, ORC, Arrow IPC, the streaming column formats — are designed
to encode billions of records, and showing a single record's encoding
in those formats is, frankly, ridiculous. The bytes you get are
dominated by header overhead and metadata that would amortize away
across a real workload.

Where this is the case, the chapter shows the encoding of a single
record honestly — including all the overhead — and then, in a separate
sidebar, shows the per-record cost projected over a million records.
The single-record encoding is the apples-to-apples comparison for
small payloads; the projected cost is the apples-to-apples comparison
for the workloads the format was actually designed for. Both numbers
are useful. Reporting only one of them is dishonest.

## A note on byte order, alignment, and word size

Throughout the book, hex bytes are shown in network order — that is,
the order they appear in the wire — even when the format is internally
little-endian. This means that for a little-endian format like BSON,
the bytes `94 00 00 00` shown at the start of the document are the
literal bytes you would see in a hex viewer, and the value they
encode is the 32-bit little-endian integer 148. Where this might
surprise the reader, a parenthetical follows the hex.

Alignment is annotated where it matters. FlatBuffers, Cap'n Proto,
Apache Arrow, and SBE all have hard alignment requirements, and the
hex tours for those formats include the padding bytes explicitly so
that the alignment is visible.

Word size assumptions in this book follow the formats themselves: a
"word" in Cap'n Proto is 8 bytes, a "word" in CBOR's spec is 4 bits
(but bytes are bytes), and a "word" in conventional CPU parlance is
ignored. When precision matters, the chapter specifies.

That is the record. Every byte tour in the rest of the book is an
encoding of these six fields, in this order, with these values. When
the bytes diverge across formats — and they diverge enormously — the
divergence is not because the input changed. The input is fixed. The
divergence is the design.
