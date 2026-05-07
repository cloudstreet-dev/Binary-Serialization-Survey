# Apache Arrow IPC

Apache Arrow is two things at once, and the relationship between them
is worth getting clear before going further. *Arrow* the in-memory
format is a specification for how columnar data should be laid out in
RAM: validity bitmaps, contiguous data buffers, length-prefixed
offsets, all aligned and padded so that any reader on the same machine
can use the same bytes without copying. *Arrow IPC* is the wire
encoding for moving Arrow-formatted data between processes — over a
socket, in a file, through a shared memory segment. The two are
designed together, and the wire format is essentially a dump of the
in-memory layout with a small amount of framing metadata around it.

This chapter covers the IPC encoding because that is the part that
ends up on disk and on the wire. The in-memory layout is described
where necessary to explain what the bytes mean.

## Origin

Arrow grew out of conversations between Wes McKinney (creator of
pandas) and several other dataframe library authors in 2016. The
problem they were solving was the *Tower of Babel* of analytical
data: pandas, R data frames, Spark DataFrames, Apache Drill's
query engine, Impala, and a dozen other systems all worked with
column-oriented data, and all of them used different in-memory
representations, and the cost of converting between representations
when these systems wanted to interoperate was approaching the cost
of the actual analysis. Every pair of systems that wanted to
exchange a million rows had to write a converter, and the converter
had to read every byte twice — once to parse the source format and
once to construct the destination format.

Arrow's design proposition was that all of these systems should use
the same in-memory format, so that interoperability would mean
sharing buffers rather than converting them. The design committee
was unusually broad — McKinney, Hadley Wickham, Jacques Nadeau,
Julian Hyde, and others, drawn from the major dataframe and query
engine projects — and the resulting specification was deliberately
constrained: a small number of layouts (each well-defined),
explicit handling of null values via validity bitmaps, and a
metadata schema layer (using FlatBuffers, of all things) for
describing column types and structures.

Arrow IPC was published as part of the Arrow 1.0 release. Its
adoption since then has been remarkable: pandas, Spark, R's
arrow package, DuckDB, ClickHouse, Polars, Datafusion, and a long
tail of analytical systems all speak Arrow IPC natively.
*Apache Parquet* (covered in the next chapter) and Arrow IPC are
the two pillars of the modern analytical data stack; Arrow is the
in-flight and in-memory format, Parquet is the at-rest format.

## The format on its own terms

Arrow IPC is a sequence of *messages*. Each message has a small
metadata header (encoded as a FlatBuffers buffer, because the Arrow
team chose FlatBuffers as the metadata format for the same
zero-copy reasons that motivate Arrow itself) and a body of one or
more *buffers*. The body buffers are the raw column data: arrays of
validity bits, offsets, and values, laid out exactly as they would
be in memory.

Two message types do most of the work. *Schema* messages describe
the column layout: the column names, types, nullability, child
columns (for nested types). A schema message has no body buffers;
its content is entirely in the metadata. *RecordBatch* messages
carry one batch of rows for a known schema. The metadata header
gives the batch's row count and a list of buffer offsets-and-lengths
within the body; the body itself is a contiguous block containing
each column's buffers in the order the schema describes.

The framing for each message is: a 4-byte continuation marker
(`0xFFFFFFFF`, distinguishing valid messages from end-of-stream),
a 4-byte little-endian integer giving the metadata length, the
metadata bytes, padding to 8-byte alignment, the body length
(implicit, derivable from the metadata), and the body bytes. The
end of a stream is marked by a 4-byte continuation marker
followed by a zero metadata length.

There are two file formats and one streaming format that wrap the
message stream. The *Arrow IPC streaming format* is the bare
sequence of messages described above. The *Arrow IPC file format*
adds a magic header (`ARROW1`), a footer with a metadata catalog
that lists every record batch's location in the file (enabling
random access), and a magic trailer (`ARROW1`). The file format
is what the canonical `.arrow` file extension refers to and is
what `pyarrow.feather.write_feather` produces (more on Feather in
chapter 19).

The columnar buffers for a primitive-typed column are: a *validity
bitmap* (one bit per row, indicating presence), and a *data buffer*
(packed values, in the natural width of the type). For
variable-length types (strings, binary), there is also an *offsets
buffer* of int32 values, with the i-th string occupying bytes
[offsets[i], offsets[i+1]) of the data buffer. For nested types
(lists, structs), the column has *child columns*, each of which
follows the same layout recursively.

This layout is what makes Arrow zero-copy across compatible
systems. A pandas DataFrame and a Polars DataFrame, both using
Arrow as their in-memory format, can share the same buffer for a
column of integers. The bytes are aligned, the validity bits are
in the standard position, the lengths and offsets are computed in
the standard way. No conversion happens.

## Wire tour

