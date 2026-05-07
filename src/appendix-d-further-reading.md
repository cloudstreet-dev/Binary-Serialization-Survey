# Appendix D: Further Reading

Annotated pointers to primary sources, secondary sources, and
adjacent material that goes deeper than this book has space for.
The book has paraphrased rather than quoted; the reading list is
where to go for verbatim authority on each format.

## Specifications and primary sources

**Protobuf.** The language reference at `protobuf.dev` and the
encoding documentation at `protobuf.dev/programming-guides/encoding`.
The `.proto` file syntax is documented separately for proto2 and
proto3. The Buf documentation (`buf.build/docs`) is the best
contemporary source for the schema-evolution discipline that
proto3 requires.

**Thrift.** The Thrift specification on the Apache Thrift wiki,
plus the BinaryProtocol and CompactProtocol specifications as
separate documents. The fbthrift project's documentation
(github.com/facebook/fbthrift) covers Facebook's fork's
extensions.

**Avro.** The Apache Avro specification at `avro.apache.org/docs`.
The Confluent Schema Registry documentation
(`docs.confluent.io/platform/current/schema-registry`) covers the
operational pattern that most Avro deployments use.

**MessagePack.** The specification at `msgpack.org`, plus the
GitHub repository (github.com/msgpack/msgpack) for the
implementation list.

**CBOR.** RFC 8949 (the current spec, replacing RFC 7049). RFC
8610 for CDDL. RFC 8152 for COSE. The IANA tag registry at
`iana.org/assignments/cbor-tags` is the authoritative list of
semantic tag assignments.

**BSON.** The specification at `bsonspec.org`. The MongoDB
manual covers the practical implications.

**Smile.** The specification at
`github.com/FasterXML/smile-format-specification`. The Jackson
data-format module is the canonical implementation.

**UBJSON.** The specification at `ubjson.org`.

**Amazon Ion.** The specification at `amzn.github.io/ion-docs`,
including separate documents for the binary format, the text
format, and the type system.

**FlatBuffers.** The documentation at `flatbuffers.dev`, including
the Building With FlatBuffers tutorial. The reference grammar
for `.fbs` files is in the source distribution. TensorFlow Lite's
schema (`tensorflow.org/lite/microcontrollers/library`) is a
substantial real-world example.

**Cap'n Proto.** The documentation at `capnproto.org`, including
the encoding spec, the RPC protocol spec, and the design
rationale. Kenton Varda's blog posts on the format's history are
worth reading.

**SBE.** The specification at `github.com/real-logic/simple-binary-encoding`.
The FIX Trading Community's FIX/SP1 documents cover the
financial-trading deployment context.

**rkyv.** The book at `rkyv.org/book` and the API documentation
at `docs.rs/rkyv`. The format is moving fast; pin to a specific
version when reading.

**Apache Arrow.** The documentation at `arrow.apache.org`,
including the columnar format specification and the IPC format
specification. The Arrow Columnar Format document is the
authoritative reference for the in-memory layout. Wes McKinney's
blog posts at `wesmckinney.com` cover the format's history.

**Parquet.** The Apache Parquet specification at
`parquet.apache.org/docs`. The original Dremel paper (Melnik et
al., 2010) is essential reading for the repetition/definition-
level encoding. Twitter's engineering blog has historical
material on Parquet's development.

**ORC.** The specification at `orc.apache.org/specification`.
The Hortonworks-era documentation covers the operational
patterns most ORC deployments use.

**Feather.** The Apache Arrow documentation, since Feather V2 is
Arrow IPC. The original Feather V1 specification is preserved at
`github.com/wesm/feather`.

**ASN.1.** The ITU-T X.680 series of recommendations is the
authoritative standard. X.680 covers the language; X.690 covers
BER, CER, DER; X.691 covers PER; X.696 covers OER. *ASN.1 — A
Communication Between Heterogeneous Systems* by Olivier Dubuisson
is the most accessible textbook.

**XDR.** RFC 4506 (current; replaces RFC 1014). The Stellar
documentation (`developers.stellar.org`) covers the modern
blockchain use of XDR.

**Borsh.** The specification at `borsh.io`. The NEAR Protocol
documentation covers the smart-contract usage.

