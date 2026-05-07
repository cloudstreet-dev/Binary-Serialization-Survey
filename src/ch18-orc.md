# ORC

ORC and Parquet are siblings. They were designed in the same year, by
overlapping communities, for the same workload, with overlapping
goals. The technical differences are real but small; the ecosystem
differences turned out to be decisive. Parquet took the Hadoop world,
then the cloud world, then everything that came after; ORC retained
strong adoption in the Hive-centric stacks where it was born and has
been losing ground gradually ever since. Reading the two formats side
by side is the clearest way to understand both, and reading ORC
specifically — even briefly — is the right way to understand the
parts of Parquet that look opaque until you have something to
compare them with.

## Origin

ORC, *Optimized Row Columnar*, was developed at Hortonworks in 2013
to replace RCFile, the early columnar format used by Hive. RCFile
was rudimentary — fixed-row-group layout, no statistics, limited
encoding choices — and Hive's query engine was leaving substantial
performance on the table because of its limitations. Hortonworks
designed ORC as a wholesale replacement: full statistics for
predicate pushdown, type-aware encodings, lightweight indexes,
support for nested types, and tight integration with Hive's
vectorized execution engine.

ORC was contributed to Apache and released under the Apache 2.0
license. Its trajectory from there ran in parallel with Parquet's
but in different orbits. Within the Hortonworks Data Platform —
Hive, Tez, Hadoop with Hortonworks distributions — ORC became the
default columnar format. Within the broader ecosystem (the
Cloudera distribution, the cloud warehouses, the dataframe
libraries, the new query engines), Parquet became the default.
The Cloudera-Hortonworks merger in 2019 brought the two
distributions together, and the combined company has supported
both formats officially since then, but the broader gravitational
pull of Parquet has continued.

The ORC project remains active. The format has had small additive
improvements over the years (bloom filters, additional encodings,
compression codecs), and the reference implementations (Java in
the Hive lineage, C++ for native consumers) are maintained. ORC
files in production environments are common; ORC files written by
new pipelines outside the Hortonworks-derived stack are rare.

## The format on its own terms

An ORC file is a sequence of *stripes*, each of which is a
row-major partition of the file's rows, followed by a *footer*
section containing metadata. The structure is:

```
[stripes: stripe 1, stripe 2, ...]
[file footer: thrift metadata, schema, stripe locations]
[postscript: compression info, footer length, version]
[postscript length: 1 byte]
```

Each stripe contains:

```
[index data: position-aware indexes per column]
[row data: column streams, encoded and compressed]
[stripe footer: thrift metadata for the stripe]
```

A *column stream* in ORC is the per-column equivalent of Parquet's
column chunk. ORC further breaks each column down into multiple
named streams: a *PRESENT* stream (the validity bitmap), a *DATA*
stream (the actual values), a *LENGTH* stream (for variable-length
types), a *DICTIONARY_DATA* stream (for dictionary encoding), and
others depending on the column's type. The streams within a column
are conceptually similar to Parquet's encoding choices but are
exposed as separate, independently-compressed substreams rather
than packed into a single page byte sequence.

Encodings in ORC are oriented around type rather than chosen
generically. Integers use *RLE v2* (a sophisticated run-length
encoding that adapts to data patterns), strings use either dictionary
or direct encoding depending on cardinality, floats use direct
encoding, booleans use bit-packing. Compression is per-stripe and
per-stream, with codec choices including Snappy, zlib, LZO,
Zstandard, LZ4, and Brotli. The codec is uniform across streams
within a stripe but can differ between stripes if the file's
metadata declares it.

The metadata format is, like Parquet's, a structured language —
ORC uses Protobuf rather than Thrift, which is one of the small
historical differences between the two formats and is
operationally invisible to most users. The footer includes the
schema, the per-stripe statistics (min, max, count, sum, etc.),
and the file-level statistics aggregated across stripes. ORC also
supports *bloom filters* on a per-column-per-stripe basis, which
are statistics structures that can answer "this stripe definitely
does not contain rows where column X equals Y" with high accuracy
and low cost. Parquet has bloom filter support too, added later;
ORC's was earlier and is more deeply integrated.

