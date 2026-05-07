# What Serialization Is, and Why "Binary" Is a Category

Serialization is the process of turning a value that exists in a program's
memory — a struct, an object, a record, a tree — into a sequence of bytes,
in such a way that the same value can later be reconstructed from those
bytes by some other program, or by the same program at a later time, or
even by the same program in the same instant after the bytes have made an
unsafe trip through a buffer somewhere. Deserialization is the reverse.
The pair is sometimes called marshaling and unmarshaling, sometimes
encoding and decoding, sometimes pickling and unpickling, sometimes
something language-specific and embarrassing. The names don't matter.
What matters is that for the duration of the byte sequence's existence —
its trip across a wire, its sojourn on a disk, its passage through a
queue — the bytes *are* the value. There is no other authoritative
representation. If they are wrong, the value is wrong. If they are
ambiguous, the value is whatever the next reader decides it is.

This is the part that often gets glossed over. A serialization format is
a contract between writers and readers about what bytes mean. Treat it
as anything less and you will eventually pay for the misunderstanding.

## Three operations, not two

Encoding and decoding are the two operations everyone thinks of, but
there is a third that earns its keep: identification. Given an arbitrary
sequence of bytes — say, the contents of a file, or a frame off a socket —
*is this even something I should attempt to decode?* In practice the
answer comes from outside the bytes (the file extension, the MIME type,
the channel the bytes arrived on). But many formats also build in their
own answer, in the form of a magic number, a version byte, a framing
header. A four-byte `PAR1` at the start of a Parquet file. A two-byte
`PK` at the start of a ZIP archive. A `0xc0` introducing a CBOR self-tag.
These bytes do no useful information-carrying work. They exist solely
so that the wrong reader, encountering the wrong bytes, fails fast and
visibly instead of producing garbage.

Most binary formats include some identification mechanism. Most text
formats do not, because text is identified by the fact that it is text,
which is itself a kind of identification — not a precise one, but enough
to disqualify a binary blob without needing to read its first sixteen
bytes. The asymmetry is one of the reasons "binary" is a useful
category in the first place.

## Three boundaries, not one

The same word — *serialization* — covers three different problems, and
formats that excel at one of them often perform badly at the others.

The first is the **process boundary**. Two parts of the same running
program, or two cooperating processes on the same machine, exchange a
value. The bytes never leave the machine, often never leave a single
shared memory segment. The relevant cost is encode and decode time, not
size. The relevant question is whether the format can be made nearly
free in the common case. Zero-copy formats (FlatBuffers, Cap'n Proto,
rkyv) are designed for this boundary. So is the in-memory layout of
Apache Arrow.

The second is the **machine boundary**. Bytes leave one host and arrive
at another, possibly running on a different architecture, a different
operating system, a different language runtime, or a different version
of the schema. Now byte order matters. Floating point representation
matters. The relative cost of a microsecond of CPU and a microsecond of
network changes. The format must be portable. It usually must also be
versionable, because the two ends are deployed on different schedules.
This is the boundary that Protobuf, Thrift, Avro, MessagePack, CBOR,
and most of the formats you have heard of were designed for. The
language is "wire format" because the wire is the metaphor.

The third is the **time boundary**. The bytes are written today and
read in five years. The reader is not just on a different machine — it
is a program that has not yet been written, by a person who has not yet
been hired, in a language that may not yet exist. This is the boundary
where archival formats (Parquet, ORC, Avro Object Container Files,
ASN.1 BER) earn their pedigree. It is also the boundary where formats
without a strong evolution story tend to fail catastrophically without
warning. A bytes-on-disk format that does not embed enough information
to be decoded twenty years later by a stranger is a time bomb. The
question to ask is not *can I read this today* but *who can read this
when I cannot*.

A format can be excellent for one boundary and disqualifying for
another. FlatBuffers is exquisite for the process boundary and
serviceable for the machine boundary, but I would not choose it for
the time boundary unless I was prepared to also commit to archiving
the schema and a working compiler for the rest of my life. CBOR is
fine for any of the three but optimal for none. Knowing which boundary
you are actually crossing is the first decision; *most arguments about
formats are people debating across different boundaries without
realizing it*.

## What "binary" actually means

The word *binary* is doing a lot of work, and it does not always do it
honestly. Let's get the easy mistakes out of the way.