**SCALE.** The specification in the Substrate documentation
(`docs.substrate.io`). The `parity-scale-codec` Rust crate
documentation covers the canonical implementation.

**NBT.** The specification on the Minecraft Wiki
(`minecraft.wiki/w/NBT_format`). The `wiki.vg` site covers the
network-protocol uses of NBT inside Minecraft.

**ROS msgs.** The ROS 1 documentation at
`wiki.ros.org/Messages`, the ROS 2 documentation at
`docs.ros.org`. The DDS specification (OMG document
formal/2015-04-10) covers the underlying wire format.

**Bond.** The documentation at `microsoft.github.io/bond`, plus
the GitHub repository at `github.com/microsoft/bond`.

**Postcard and bincode.** The crate documentation on docs.rs
(`docs.rs/postcard`, `docs.rs/bincode`). The bincode 2.0
release notes cover the migration from 1.x.

**Hessian.** The Hessian 2.0 specification at
`caucho.com/resin-3.1/doc/hessian-serialization.xtp` (Caucho's
historical site; mirrored elsewhere). The Apache Dubbo
documentation covers the modern deployment context.

## Secondary sources and analyses

Several secondary sources are worth knowing about as context for
the formats covered:

*Designing Data-Intensive Applications* by Martin Kleppmann (2017)
includes a serialization-format chapter that complements this
book's coverage; Kleppmann's framing is more concise and less
opinionated than this book's.

*The Dremel paper* (Melnik et al., 2010, "Dremel: Interactive
Analysis of Web-Scale Datasets") is the foundational paper for
columnar nested-data encoding and is essential for understanding
Parquet, Arrow, and ORC.

*The Avro specification* itself is unusually well-written for a
specification document and is recommended reading even if you
do not use Avro.

*The Cap'n Proto encoding documentation* is similarly well-
written and conveys the design rationale better than most format
specs.

The blog posts at `wesmckinney.com` cover the rationale for
Arrow and the history of pandas, which is essential context for
the analytical-data-format space.

The book *RFC 8949 in 100 Pages* (a community resource, not an
official IETF publication) is a more accessible introduction to
CBOR than the RFC itself.

## Adjacent material

Several adjacent topics deserve mention:

*Compression codecs.* Most binary serialization deployments use
compression on top. The standard references are: zstandard's
documentation at `facebook.github.io/zstd`, Snappy's at
`google.github.io/snappy`, gzip's RFC 1952. Yann Collet's blog
covers the algorithmic side.

*Cryptographic protocols.* TLS, X.509, COSE, JOSE. The RFCs are
authoritative; for accessible introductions, the IETF
informational RFCs (8446 for TLS 1.3) are well-written.

*Database storage formats.* RocksDB's SST file format, LevelDB's
table format, MongoDB's WiredTiger format, PostgreSQL's heap
format. These are not covered in this book but use design
techniques (LSM-tree variants, B+-tree variants) that intersect
with serialization-format design.

*Distributed log formats.* Kafka's log format, AWS Kinesis's
record format. These wrap the formats covered in this book in
their own framing layers.

*Schema languages.* JSON Schema, OpenAPI, GraphQL SDL. These
are not binary serialization formats but cover schema-evolution
ground that overlaps with the formats in this book.

## A note on what this book is not

The book has been deliberately scoped to *binary* serialization
formats. JSON, XML, YAML, TOML, CSV, MIME-type-encoded protocols,
and other text formats are mostly out of scope, with mentions
where they intersect (Protobuf's JSON encoding, Avro's JSON
encoding, BSON's Extended JSON). A full survey of text formats
would be a different book.

The book has also been scoped to *single-record-and-stream*
formats. Database storage formats (LSM trees, B-trees, columnar
storage layers underneath query engines), filesystem formats
(ZFS, Btrfs, ext4 inode structures), and networking-protocol
framing (TCP segments, IP packets, HTTP/2 frames) are all
outside scope. They are interesting; they are not what this
book covered.

## A closing pointer

If you encounter a format not covered here, the most useful
exercise is the one chapter 2 introduced: take the format's
specification and locate, within an hour, where it sits on the
seven axes. The exercise will tell you what kind of format it
is, what trade-offs it makes, and where in this book's
conceptual landscape it belongs. That is the skill the book
has tried to teach. The formats themselves come and go; the
skill is durable.
