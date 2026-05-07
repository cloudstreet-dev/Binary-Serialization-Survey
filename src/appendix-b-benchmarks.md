# Appendix B: Benchmark Methodology

This book has refused to publish a benchmark shootout. The
foreword explained briefly why, and several chapters touched
the topic in passing. This appendix collects the substantive
argument: most published serialization benchmarks are misleading,
the misleading ones are usually misleading in predictable ways,
and the small number that are honest are dramatically less useful
than they look.

The appendix also describes what an honest benchmark looks like,
in case you want to run one yourself. The conclusion will
probably be that you should not.

## The seven ways benchmarks lie

**The implementation, not the format.** A serialization
benchmark measures the implementation, not the format. Two
Protobuf implementations can differ in encode/decode speed by 5×
or more. A benchmark of "Protobuf vs. CBOR" using the slowest
Protobuf implementation against the fastest CBOR library
produces a defensible-looking result that is actually a comment
on library quality. The problem is that the benchmark's title
suggests it is about the formats; readers extract format-level
conclusions from implementation-level data.

The mitigation: every published benchmark should name the exact
library and version under test, and the conclusions should be
phrased in terms of those libraries, not the formats.

**The payload that suits the format under test.** Pick a payload
with many small integers, and Protobuf wins on size. Pick a
payload with many strings, and the gap shrinks. Pick a payload
with deeply nested optional fields, and Avro looks bad while
schemaless formats look comparable. The choice of payload
determines the conclusion. Benchmarks that present results from
a single payload, especially one chosen by the author, should be
treated as a directional hypothesis rather than a definitive
finding.

**Cold-start vs. steady-state misalignment.** Some benchmarks
measure the first encode/decode of a fresh process, including
JIT warmup, allocation overhead, and code-path priming. Some
measure the steady state after thousands of iterations. The
ratio of these two numbers can be 10× or more for some formats,
and the choice of which to report depends on the use case.
Benchmarks that do not specify which they measured can be off
by an order of magnitude in either direction.

**Allocator overhead.** Encoding and decoding allocate memory
for the in-memory representation. The allocator's cost is
sometimes the bulk of the benchmark, and the allocator's
behavior depends on the language runtime, the OS, the binary's
heap configuration, and the memory pressure on the system. A
benchmark of "decode speed" that runs in a constrained-memory
container produces different numbers than one running on a
large machine.

**Compression layer ignorance.** Most production deployments
compress the wire bytes (gzip, snappy, zstandard). Benchmarks
that report uncompressed sizes ignore the layer that often
dominates the actual size on the wire. Two formats that produce
substantially different uncompressed bytes may produce similar
compressed bytes, and the size question for the deployment is
about compressed bytes.

**Network-stack omission.** A benchmark of "RPC speed" that
measures only the encode-and-decode time misses the network
stack, the TLS layer, the proxy, and the queueing delays that
in practice dominate end-to-end latency. The format choice
matters in proportion to its share of the total time, and that
share is often single-digit percent.

**Selective quoting.** Benchmark results that show one format
"3× faster" sometimes come from comparisons where the absolute
time is microseconds, and the 3× is from 1µs to 3µs. The
relative number is real; the absolute number is below the
noise floor of the actual production workload. Benchmark
quoting that strips the absolute numbers loses critical context.

## The eighth way benchmarks lie: composition

These seven failure modes compound. A benchmark that uses an
old library version on a payload chosen for its format's
strengths in cold-start mode without compression and reports
relative numbers is wrong in seven ways at once. The seven
failures do not cancel; they compound. A benchmark with five
of these problems is essentially noise, regardless of how
careful the rest of its methodology was.

This is the genuine reason the book has refused to publish a
shootout. A benchmark that I would believe (no compounding
failures, fair payload selection, multiple library versions,
both cold-start and steady-state numbers, with-and-without
compression, in a real network stack) would take weeks to
produce, would apply only to one specific workload, and would
need to be re-run every time a library updated. The result
would be modestly useful for that specific workload and
actively misleading for any other. The honest answer was to
not publish.

## What an honest benchmark looks like