Binary does not mean *compressed*. Many binary formats are not
compressed in any nontrivial sense. Protobuf is dense but not
compressed; you can usually shrink a Protobuf message further by
gzipping it. FlatBuffers is barely compressed at all — it intentionally
includes padding so that fields can be addressed by offset. Conversely,
text formats can be compressed: gzipped JSON is smaller than ungzipped
CBOR for almost any realistic payload. Compression is orthogonal to
the binary/text distinction.

Binary does not mean *fast*. A poorly written binary parser will lose to
a well-tuned JSON parser. The fastest JSON parser I am aware of can
exceed a gigabyte per second on commodity hardware; few binary parsers
clear that bar. Speed is a property of an implementation, not a format.
What a binary format gives you is a *higher ceiling* on speed, because
parsing structured numbers is faster than parsing decimal strings, and
because length-prefixed lengths are faster to scan than delimiter
hunting. The ceiling is real. Reaching it is work.

Binary does not mean *opaque*. The bytes of a Protobuf message are
fully understandable to anyone with the schema. Even without the
schema, the wire format is regular enough that you can usually
reconstruct most of it. *Security through obscurity* and *binary
encoding* are not the same thing, and conflating them has produced
many bad architectural decisions.

Binary does not mean *typed*. Some binary formats (FlatBuffers, SBE)
encode strict types in the bytes. Some binary formats (MessagePack,
CBOR) encode type tags but are otherwise schemaless. Some binary
formats (XDR, raw Go gob) lean on an external schema. JSON, despite
being text, has a richer type system than some self-describing
binary formats — though a more limited one than many.

So what does binary mean? Operationally, it means: *the format is not
designed to be readable by a human without tools.* Bytes outside the
printable ASCII range appear deliberately. The output of `cat` is
gibberish. `grep` and `sed` cannot be trusted on the payload. A
junior engineer cannot tell at a glance whether the encoded value is
right. To inspect or modify the bytes, you reach for a parser, a hex
viewer, or a format-specific debugging tool. *That* is the category.
Everything else — speed, density, schema requirements, evolution
strategy — varies enormously within the category.

This framing makes the choice between binary and text more interesting,
because it foregrounds the actual cost. Choosing binary means choosing
to always carry the parser around. Always: at debug time, at log
inspection time, at "we have a production incident at 3 a.m. and a
junior engineer needs to understand this" time. The parser has to
exist for every language and tool that needs to read the bytes. The
parser has to be available offline, in environments without
dependencies, on hardware that may not have the bandwidth to download
a 30-megabyte SDK. These costs are real. Sometimes they are worth
paying. Sometimes they are not. The point is that they are *paid*,
and a format that does not acknowledge them is selling you only half
of the trade.

## Where binary earns its place

Binary formats are not a default. They earn their place in a system,
and the cases where they earn it are surprisingly narrow once you
write them out.

The clearest case is **density at scale**. If you are storing a
trillion records, the difference between forty bytes per record and
two hundred bytes per record is the difference between a $X
storage bill and a $5X storage bill, and the cost of the parser is
amortized to zero. Most analytical workloads — data lakes, time
series databases, message queues with retention — are here. Parquet
won this category not because it is fast (it is, but that is incidental)
but because column-oriented binary encoding makes the storage bill
go down by an order of magnitude.

The second case is **type fidelity**. A 64-bit unsigned integer is a
distinct value from the string `"18446744073709551615"`, and many
text formats lose this distinction in transit. JSON has a famous
problem with integers above 2^53. CSV has no concept of types at all.
If your data includes nontrivial numeric values, NaN-bearing floats,
binary blobs, or precise timestamps, a text format will, at some point,
quietly mangle one of them. A binary format with a type system encodes
the value in its full precision. This is not a speed argument. It is
a correctness argument.

The third case is **bandwidth-constrained or latency-constrained
links**. Embedded systems, satellite links, mobile radio protocols,
financial market feeds. Here the cost of a byte is high, and the
cost of CPU is fixed in advance because the hardware is fixed. ASN.1
PER, SBE, FlatBuffers, and the various format families used in
telecom and finance live in this case. Most readers of this book
will never work in this case. The handful who do tend to know it
already.

The fourth case is **read-heavy in-memory access**. If a value will
be read many times for every time it is written, and if reads happen
in performance-sensitive code, a format that allows access without
parsing is a real win. This is the FlatBuffers and Cap'n Proto and
rkyv pitch. The bytes on disk and the bytes the program reads are the
same bytes; there is no decode step. This is not free — it constrains
the format heavily — but where it applies, it applies decisively.

