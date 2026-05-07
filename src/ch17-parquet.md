# Parquet

Parquet is the format that won at-rest analytical storage. It is the
default file format for almost every modern data warehouse, the
canonical output of Spark and Snowflake and BigQuery, the on-disk
format that table-format projects (Iceberg, Delta Lake, Hudi) wrap
without replacing, and the format that any new analytical system has
to support before it can be taken seriously. The wire format is
considerably more complex than anything else in this book, and the
complexity is in service of a single property — *predicate pushdown
and column pruning at file granularity* — that pays for itself in
storage costs and query latency on every workload Parquet was built
for.

## Origin

Parquet was created jointly at Twitter and Cloudera in 2012-2013,
based on the Dremel paper that Google had published in 2010.
Dremel's central insight was that columnar storage with explicit
*repetition* and *definition levels* could represent arbitrarily
nested data structures losslessly while preserving columnar access
patterns. Twitter's data engineering team had a Hadoop deployment
processing petabytes of nested log data, and the existing options —
row-oriented sequence files, the early columnar attempts in Hive
RCFile and ORC — either lacked the nested-data story or lacked the
broad ecosystem support necessary for deployment. Twitter and
Cloudera collaborated on Parquet to fill the gap.

The format was donated to Apache in 2013 and graduated to top-level
project status in 2015. Adoption since then has been uniform across
the analytical-data world: every major open-source data engine
supports Parquet for read and write, every major cloud data
warehouse can ingest Parquet directly, and every modern table-format
project (Apache Iceberg, Delta Lake, Apache Hudi) uses Parquet as
its underlying file format. The format's specification has been
remarkably stable; the v1.0 spec from 2015 is largely unchanged in
its essentials, with the post-2015 additions (Parquet v2.0
encodings, stricter type metadata, the v2 LIST and MAP types) all
backward-compatible.

The reason Parquet won is straightforward when the workload is
clear. For analytical queries that touch a small fraction of
columns out of many, and that filter a small fraction of rows out
of many, columnar storage with statistics-driven row-group skipping
reduces the I/O cost dramatically. A query that selects three
columns from a thousand, and filters to the last day's worth of
rows out of a year's, can read 0.1% of the file's bytes if the
file is well-organized. The orders of magnitude this represents
are why every cloud warehouse storage bill is denominated in
Parquet bytes.

## The format on its own terms

A Parquet file is a sequence of *row groups*, each containing
*column chunks*, each containing one or more *pages*, all preceded
and followed by magic numbers and concluded by a metadata footer
encoded in Thrift.

The file's overall structure is:

```
[magic: "PAR1" (4 bytes)]
[row group 1]
  [column chunk 1.1: dictionary page, data page(s)]
  [column chunk 1.2: ...]
  ...
[row group 2]
  ...
[footer metadata: thrift-encoded FileMetaData]
[footer metadata length: 4 bytes little-endian]
[magic: "PAR1" (4 bytes)]
```

The footer metadata is read first by every Parquet reader. The
last 8 bytes of the file give the metadata length and the trailing
magic; the reader seeks to the metadata length, reads the metadata
backward into memory, and obtains a complete schema, the
locations of every row group and column chunk, statistics
(min/max/null count) for every column chunk, and the encodings
used. This footer-first design means a reader can do all
metadata-driven decisions (which row groups to skip, which columns
to read, which pages to decompress) without reading any of the
data pages.

A *row group* is a horizontal partition of the file's rows,
typically containing a few hundred thousand to a few million rows.
Each row group is self-contained: it has all of its columns'
chunks, all the statistics, and can be processed independently.
The row group is the granularity at which parallelism happens;
multiple workers each read different row groups in parallel.

A *column chunk* is the contiguous bytes for a single column
within a single row group. Column chunks contain one or more
*pages*: a *dictionary page* (if the column uses dictionary
encoding) followed by *data pages*. Each page has its own header,
its own encoding, and its own optional compression. The page is
the smallest unit at which encoding and compression are applied.

The page-level encodings are where Parquet's density comes from.
*PLAIN* encoding writes the values verbatim (8 bytes per int64,
4 bytes per int32, length-prefixed for variable types).
*Dictionary encoding* replaces values with integer indices into a
dictionary page, which is dramatic for low-cardinality columns.
*RLE* (run-length encoding) compacts runs of repeated values.
*Bit-packing* combines small integer values into shared bytes.
*Delta encoding* writes successive values as differences from
the previous, which is dramatic for sorted or nearly-sorted columns
like timestamps. Combinations of these (RLE + bit-packing,
delta + dictionary) are common.

On top of the per-page encodings, each column chunk can be
compressed with one of several codecs: Snappy, gzip, Brotli, LZ4,
or Zstandard. The choice trades CPU cost for compression ratio;
Snappy is the historical default, Zstandard is increasingly
preferred for modern hardware where the decode cost is dominated
by I/O.