If you must run one yourself, the structure is:

1. **Define the workload precisely.** What is the payload shape?
   How many records per message? What is the access pattern
   (encode, decode, partial decode, random access)? What is the
   compression layer? What is the language runtime, the OS, the
   hardware?

2. **Choose libraries deliberately.** Use the most-recent stable
   version of each format's primary library, with the
   configuration most production deployments would use. If
   multiple libraries exist for a format, run all of them or
   document which you chose and why.

3. **Measure both encode and decode separately.** Encode time
   is paid by writers; decode by readers; the costs are not
   symmetric and many workloads care about one more than the
   other.

4. **Measure size at multiple compression levels.** Uncompressed,
   compressed with the codec you would use in production, and
   ideally with two or three codecs for comparison.

5. **Measure cold-start and steady-state separately.** Both
   numbers are useful for different workloads; reporting only
   one obscures.

6. **Measure resource use.** Memory allocations, peak heap, CPU
   utilization. Format choice can affect these even when
   throughput numbers are similar.

7. **Run on representative hardware.** A benchmark on a 64-core
   server tells you nothing about a 4-core embedded device. If
   your deployment target differs from the benchmark machine,
   note it explicitly.

8. **Run repeatedly with statistical rigor.** Single runs are
   noise. Report means, standard deviations, and outlier
   counts. Outlier behavior often differs more between formats
   than mean behavior.

9. **Include a real workload, not a synthetic one.** Synthetic
   workloads (random data, fixed-size records) miss the
   characteristics of real data (skewed distributions,
   correlated fields, locality patterns) that affect format
   performance.

10. **Publish the code and the data.** A benchmark that cannot
    be reproduced is not a benchmark.

## Why most benchmarks fail

The methodology above is expensive. Producing one benchmark for
one workload takes a person-week. Producing benchmarks for the
range of workloads you actually care about takes more. Most
people do not have the time, and so they publish a benchmark
that took an afternoon and skipped the steps that would have
cost most of the work.

The afternoon-benchmark genre is not malicious. It is the
result of well-meaning engineers trying to inform a community
discussion about format choice. The community would be better
served by no benchmark at all than by an afternoon-benchmark,
because the afternoon-benchmark is treated as authoritative
and informs decisions that should have been driven by other
considerations.

## What format-choice decisions should be driven by instead

Almost everything except benchmarks. The decision frameworks
chapter laid out the questions to ask. The format chapters laid
out the trade-offs each format makes. The right inputs to a
format-choice decision are: the workload's structural properties
(boundary, schema control, read/write ratio, determinism,
language landscape, data shape, operational appetite), the
format's track record on the specific properties that matter,
and the team's familiarity with the surrounding tooling. Speed
is somewhere on the list, but rarely near the top.

The corollary is that *if a format is fast enough for your
workload, the speed difference between it and an alternative is
mostly irrelevant*. This is the case for almost every workload.
The cases where speed dominates the choice are real but rare.
Most teams are choosing between formats whose speeds are within
2× of each other, and within 2× the speed difference is almost
always smaller than the difference in operational properties.

## A small acknowledgment

The argument above is not that no benchmarks should ever be
run. It is that benchmarks should be run *for your specific
workload, under your specific conditions, at the time you make
the decision*, and that published benchmarks should be treated
as background context rather than as authoritative input. The
distinction is small but important. Run your own benchmarks if
the format choice is on the margin. Do not let someone else's
benchmarks make the choice for you.

## A summary list

Things to remember about benchmarks:

- The benchmark measures the implementation, not the format.
- The payload determines the result; the wrong payload produces
  the wrong conclusion.
- Cold-start and steady-state numbers can differ by an order
  of magnitude.
- Allocator overhead, network stack, and compression all
  matter and are often missing.
- Selective quoting (relative numbers without absolute context)
  is misleading.
- Composing several of these failures is what produces the
  benchmarks that fail most badly.
- Honest benchmarks are expensive to produce.
- Most format-choice decisions should not be benchmark-driven.

The next appendix is the glossary, where the vocabulary
introduced across the book is collected for reference.