The schema follows a similar primitive/logical type split to
Parquet's. ORC's nested-type story is via STRUCT, LIST, MAP, and
UNION at the schema level, with the same Dremel-style repetition
and definition level encoding underneath. The implementation
details differ slightly (ORC's nested encoding uses different
streams than Parquet's), but the core ideas are the same.

## Wire tour

A single Person record in an ORC file follows a structurally
similar layout to Parquet:

```
[stripe 1]
  [index data: minimal (a single row)]
  [row data]
    [column 1 (id) streams: PRESENT (1 bit), DATA (8 bytes)]
    [column 2 (name) streams: PRESENT, LENGTH, DATA]
    [column 3 (email) streams: PRESENT, LENGTH, DATA]
    [column 4 (birth_year) streams: PRESENT, DATA (RLE v2)]
    [column 5 (tags) streams: PRESENT, LENGTH (list lengths), 
      child column STRING streams]
    [column 6 (active) streams: PRESENT, DATA]
  [stripe footer: protobuf-encoded stripe metadata]
[file footer: protobuf-encoded schema, stripe locations, statistics]
[postscript: compression codec, footer length, version]
```

The single-record file is approximately 1.5 to 2 KB, comparable to
Parquet for the same payload. The fixed overhead — postscript,
footer, stripe footer, per-stream headers — dominates the encoded
size for tiny files, exactly as it does in Parquet, and amortizes
to nothing for realistic batch sizes.

For a million Person records in a single ORC file with stripes of
~100,000 rows, the per-record cost averages around 25 bytes,
similar to Parquet on the same workload. Where ORC distinguishes
itself is in the I/O cost on certain access patterns: ORC's
multi-stream per-column layout means that reading just the PRESENT
bitmap (to count nulls without reading values) is a smaller I/O
than Parquet's equivalent. For specific predicate-pushdown
patterns where the access pattern reads one substream per column
across all stripes, ORC can win modestly. Whether the modest win
matters in practice depends on the workload.

## Evolution and compatibility

ORC's evolution story is essentially identical to Parquet's. The
schema is in the footer; files are written with a fixed schema;
adding columns means writing new files; the reconciliation between
files of different schemas is the table format's responsibility,
not ORC's.

Within a file, the column types are fixed. Cross-file evolution
proceeds by name (or by field ID, if the table format supports it).
Type promotion rules are similar to Parquet's: int32 to int64 is
safe, smaller-to-larger numeric promotions are safe, narrowing or
sign changes are not.

The deterministic-encoding question for ORC is the same as for
Parquet: the format is deterministic given fixed encoding strategy
and compression codec choices, but the encoder has substantial
freedom that is not constrained by the spec. Two ORC files with
the same logical content can be byte-different.

The version differences within ORC are worth noting. ORC v0 was
the original 2013 release; ORC v1 added improvements through 2015;
ORC v2 (sometimes called ORC v1 with additions) brought RLE v2 and
broader encoding options. The current spec version is sometimes
referred to as ORC v2.0; backward compatibility has been preserved,
and modern readers can read v0 files, but very old readers may not
read newer files.

## Ecosystem reality

ORC's ecosystem is concentrated in the Hadoop-Hive-Hortonworks
lineage. Apache Hive uses ORC as its default columnar format.
Apache Tez, Apache Pig, and other Hadoop ecosystem tools support
ORC for read and write. Apache Spark supports ORC reads and
writes; Spark's ORC support is high-quality and is maintained
alongside its Parquet support, but the Spark community generally
defaults to Parquet when given the choice.

The cloud data warehouses that support ORC do so as a secondary
format. AWS Athena, Google BigQuery, and Snowflake can read ORC
files, but their canonical external table formats are Parquet,
and ORC is the format you reach for when you have legacy data that
was already in ORC. Writing new ORC files is uncommon outside the
Hortonworks-derived stacks.

The reference implementations are ORC Java (the canonical Hive
implementation) and ORC C++ (used by C++/Python tools that need
native ORC support). The Apache Arrow project includes ORC
support in its C++ implementation, which is the path most modern
Python and Rust tools take to read ORC files. The Java
implementation is mature and feature-complete; the C++
implementation lags slightly on newer features.

The deployments where ORC retains strong adoption are: Hive
metastores (where the metastore's table definitions specify ORC as
the storage format), Hortonworks-derived data lakes that have not
migrated to Parquet, several large enterprise environments where
ORC was chosen in 2014 and the cost of migration has not yet been
worth it. The trend has been migration toward Parquet, but the
trend is slow, and ORC files in production will remain common for
years to come.

The ecosystem gotchas worth noting. First, *the timestamps issue*:
ORC has its own timestamp encoding history, distinct from
Parquet's. Mixing ORC and Parquet timestamps in cross-format
pipelines is a known source of bugs. Second, *bloom filter
configuration*: ORC's bloom filters are powerful but disabled by
default; enabling them requires explicit per-column configuration
in the writer, and many ORC files in the wild lack them despite
benefiting from them. Third, *the nested-type story*: ORC's
nested-type encoding is structurally different from Parquet's,
and converting between the two for nested data is more nuanced
than for flat schemas.

## When to reach for it

ORC is the right choice when interoperating with a Hive-centric
stack: the metastore expects ORC, the existing data is in ORC, the
query engines are tuned for ORC. It is a defensible choice when
the bloom filter support is a performance differentiator for the
workload and the Parquet bloom filter implementation in your
chosen reader is weaker.

It is the right choice when you are extending an existing ORC
deployment and the cost of switching to Parquet is not yet worth
the modest gains.

## When not to

ORC is the wrong choice for new analytical pipelines outside the
Hive lineage. The ecosystem momentum is overwhelmingly with
Parquet, and the technical advantages of ORC are too small to
overcome the advantages of being in the format every modern tool
prefers.

It is also the wrong choice when interoperating with non-JVM
analytical stacks (Polars, DuckDB, modern Rust query engines all
prefer Parquet) and when interfacing with cloud data warehouses
where Parquet is the canonical external table format.

## Position on the seven axes

ORC's stance on the seven axes is essentially identical to
Parquet's: schema-required, self-describing (footer), columnar,
parse with metadata-driven read avoidance, codegen for the
Protobuf footer, non-deterministic in bytes, file-level evolution.

The single point of difference, perhaps, is the *texture* of the
encodings. ORC's per-column multi-stream layout is more granular
than Parquet's pages-per-column-chunk; this gives ORC a slight
advantage on certain patterns and a slight disadvantage on
others. The advantage is real but modest, and the modest size of
the advantage is the substantive answer to why ORC has not
replaced Parquet despite predating it in some sense and being
designed by people with similar expertise.

## Why the difference between ORC and Parquet stuck

It is worth spending a few hundred words on why two formats this
similar produced such different ecosystem outcomes, because the
answer is not that one format is technically superior. The answer is
that ecosystem momentum compounded around small early advantages
that ended up being decisive.

Parquet had three early advantages. First, Twitter and Cloudera
together had broader reach than Hortonworks alone — Twitter's open
source contributions had visibility outside the Hadoop world, and
Cloudera's distribution was the more popular of the two Hadoop
distributions in 2013-2014. Second, Parquet's documentation and
specification were stronger early on, with cleaner writeups, more
external blog posts, and more conference talks. Third, the C++
implementation of Parquet (`parquet-cpp`, later folded into
`arrow-cpp`) was started earlier than ORC's, which mattered
disproportionately because the dataframe library ecosystem (pandas,
followed by everything that imitated pandas) needed C++ support to
build Python bindings, and that dependency chain pulled the broader
non-JVM ecosystem toward Parquet.

The technical merits diverged less than the ecosystem-level momentum.
By 2017, Parquet's encoding repertoire had expanded to match ORC's
(the v2 encodings closed most of the density gap), and Parquet's
bloom filter support had been added (closing the predicate-pushdown
gap). At that point the formats were roughly equivalent on the wire,
but the ecosystem had already declared a winner: every new analytical
tool added Parquet support as the priority, and ORC support — when
it appeared at all — was secondary.

The Cloudera-Hortonworks merger in 2019 was the formal end of the
race. The combined company supports both formats officially, but the
default for new pipelines is Parquet, and the recommendation for
new deployments is Parquet. Existing ORC files continue to be read
and processed; new ORC files are written when the existing data is
in ORC and the cost of migration exceeds the benefit of switching.

This is the canonical lesson about ecosystem momentum in
technology choices. Two technically equivalent formats, designed in
parallel by overlapping communities, can end up with wildly
different adoption curves based on early decisions that compound. By
the time the formats reach technical parity, the choice has been
made.

## A note on the Hive transactions story

One area where ORC retained a meaningful technical lead for several
years is *ACID transactions* on data lake tables. Hive added
transaction support around 2016 using ORC as the underlying file
format and a delta-files scheme for concurrent updates: each
transaction wrote a new ORC file containing inserts, updates, and
deletes, and the Hive query engine reconciled them at read time.
The mechanism worked but was specific to ORC and tied to Hive.

The table format projects (Iceberg, Delta Lake, Hudi) eventually
generalized this idea: any file format underneath, ACID semantics
in the table format layer above. Iceberg explicitly chose to be
file-format-agnostic and supports both Parquet and ORC. Delta Lake
chose Parquet exclusively. Hudi supports both. The generalization
has made Hive's ORC-specific transaction story largely obsolete;
the equivalent capability now exists for Parquet via Iceberg or
Delta Lake.

This is mentioned because it was, for a period, a genuine reason
to choose ORC over Parquet for certain workloads. That reason has
been retired by the table-format layer. The cell ORC occupied —
columnar storage with ACID semantics on top — is now occupied by
Parquet plus Iceberg, with the ACID semantics handled at a layer
above the file format and the ORC dependence eliminated.

## Epitaph

ORC is Parquet's contemporaneous twin, slightly better on a few
axes that didn't matter; preserved by the Hortonworks lineage,
displaced everywhere else by ecosystem gravity.