The Dremel-derived part is the encoding of nested data. A
non-nested column (an int32 column, say) just has values. A
column inside a list, or a column inside a struct that may be
absent, requires extra information to know which value belongs to
which row, especially when the row's nesting structure is
irregular. Parquet encodes this with two streams of integers per
column: *repetition levels* (which level of nesting starts a new
list at this value) and *definition levels* (how many of the
optional/list levels are present at this value). For our flat
Person record there are no repetition levels and the definition
levels are trivial; for a nested record (a Person with multiple
addresses, each with multiple phone numbers) the levels become
substantial and are the part of Parquet that takes a year to
fully internalize.

The schema itself is part of the footer metadata, expressed as a
Thrift-encoded recursive structure of *primitive types* (int32,
int64, float, double, boolean, byte_array, fixed_len_byte_array,
int96 for timestamps in older versions) wrapped in *logical types*
(string, decimal, date, timestamp, list, map). The split between
primitive and logical types is the format's mechanism for adding
new high-level types over time without changing the wire format:
new logical types can wrap existing primitives, and old readers
that don't recognize the logical type fall back to the primitive.

## Wire tour

Encoding our single Person record into a Parquet file is, again,
maximally inefficient — Parquet's overhead is fixed per file and
per row group, and a single-row file pays the full overhead.
The structure looks like:

```
[PAR1 (4 bytes)]
[row group 1]
  [column chunk for id]
    [data page header (~30 bytes)]
    [data page: definition level (1 bit, packed) + value 42 (8 bytes PLAIN)]
  [column chunk for name]
    [data page header]
    [data page: definition level + length-prefixed "Ada Lovelace"]
  [column chunk for email]
    ...
  [column chunk for birth_year]
    ...
  [column chunk for tags (list<string>)]
    [data page: repetition levels, definition levels, length-prefixed strings]
  [column chunk for active]
    ...
[footer: thrift metadata describing schema, row group, column chunks, stats]
[footer length (4 bytes)]
[PAR1 (4 bytes)]
```

The total size for a single Person record is approximately
1.5 to 2 KB depending on the encoding choices and the verbosity
of the metadata. The schema metadata alone is several hundred
bytes; each column chunk has a 30-50 byte page header even if the
data is tiny; the file footer with statistics for each column is
substantial.

The right way to read this is again to project over a realistic
workload. For a million Person records in a single Parquet file
with row groups of, say, 100,000 rows each:

```
id (uint64): dictionary-encoded if the IDs are dense (rare for IDs); 
   otherwise PLAIN at 8 bytes/row, ~8 MB total.
name: usually PLAIN-encoded with snappy compression; the length 
   prefix plus UTF-8 bytes plus compression yields ~8 bytes/row 
   on average for English-language names.
email: same, ~14 bytes/row.
birth_year: dictionary-encoded if the year range is small; with 
   ~100 distinct years out of a million rows, dictionary plus 
   bit-packing yields ~1 byte/row. PLAIN encoding would be 4 
   bytes/row.
tags: list<string> with low-cardinality tags is heavily 
   dictionary-encoded; ~3 bytes/row average for our two-tag rows.
active: a single bit per row plus minimal overhead; ~1 KB total 
   for a million rows.
```

Per record, Parquet on a typical workload averages 20-30 bytes per
record, often less. This is below MessagePack and well below JSON
or BSON, in storage cost terms, and the storage-cost reduction is
without any loss of structure. The format is also queryable: a
predicate that filters to active=true can use the active column's
statistics to skip whole row groups; a query that selects only
`id` and `birth_year` reads only those column chunks and ignores
everything else.

## Evolution and compatibility

Parquet's schema-evolution story is unusual and worth understanding
in detail because it intersects with the table-format projects that
sit on top of Parquet.

At the file level, the schema is fixed: every row group in a Parquet
file conforms to the file's footer-declared schema. Adding a column
means writing a new file with the new schema; the old files retain
the old schema, and a query engine reading both must reconcile.
The reconciliation rules are not part of Parquet itself; they are
the responsibility of the table format (Iceberg, Delta Lake, Hudi)
or the query engine (Spark, DuckDB) reading the files.