Encoding a single Person record as a one-row Arrow IPC stream is
maximally inefficient — Arrow's overhead is fixed and amortizes
over the row count — but produces an honest demonstration of the
format. The schema, expressed as a FlatBuffers message, declares
six fields with their types; that schema is roughly 400 bytes when
encoded. The record batch metadata, also a FlatBuffers message, is
about 200 bytes for our six columns; it lists the buffer offsets
and lengths within the body. The body is the columnar data:

```
column 0 (id, uint64):
  validity bitmap: 1 byte (0x01, indicating row 0 is valid), padded to 8
  data: 8 bytes (42 little-endian) padded to 8
  total: 16 bytes

column 1 (name, string):
  validity bitmap: 1 byte (0x01), padded to 8
  offsets: 2 int32 values (0, 12), padded to 8
  data: 12 bytes "Ada Lovelace", padded to 16
  total: 32 bytes

column 2 (email, string nullable):
  validity bitmap: 1 byte (0x01), padded to 8
  offsets: 2 int32 values (0, 21), padded to 8
  data: 21 bytes, padded to 24
  total: 40 bytes

column 3 (birth_year, int32):
  validity: 8 bytes; data: 4 bytes padded to 8
  total: 16 bytes

column 4 (tags, list<string>):
  list validity: 8 bytes
  list offsets: 2 int32 (0, 2), padded to 8
  child: string column with 2 entries
    child validity: 8 bytes
    child offsets: 3 int32 (0, 13, 23), padded to 16
    child data: 23 bytes, padded to 24
    child total: 48 bytes
  list total: 8 + 8 + 48 = 64 bytes

column 5 (active, bool):
  validity: 8 bytes; data: 1 bit, padded to 8
  total: 16 bytes
```

Body total: 16 + 32 + 40 + 16 + 64 + 16 = 184 bytes. Add the
schema message (about 408 bytes including framing), the record
batch message (about 240 bytes including framing), and the
end-of-stream marker (4 bytes), and the on-the-wire byte count for
this single record is approximately 836 bytes. This is the worst
single-record showing of any format in this book by a wide margin.

The right way to read this number is to project it over a workload
where the format makes sense. A million Person records in Arrow
IPC, in a single record batch, would have approximately the same
fixed metadata overhead (about 650 bytes) plus the columnar body.
The body for a million records is roughly:

```
id:           ~8 MB (1M rows * 8 bytes)
name:         ~12 MB if names average 12 bytes (offsets + data)
email:        ~21 MB
birth_year:   ~4 MB
tags:         variable (most rows have a small tag list)
active:       ~125 KB (1M bits)
```

Per record, that is around 50 bytes plus the average tags size,
which is competitive with Protobuf and below Avro. The fixed
overhead amortizes to nothing. Arrow's per-record cost is low; its
per-batch cost is high; and the right read is at the batch
granularity.

## Evolution and compatibility

Arrow IPC's evolution story is unusual because the format is
designed primarily for in-memory interoperability rather than
long-term archival. The schema is part of the stream, which makes
streams self-describing in the sense that any reader can decode
them given just the bytes. The schema can change from stream to
stream, but it cannot change *within* a stream; once a schema
message has been emitted, all subsequent record batches must
conform to it.

For long-term storage of Arrow IPC files, the schema is in the
file's footer, and the file is self-describing. Adding a column
requires writing a new file with the new schema; old files with
the old schema continue to be readable. There is no in-format
mechanism for a single file to contain multiple schema versions,
but Arrow files are typically managed by file-level orchestration
(partitioned by date, by version) where the schema-per-file
arrangement is acceptable.

Within a schema, the column types form a closed set: integers of
various widths, floats, decimals, strings, binary, dates,
timestamps, durations, intervals, lists, structs, unions, maps,
and a handful of others. Arrow has been adding types incrementally
(decimal128 was followed by decimal256, the various interval types
were added over time), and consumers of older Arrow versions
cannot read streams that use types they don't recognize. The
canonical advice is to pin to a stable Arrow version for long-lived
files and rely on Arrow's *compatibility guarantees* (which are
documented and have held up well) for the interoperability cases.

The deterministic-encoding question for Arrow IPC is partially
answered. The columnar layout is deterministic given a value: the
buffers are the same bytes regardless of who produced them. The
metadata, however, is FlatBuffers, and FlatBuffers is not
deterministic by default. Two Arrow producers can produce
byte-different metadata for the same logical schema, and so two
Arrow IPC streams encoding the same data are not guaranteed to be
byte-equal. For applications that need byte-equality, the body
bytes can be hashed independently of the metadata.

## The Arrow Flight protocol

