# Appendix C: Glossary

A reference for the vocabulary used throughout the book. Terms
are listed alphabetically. Cross-references to other glossary
entries are *italicized*; cross-references to chapters are by
chapter number.

**Alignment.** The requirement that a multi-byte value start at
a byte offset divisible by some power of two. Zero-copy formats
(*Cap'n Proto*, *FlatBuffers*, *rkyv*, *Apache Arrow*) require
alignment so that values can be read with native CPU loads. The
cost is padding bytes between fields.

**Append-only evolution.** A schema-evolution rule that allows
new fields to be added at the end of a record but forbids any
other change. *FlatBuffers* and *Cap'n Proto* use this rule.

**ASN.1.** *Abstract Syntax Notation One.* A schema language
specified by the ITU-T, with a family of *encoding rules* (BER,
DER, PER, OER, etc.) that produce bytes from values. Chapter 20.

**Backward compatibility.** A schema-evolution property: code
written for the new schema can read bytes produced under the
old schema.

**BER.** *Basic Encoding Rules.* The original ASN.1 encoding,
TLV-shaped and non-deterministic. Chapter 20.

**Big-endian.** A byte-ordering convention where the most
significant byte of a multi-byte value comes first.

**Bond.** Microsoft's schema-first wire format. Chapter 25.

**Borsh.** *Binary Object Representation Serializer for
Hashing.* A deterministic binary format from the NEAR Protocol
ecosystem. Chapter 22.

**BSON.** *Binary JSON.* The wire format used by MongoDB.
Chapter 6.

**Canonical encoding.** A subset of a format's encoding rules
that produces a unique byte representation for each value.
*CBOR*, *ASN.1 DER*, *Cap'n Proto*, and several others have
canonical-encoding subsets.

**Cap'n Proto.** A zero-copy schema-first format with a
capability-based RPC layer. Chapter 13.

**CBOR.** *Concise Binary Object Representation.* RFC 8949.
Chapter 5.

**CDR.** *Common Data Representation.* The OMG-standardized wire
format used by DDS and ROS 2. Chapter 24.

**Codegen.** Generation of language bindings from a schema at
build time. Most schema-first formats have a codegen step.

**Columnar.** A data layout where all values of one field across
many records are stored contiguously, in contrast to *row-
oriented* layouts. *Parquet*, *ORC*, and *Apache Arrow* are
columnar. Chapters 16-19.

**Compression.** A separate layer applied on top of a
serialization format to reduce wire size. Common codecs include
gzip, snappy, zstandard, LZ4. The format choice is orthogonal
to the compression choice but the two interact.

**Conformance suite.** A set of test cases, in a language-
neutral format, that any implementation of a format must pass.

**Decoder hostility.** A defensive-coding posture where the
decoder validates input thoroughly to handle malformed,
adversarial, or corrupted bytes. Chapter 32.

**Delta encoding.** A compression technique where successive
values are encoded as differences from the previous value.
Effective for sorted or nearly-sorted columns; used in *Parquet*
and *ORC*.

**DER.** *Distinguished Encoding Rules.* The deterministic
subset of *ASN.1 BER*, used for X.509 certificates. Chapter 20.

**Deterministic encoding.** A property of a format where the
same value always produces the same bytes. Required for
hashing, signing, and content-addressable storage.

**Dictionary encoding.** A compression technique where a column's
values are replaced by integer indices into a separate dictionary
buffer. Effective for low-cardinality columns; used in *Parquet*,
*ORC*, *Apache Arrow*.

**Discriminant.** A byte or integer in a serialized union or
enum that identifies which variant the value is.

**Evolution.** Schema change over time. The format's evolution
rules determine which changes are safe and which are breaking.
Chapter 11.

**Extension marker.** ASN.1's `...` syntax that indicates fields
may be added in future versions; old decoders skip the new
fields cleanly. Chapter 20.

**Field tag.** A small integer (or, in some formats, a string)
that identifies a field within a record. Tagged-field formats
use field tags as the wire-level identifier; positional formats
do not.

**FlatBuffers.** A zero-copy schema-first format originally for
mobile games. Chapter 12.

**Forward compatibility.** A schema-evolution property: code
written for the old schema can read bytes produced under the
new schema (typically by ignoring unknown fields).

**Hessian.** A self-describing Java RPC format with object-
reference preservation. Chapter 27.

**IDL.** *Interface Definition Language.* The schema language
of a format. Protobuf's `.proto` files, Thrift's `.thrift`
files, Cap'n Proto's `.capnp` files are all IDLs.

**Ion.** Amazon's typed binary-and-text data format. Chapter 7.

**Length-delimited.** A wire-format pattern where a value is
preceded by an integer giving its byte length, followed by that
many bytes. Used for strings, byte arrays, and nested messages
in many formats.

**Little-endian.** A byte-ordering convention where the least
significant byte of a multi-byte value comes first. The native
order on x86 and most ARM systems.

**Magic number.** A fixed sequence of bytes at the start of a
file or message that identifies the format. *Parquet*'s `PAR1`
is an example.

**MessagePack.** A schemaless self-describing binary format,
data-model-equivalent to JSON. Chapter 4.

**MD5 hash gate.** ROS 1's mechanism for enforcing exact schema
match between publisher and subscriber. Chapter 24.

**NBT.** *Named Binary Tag.* The format used by Minecraft.
Chapter 23.

**OER.** *Octet Encoding Rules.* A modern ASN.1 encoding that
trades some of PER's density for simpler implementation.
Chapter 20.

**Optional field.** A field that may or may not be present in
a record. Different formats represent absence differently:
omission (MessagePack, CBOR), null union branch (Avro), zero
discriminant byte (Borsh, Postcard), missing tag in vtable
(FlatBuffers).

**ORC.** *Optimized Row Columnar.* A columnar storage format
in the Hive lineage. Chapter 18.

**Parquet.** The dominant at-rest analytical storage format.
Chapter 17.

**PER.** *Packed Encoding Rules.* The dense ASN.1 encoding used
in cellular signaling protocols. Chapter 20.

**Postcard.** A no_std-friendly serde-binary format for Rust.
Chapter 26.

**Predicate pushdown.** An optimization where a query's filter
predicate is evaluated against column statistics before reading
the actual data, allowing entire row groups or files to be
skipped.

**Protobuf.** *Protocol Buffers.* Google's schema-first wire
format. Chapter 8.

**Repetition level.** Parquet/Dremel-style metadata indicating
which level of nesting a value belongs to. Used to encode
repeated and optional nested fields.

**Reserved.** Protobuf's keyword for retiring a field number
without deleting it from the schema, so that the number cannot
be reused for a different field. Chapter 8.

**rkyv.** A zero-copy serde-adjacent format for Rust. Chapter
15.

**RLE.** *Run-Length Encoding.* A compression technique that
replaces runs of identical values with a count and a single
value. Used by Parquet and ORC.

**ROS msgs.** The wire formats of the Robot Operating System
(ROS 1 custom; ROS 2 uses CDR). Chapter 24.

**SBE.** *Simple Binary Encoding.* The format for low-latency
financial trading. Chapter 14.

**Schema.** A description of the structure and types of values
that will be encoded. Schemas can live in the bytes (Avro
container files), in a file (Protobuf .proto), or in source
code (rkyv, postcard).

**Schema registry.** An external service that stores and
distributes schemas, often with automated compatibility
checking. The Confluent Schema Registry is the canonical example.

**Schema resolution.** Avro's algorithm for reconciling a
reader's schema with a writer's schema at decode time.

**Schemaless.** A format that does not require an external
schema; the bytes carry their own type and structural
information. *MessagePack*, *CBOR*, *BSON*, *NBT* are
schemaless.

**SCALE.** *Simple Concatenated Aggregate Little-Endian
Encoding.* Substrate/Polkadot's deterministic binary format.
Chapter 22.

**Self-describing.** A format whose bytes contain enough
information to be decoded without external metadata. Note that
self-describing and schemaless are not synonyms; *Avro Object
Container Files* are self-describing but schema-required.

**Smile.** Jackson's binary JSON format with key sharing.
Chapter 7.

**Tagged-field evolution.** A schema-evolution model where
fields are identified by stable numeric tags on the wire,
allowing additions, removals, and renames without breaking
compatibility. Used by *Protobuf*, *Thrift*, *Bond*.

**Thrift.** Apache Thrift, the schema-first wire format from
Facebook. Chapter 9.

**TLV.** *Tag-Length-Value.* A wire format pattern where each
value is preceded by a type tag and a length. Used by *ASN.1
BER*.

**UBJSON.** *Universal Binary JSON.* Chapter 7.

**Varint.** A variable-length integer encoding. Multiple
flavors exist (LEB128, Protobuf's varint, ZigZag-encoded
signed varint); they differ in details but share the property
that small values use few bytes.

**Vtable.** *Virtual Table.* FlatBuffers' per-instance metadata
that records which fields are present in a particular instance.
Chapter 12.

**Wire format.** The on-the-wire byte representation of a
serialization format, distinct from the in-memory representation
or the source-code IDL.

**XCDR2.** Extended CDR version 2. The CDR variant most used by
ROS 2. Chapter 24.

**XDR.** *External Data Representation.* RFC 4506. Chapter 21.

**ZigZag encoding.** A mapping from signed integers to unsigned
integers that places small absolute values close to zero, so
that varint encoding produces compact bytes for negative
numbers. Used by Protobuf's `sint32`/`sint64`, Thrift Compact,
Avro, and several others.

**Zero-copy.** A property of a format where decoded values are
accessed directly from the byte buffer without parsing or
copying. *FlatBuffers*, *Cap'n Proto*, *rkyv*, *Apache Arrow* in
the in-memory case.