## What happens when the bytes are wrong

A surprising amount of a format's character is revealed by what it does
when given bytes that aren't quite right. Pure speed and pure size are
easy to compare. Failure semantics are not, and they are where the
formats differ most consequentially in production.

Consider four kinds of "wrong." There are bytes that are *truncated*:
the writer was interrupted, the network dropped a packet, the file
was copied incompletely. There are bytes that are *corrupted*: a
single bit flipped on disk, a buffer was overwritten, a protocol
stack stripped or added something. There are bytes that are *valid
encodings of unexpected schemas*: a producer was upgraded before a
consumer, an unknown field appeared, an enum value turned up that
nobody had heard of. And there are bytes that are *adversarial*: a
malicious or careless party constructed a payload designed to confuse
the parser, allocate too much memory, recurse too deeply, or trigger
a parser bug.

Different formats handle these four cases very differently, and the
choice is rarely advertised on the front page of the spec. Length-
prefixed formats fail loudly on truncation: the prefix says ten bytes,
you got six, the parser stops. Delimited formats may not notice
truncation at all if the truncation happens at a record boundary;
streaming Protobuf-over-HTTP has had years of subtle bugs caused by
exactly this. Self-describing formats with type tags catch some kinds
of corruption — the tag byte says "string" but what follows isn't
valid UTF-8, so something is wrong — but they catch single-bit flips
in the *middle* of a string only if a higher layer is doing
checksumming. Schema-required formats can fail in surprising ways
when an unknown field appears: Protobuf 2 required you to declare
which fields were required, and decoders were obligated to fail if
they were missing, which is why Protobuf 3 removed the concept; in
Avro, an unknown field in a record fails decoding outright unless
the reader's schema declares a default; in MessagePack, an unknown
field is just a key in a map and is silently passed through.

Each of these is a defensible position. *Each is also a position*:
your application is going to inherit it whether you noticed or not.
A format that silently drops fields it doesn't recognize is one that
will allow your producers to add fields without coordination, and one
in which a typo in a field name will silently throw your data away. A
format that hard-fails on unknown fields will catch the typo, and will
also turn every schema deployment into a coordinated dance. Neither
is wrong; neither is free.

Adversarial input deserves special mention. Most binary formats were
designed in environments where the writer and reader were trusted
peers, and as a result almost every format has had a memory exhaustion
bug — a length prefix claiming a four-gigabyte string, a recursion
limit easy to defeat, a tag that triggers an unbounded loop in a
specific implementation. The formats that have been deployed in
hostile environments for decades (TLS's wire format, ASN.1 in the
cellular network, the various IETF protocols) have hardened over
time, but every format that grows a public attack surface for the
first time grows a CVE list along with it. If your binary format is
about to be exposed to untrusted input for the first time, treat the
parser as security-critical code and budget accordingly.

## Where binary is a mistake

The mistakes are also worth saying out loud.

Configuration files, debugging interchange, logs, and any payload
likely to be read by a human in the course of operating the system
should not be binary. Yes, you could encode your config in CBOR and
write a tool to render it. No, you should not. The cost of `cat`-ing
a config file unmodified is enormous and unreplaceable, and any
"savings" from encoding it densely are imaginary because configs are
not on the hot path of anything.

Public APIs to undifferentiated clients should generally not be
binary, even when the request volume is high enough to justify it.
The cost imposed on every consumer of writing a parser will, on
average, exceed the savings to the server. Public APIs that already
have a sophisticated client ecosystem (gRPC) are an exception, and
they exist because the ecosystem solved the parser-distribution
problem. Without that ecosystem, demanding that every caller speak
your binary protocol is a tax on the world to subsidize your
bandwidth bill.

Anywhere a junior engineer needs to be productive without ramping up,
text is usually the right default, and the right binary format is one
that has a *good text projection* (Protobuf's text format, Avro's JSON
encoding, MessagePack's relationship to JSON) so that the binary
representation is visible when wanted and dense when shipped.

The point of this chapter is not to argue you out of using binary
formats. The book would be very short. The point is that *binary* is
a category with costs you can see if you bother to look, and the
formats inside the category differ from each other along enough axes
that the choice between two binary formats can be larger than the
choice between binary and text. The next chapter is the map.