The conventional reconciliation rules are: a column present in some
files but not others is treated as null in the files where it's
absent; a column whose type has changed is reconciled via a small
set of allowed type promotions (int32 → int64, etc.); a renamed
column is matched by name across files (if the table format
supports field-ID-based mapping, as Iceberg does, the mapping is
explicit; otherwise it's name-based).

The deterministic-encoding question for Parquet is interesting. The
format is deterministic given a fixed encoding strategy and a
fixed compression codec, but the encoder has substantial freedom
in row group boundaries, column chunk encoding choices, page sizes,
dictionary contents, and statistics emission. Two Parquet files
with the same logical content can be byte-different if the
encoders make different choices. There is no canonical encoding;
applications that need byte-equality (content-addressed storage,
immutable file commits) typically hash the *logical content* rather
than the bytes.

The biggest evolution decision Parquet has made was the v2
encodings introduced around 2017: DELTA_BINARY_PACKED for integers,
DELTA_LENGTH_BYTE_ARRAY for variable-length strings,
DELTA_BYTE_ARRAY for sorted strings, and a few others. These
encodings are dramatically denser for the workloads they target
(timestamps, sorted ID columns, sorted string columns) but were
not universally supported by readers for several years after their
specification. As of 2026, all major Parquet readers support v2
encodings, but pinning to v1 encodings is still common in
deployments that need to support older readers.

## Ecosystem reality

Parquet's ecosystem is the largest of any format in this book.
The reference Java implementation (`parquet-mr`) is the canonical
producer for the JVM stack: Spark, Hive, Hadoop, and most of the
data engineering tools in that lineage. The C++ implementation
(`parquet-cpp`) is the producer for the C++/Python stack: pandas,
DuckDB, ClickHouse, and the broader Arrow-aligned tooling. The
Rust implementation (`parquet`, part of the arrow-rs project) is
the producer for the Rust stack: DataFusion, Polars in some
configurations, the various analytical Rust projects.

The cloud data warehouses — BigQuery, Snowflake, Redshift, Athena —
all support Parquet as their canonical external table format. A
Parquet file in cloud object storage is the universal interchange
format for analytical data; emit a Parquet file and any of the
warehouses will read it.

The table-format projects layered on top of Parquet are the most
important architectural development in analytical data of the past
decade. Apache Iceberg, Delta Lake (originated at Databricks), and
Apache Hudi all use Parquet as the underlying file format and add
a metadata layer that handles ACID transactions, time travel,
schema evolution, and partition management. The metadata
abstraction is what makes data lakes behave like databases, and
Parquet is the layer underneath it.

The most consequential ecosystem fact about Parquet is that it
has converged with Arrow. The `arrow-rs` and `parquet-mr` projects
share substantial implementation logic; reading a Parquet file
into an Arrow record batch is the standard ingest path for most
analytical engines; the Parquet schema and the Arrow schema are
designed to map cleanly onto each other. The Parquet-Arrow
combination is the at-rest plus in-memory pair that defines the
modern analytical stack.

Ecosystem gotchas worth noting. First, *timestamp encoding*: older
Parquet files used the int96 type for timestamps, which has been
deprecated in favor of int64 with logical type TIMESTAMP. Mixing
int96 and int64 timestamps across files in the same dataset is a
common source of bugs. Second, *decimal precision*: Parquet
supports decimals via fixed-length byte arrays with a logical
type, and the precision must match between writer and reader.
Third, *nested type evolution*: changing the structure of a list<>
or map<> column is not always safe across versions, and Iceberg's
field-ID-based mapping is the recommended way to handle nested
schema evolution.

## When to reach for it

Parquet is the right choice for at-rest analytical data. Period.
There is no other format that competes for this role across the
breadth of tooling Parquet supports.

It is the right choice as the underlying file format for table
formats (Iceberg, Delta Lake, Hudi). It is the right choice for
cloud-object-storage-backed data warehouses. It is the right
choice for any analytical pipeline whose output will be consumed
by more than one system.

## When not to

Parquet is the wrong choice for transactional storage; the
write-once batch-oriented file format does not support row-level
updates without a table format on top. It is the wrong choice for
small data sets where the fixed metadata overhead dominates. It
is the wrong choice for streaming data ingestion where files
need to be visible the moment a record arrives; the row-group
discipline imposes batching.

It is also the wrong choice for inter-service RPC, log payloads,
or any other workload where the columnar layout is not what the
consumer wants. Use Protobuf, Avro, or MessagePack for those.

## Position on the seven axes

Schema-required (the schema is in the file footer). Self-describing
(the schema, statistics, and encoding metadata are all in the
footer). *Columnar*. Parse rather than zero-copy in the strict
sense, though the metadata-then-pages pattern enables substantial
read avoidance. Codegen for the Thrift footer; runtime for
everything else. Non-deterministic in bytes; canonical encoding
not specified. Evolution is file-level; in-format evolution is
not supported.

The cell Parquet occupies — schema-required, self-describing,
columnar, predicate-pushdown-aware — is the canonical at-rest
analytical format and the strongest expression of the columnar
idea for storage.

## Epitaph

Parquet is the at-rest format the analytical-data world settled on,
because columnar storage with metadata-driven pruning is, on
realistic queries, an order of magnitude cheaper than every
alternative.
