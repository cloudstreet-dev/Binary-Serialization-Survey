# Survey of Binary Serialization Formats

An opinionated tour through the binary serialization formats that move
data between processes, machines, and disks.

**Read it:** <https://cloudstreet-dev.github.io/Binary-Serialization-Survey/>

## Thesis

Binary serialization formats are not interchangeable. Each one is an answer
to a question of the form *what do you want to optimize for, and what are
you willing to give up?* The book teaches the reader to read a wire-format
specification the way a lawyer reads a contract: looking for the clauses
that bind, the omissions that matter, and the words whose definitions are
load-bearing.

The axes that organize the survey:

- schema-required vs. schemaless
- self-describing vs. external schema
- row-oriented vs. columnar
- zero-copy vs. parse
- code generation vs. runtime reflection
- deterministic vs. canonical-but-not-deterministic vs. non-deterministic
- evolution strategy: tagged fields, ordering, hashes, none

Every chapter encodes the same record — Ada Lovelace, six fields — and walks
the resulting bytes line by line.

## Table of contents

**Part I — Foundations**
1. What Serialization Is, and Why "Binary" Is a Category
2. The Axes
3. The Person Record

**Part II — The Self-Describing Family**
4. MessagePack
5. CBOR
6. BSON
7. Smile, UBJSON, Amazon Ion

**Part III — Schema-First Wire Formats**
8. Protobuf
9. Thrift
10. Avro
11. Schema Evolution Compared

**Part IV — Zero-Copy and Random Access**
12. FlatBuffers
13. Cap'n Proto
14. SBE
15. rkyv

**Part V — Columnar and Analytical**
16. Apache Arrow IPC
17. Parquet
18. ORC
19. Feather

**Part VI — Domain-Specific and Underappreciated**
20. ASN.1 (BER/DER/PER)
21. XDR
22. Borsh and SCALE
23. NBT
24. ROS msgs
25. Bond
26. Postcard and bincode
27. Hessian

**Part VII — Choosing**
28. Decision Frameworks
29. Anti-Patterns
30. Migration Paths

**Part VIII — Rolling Your Own**
31. When It's Defensible
32. What You Owe Your Future Self
33. A Worked Example

**Appendices**
- A. Hex Tours
- B. Benchmark Methodology
- C. Glossary
- D. Further Reading

## Building locally

Install [mdBook](https://rust-lang.github.io/mdBook/), then:

```
mdbook serve --open
```

The published site is built and deployed by `.github/workflows/deploy.yml`
on every push to `main`.

## License

CC0 1.0 Universal. See [`LICENSE`](./LICENSE). The text of the book is
released into the public domain, to the extent permitted by law.