A note worth including: Arrow IPC is the wire format underneath
Arrow Flight, the gRPC-based service interface for moving Arrow
data between systems at high throughput. Flight wraps Arrow IPC
streams in gRPC's bidirectional streaming and adds metadata for
authentication, schema discovery, and parallel data transfer.
Flight is to Arrow IPC roughly what gRPC is to Protobuf: the
canonical RPC layer.

Flight has been adopted by Dremio, several cloud data services,
and many of the dataframe libraries that integrate with Arrow.
The performance numbers Flight achieves (multi-gigabyte-per-second
single-stream transfers, for typical analytical workloads) are
unmatched by any other RPC over typed records, because the wire
bytes are the in-memory format and the deserialization cost on
both ends is essentially zero.

## Ecosystem reality

Arrow's ecosystem is the largest in the columnar-format space and
is growing. The reference implementations are in C++, Java, Rust,
Go, Python (`pyarrow`), R (`arrow`), JavaScript (`apache-arrow`),
C# (`Apache.Arrow`), and Julia. The C++ and Java implementations
are first-class and feature-complete; the Rust implementation
(`arrow-rs`) is mature and increasingly the basis for other
high-performance analytical systems (DataFusion is built on it).
Python and R use the C++ implementation under the hood.

The deployments that use Arrow IPC are concentrated in analytical
workloads. Pandas can read and write Arrow files. Spark uses
Arrow internally for its Python UDF execution and for
PySpark/pandas interop. DuckDB can ingest and emit Arrow streams
natively. ClickHouse has Arrow support for its `Arrow` and
`ArrowStream` formats. Polars uses Arrow as its native in-memory
format, full stop. The dataframe library landscape has converged
on Arrow as the lingua franca, and the convergence has been the
single largest improvement in analytical-data tooling in the last
decade.

The most consequential ecosystem fact about Arrow is that the
in-memory layout is genuinely zero-copy across systems. A query
engine in Rust and a visualization library in Python can hold the
same buffer simultaneously, with no conversion, and have the same
view of the data. This is rare; almost no other format achieves it
in practice, and Arrow's design is the proof that it is achievable
when the systems agree to align their internal representations
around the format.

The ecosystem gotchas are worth noting. First, the metadata
overhead: for small batches, the FlatBuffers metadata can dominate
the message size. The right pattern is to batch aggressively
(thousands to millions of rows per batch), and consumers that
emit one-row batches are paying enormous overhead. Second,
*dictionary encoding* — Arrow's mechanism for representing
low-cardinality strings as integer indices into a separate
dictionary buffer — is supported by all the major implementations
but with subtle differences in default behavior; producers and
consumers should agree on dictionary policy explicitly. Third,
the *delta dictionary* mechanism (which lets dictionaries grow
across batches) is supported unevenly; sticking to the more
restrictive *replacement dictionary* mode is safer for
cross-implementation interop.

## When to reach for it

Arrow IPC is the right choice for analytical data interchange
between processes that can keep the bytes in memory: dataframe
library interop, query-engine-to-dashboard streaming, parallel
data transfer between cluster nodes. It is the right choice as the
runtime format for modern analytical engines (DuckDB, Polars,
DataFusion all use it natively).

It is the right choice for Arrow Flight services where high
throughput is required.

It is a defensible choice for short-term on-disk caching of
intermediate analytical results, especially if the next consumer
is going to load the file into a dataframe library. The
self-describing schema makes such files well-formed without
external metadata.

## When not to

Arrow IPC is the wrong choice for long-term archival storage; the
schema-per-file arrangement is workable but Parquet is purpose-built
for the at-rest case and is better at it. It is the wrong choice
for transactional data (single records, frequent updates); the
batch-oriented layout amortizes badly. It is the wrong choice for
inter-service typed RPC where the messages are small business
records; Protobuf or Avro fit those workloads better.

It is also the wrong choice when the consumers cannot use the
in-memory format directly, because the principal benefit of
Arrow is that the bytes are usable as memory. If the consumer is
going to translate to its own representation anyway, the cost of
Arrow IPC's metadata overhead is harder to justify.

## Position on the seven axes

Schema-required (the schema is in the stream). Self-describing
(the schema is in the stream — Arrow IPC is one of the few
formats where these two are simultaneously true). *Columnar*.
Zero-copy in the strict sense for compatible in-memory consumers.
Codegen for the FlatBuffers metadata; runtime-typed otherwise.
Body bytes are deterministic; metadata is not by default.
Evolution within a stream is forbidden; cross-stream evolution
is the application's concern.

Arrow's stance is the strongest expression of the columnar idea
in this book and the strongest demonstration that *self-describing*
and *schema-required* are not antonyms; the schema is part of the
self-description.

## Epitaph

Arrow IPC is the wire format for the in-memory layout that
dataframe libraries finally agreed on; column-oriented, zero-copy
across compatible systems, and the reason analytical data tooling
in 2026 is so much less painful than it was in 2014.
