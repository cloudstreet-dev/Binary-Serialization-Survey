# Appendix A: Hex Tours

This appendix collects the wire-tour bytes for our Person record
across every format in the book, side by side. The intent is
that you can scan vertically through the table to compare how
each format handles the same logical value. The chapter that
introduces each format walks the bytes in detail; the appendix
is the reference for the bytes themselves.

The Person record, restated:

```
id          uint64            42
name        string            "Ada Lovelace"
email       optional string   "ada@analytical.engine"
birth_year  int32             1815
tags        list<string>      ["mathematician", "programmer"]
active      bool              true
```

For columnar formats (Arrow IPC, Parquet, ORC, Feather), a single
record is uncharacteristic of the format's intended workload; the
encoding shown is for one record, with the realistic per-record
amortized cost noted in parentheses where applicable.

## Size summary

The same Person record, encoded by every format in this book:

| Format                  | Bytes (single record) | Notes                                    |
|-------------------------|-----------------------|------------------------------------------|
| postcard                | 66                    | Tied for smallest                        |
| bincode 2.0 (default)   | ~66                   | Effectively tied with postcard           |
| Avro                    | 67                    | Smallest schema-first format             |
| Protobuf                | 71                    |                                          |
| Thrift Compact          | 71                    | Effectively tied with Protobuf           |
| ASN.1 PER (unaligned)   | ~73                   | Smaller with constraints                 |
| SCALE                   | 75                    |                                          |
| Bond Compact Binary     | ~75                   |                                          |
| ASN.1 DER               | 78                    | Fully deterministic                      |
| ROS 1                   | 89                    |                                          |
| Borsh                   | 90                    |                                          |
| Smile                   | ~90                   | Single-record (key sharing not engaged)  |
| SBE                     | 92                    | Includes 8-byte message header           |
| ROS 2 (CDR / XCDR2)     | ~100                  | Includes 4-byte CDR header               |
| MessagePack             | 104                   |                                          |
| XDR                     | 104                   |                                          |
| CBOR                    | 105                   | One byte over MessagePack on `id`        |
| bincode 1.x (default)   | 110                   | Fixed-width 8-byte length prefixes       |
| Hessian 2 (single)      | ~125                  | Class-def overhead amortizes per stream  |
| FlatBuffers             | ~130                  | Aligned + padded for zero-copy           |
| NBT                     | 135                   | Uncompressed; gzip cuts to ~100          |
| Cap'n Proto             | 144                   | Word-aligned                             |
| rkyv                    | ~144                  | Comparable to Cap'n Proto                |
| BSON                    | 148                   | Array as document with stringified keys  |
| Arrow IPC               | ~836                  | Single record; ~50 bytes/record at scale |
| Parquet                 | ~1500-2000            | Single record; ~25 bytes/record at scale |
| ORC                     | ~1500-2000            | Single record; ~25 bytes/record at scale |
| Feather V2              | ~836                  | = Arrow IPC file format                  |

The headline observation from the table is that the schema-first
formats cluster between 67 and 100 bytes, the schemaless self-
describing formats cluster between 100 and 150, the zero-copy
formats cluster between 130 and 150, and the columnar formats are
catastrophically inefficient for single records but dominate at
scale.

## Schema-first row formats

### Protobuf (71 bytes)

```
08 2a                                        field 1 (id), varint, value 42
12 0c 41 64 61 20 4c 6f 76 65 6c 61 63 65    field 2 (name), len 12, "Ada Lovelace"
1a 15 61 64 61 40 61 6e 61 6c 79 74 69 63
   61 6c 2e 65 6e 67 69 6e 65                field 3 (email), len 21, "ada@..."
20 97 0e                                     field 4 (birth_year), varint, value 1815
2a 0d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e field 5 (tags), len 13, "mathematician"
2a 0a 70 72 6f 67 72 61 6d 6d 65 72          field 5 (tags), len 10, "programmer"
30 01                                        field 6 (active), varint, value 1
```

### Thrift Compact (71 bytes)

```
16 54                                        field 1 (delta 1, type i64=6), zigzag(42)=84
18 0c 41 64 61 20 4c 6f 76 65 6c 61 63 65    field 2 (string), len 12, "Ada Lovelace"
18 15 61 64 61 40 61 6e 61 6c 79 74 69 63
   61 6c 2e 65 6e 67 69 6e 65                field 3 (string), len 21, "ada@..."
15 ae 1c                                     field 4 (i32), zigzag(1815)=3630
19 28                                          field 5 (list)
   28 0d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e
   0a 70 72 6f 67 72 61 6d 6d 65 72
11                                           field 6 (bool=true=1)
00                                           stop field
```

### Avro (67 bytes)

```
54                                           id: zigzag(42) = 84
18 41 64 61 20 4c 6f 76 65 6c 61 63 65       name: len 12, "Ada Lovelace"
02                                           email union branch 1 (string)
   2a 61 64 61 40 61 6e 61 6c 79 74 69 63 61 6c
      2e 65 6e 67 69 6e 65                   email value: len 21, "ada@..."
ae 1c                                        birth_year: zigzag(1815)
04                                             tags array block of 2
   1a 6d 61 74 68 65 6d 61 74 69 63 69 61 6e   "mathematician"
   14 70 72 6f 67 72 61 6d 6d 65 72            "programmer"
00                                             tags array terminator
01                                           active: true
```

### ASN.1 DER (78 bytes)

```
30 4c                                        SEQUENCE, length 76
   02 01 2a                                  INTEGER, length 1, value 42
   0c 0c 41 64 61 20 4c 6f 76 65 6c 61 63 65 UTF8String len 12
   0c 15 61 64 61 40 61 6e 61 6c 79 74 69 63
        61 6c 2e 65 6e 67 69 6e 65           UTF8String len 21
   02 02 07 17                               INTEGER 1815
   30 1b                                     SEQUENCE OF len 27
      0c 0d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e
      0c 0a 70 72 6f 67 72 61 6d 6d 65 72
   01 01 ff                                  BOOLEAN TRUE (DER: 0xff)
```

## Schemaless self-describing formats

### MessagePack (104 bytes)

```
86                                           map of 6 entries
  a2 69 64                                     key "id"
  2a                                             value 42 (positive fixint)
  a4 6e 61 6d 65                               key "name"
  ac 41 64 61 20 4c 6f 76 65 6c 61 63 65       value "Ada Lovelace" (fixstr 12)
  a5 65 6d 61 69 6c                            key "email"
  b5 61 64 61 40 61 6e 61 6c 79 74 69 63
     61 6c 2e 65 6e 67 69 6e 65                value "ada@..." (fixstr 21)
  aa 62 69 72 74 68 5f 79 65 61 72             key "birth_year"
  cd 07 17                                     value 1815 (uint16, BE)
  a4 74 61 67 73                               key "tags"
  92                                             array of 2
    ad 6d 61 74 68 65 6d 61 74 69 63 69 61 6e   "mathematician"
    aa 70 72 6f 67 72 61 6d 6d 65 72            "programmer"
  a6 61 63 74 69 76 65                         key "active"
  c3                                           true
```

### CBOR (105 bytes)

```
a6                                           map of 6 entries
  62 69 64                                     key "id" (text string len 2)
  18 2a                                        value 42 (uint, 1-byte follow)
  64 6e 61 6d 65                               key "name"
  6c 41 64 61 20 4c 6f 76 65 6c 61 63 65       value "Ada Lovelace"
  65 65 6d 61 69 6c                            key "email"
  75 61 64 61 40 61 6e 61 6c 79 74 69 63
     61 6c 2e 65 6e 67 69 6e 65                value "ada@..."
  6a 62 69 72 74 68 5f 79 65 61 72             key "birth_year"
  19 07 17                                     value 1815 (uint, 2-byte follow)
  64 74 61 67 73                               key "tags"
  82                                             array of 2
    6d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e
    6a 70 72 6f 67 72 61 6d 6d 65 72
  66 61 63 74 69 76 65                         key "active"
  f5                                           value true
```

### BSON (148 bytes)

```
94 00 00 00                                  document length 148 (LE)
12 69 64 00                                  type i64, key "id"
   2a 00 00 00 00 00 00 00                   value 42
02 6e 61 6d 65 00                            type string, key "name"
   0d 00 00 00                               len 13 (12 + null)
   41 64 61 20 4c 6f 76 65 6c 61 63 65 00    "Ada Lovelace\0"
02 65 6d 61 69 6c 00                         type string, key "email"
   16 00 00 00                               len 22
   61 64 61 40 61 6e 61 6c 79 74 69 63 61
      6c 2e 65 6e 67 69 6e 65 00             "ada@...\0"
10 62 69 72 74 68 5f 79 65 61 72 00          type i32, key "birth_year"
   17 07 00 00                               value 1815 LE
04 74 61 67 73 00                            type array, key "tags"
   2c 00 00 00                               inner doc length 44
   02 30 00                                    type string, key "0"
      0e 00 00 00 6d 61 74 68 65 6d 61 74 69 63 69 61 6e 00
   02 31 00                                    type string, key "1"
      0b 00 00 00 70 72 6f 67 72 61 6d 6d 65 72 00
   00                                          inner doc terminator
08 61 63 74 69 76 65 00                      type bool, key "active"
   01                                        value true
00                                           document terminator
```

## Zero-copy and fixed-layout formats

### FlatBuffers (~130 bytes)

The byte-level layout depends on encoder choices and alignment.
The structure is: 4-byte root offset, out-of-line strings and the
tags vector, the Person table's vtable (16 bytes), and the
Person table's data section (32 bytes including padding). Total
in the 130-150 byte range.

### Cap'n Proto (144 bytes)

```
00 00 00 00 12 00 00 00         frame: 0 segments-minus-1, 18 words
00 00 00 00 02 00 03 00         root pointer: 2 data words, 3 ptr words
2a 00 00 00 00 00 00 00         id = 42
17 07 00 00 01 00 00 00         birthYear=1815, active=true (low bit)
0d 00 00 00 62 00 00 00         pointer to "Ada Lovelace"
19 00 00 00 b2 00 00 00         pointer to email
21 00 00 00 17 00 00 00         pointer to tags list
... (out-of-line text and list data; see chapter 13 for the full walk)
```

### SBE (92 bytes)

```
10 00 01 00 01 00 01 00         message header
2a 00 00 00 00 00 00 00         id = 42
17 07 00 00                     birthYear = 1815
01                              active = 1
00 00 00                        padding
0c 00 41 64 61 20 4c 6f 76 65 6c 61 63 65         name (len 12 + bytes)
15 00 61 64 61 40 ...                              email (len 21 + bytes)
00 00 02 00 0d 00 6d 61 ... 0a 00 70 72 ...        tags group (count 2 + entries)
```

### rkyv (~144 bytes)

The exact layout depends on the rkyv version and the encoder's
ordering choices. The general shape: out-of-line strings first,
then the tags list of pointers, then the Person struct, with a
root pointer at the end of the buffer.

## Deterministic and constrained formats

### Borsh (90 bytes)

```
2a 00 00 00 00 00 00 00         id: u64 LE
0c 00 00 00                     name length: u32 LE
41 64 61 20 4c 6f 76 65 6c 61 63 65   "Ada Lovelace"
01                              email Some discriminant
15 00 00 00                     email length: u32 LE
61 64 61 40 ...                 "ada@..."
17 07 00 00                     birth_year: i32 LE
02 00 00 00                     tags count: u32 LE
0d 00 00 00 6d 61 74 ...        "mathematician"
0a 00 00 00 70 72 6f ...        "programmer"
01                              active
```

### SCALE (75 bytes)

```
2a 00 00 00 00 00 00 00         id: u64 LE
30                              name compact length: 12
41 64 61 20 4c 6f 76 65 6c 61 63 65   "Ada Lovelace"
01                              email Some
54                              email compact length: 21
61 64 61 40 ...                 "ada@..."
17 07 00 00                     birth_year: i32 LE
08                              tags compact count: 2
34 6d 61 74 ...                 mathematician
28 70 72 6f ...                 programmer
01                              active
```

### XDR (104 bytes)

```
00 00 00 00 00 00 00 2a         id: u64 BE
00 00 00 0c                     name length: 12
41 64 61 20 4c 6f 76 65 6c 61 63 65   "Ada Lovelace" (no padding, len 12)
00 00 00 01                     email present flag
00 00 00 15                     email length: 21
61 64 61 40 ... 65 00 00 00     "ada@..." + 3 bytes padding
00 00 07 17                     birth_year: i32 BE
00 00 00 02                     tags count: 2
00 00 00 0d 6d 61 74 ... 6e 00 00 00   "mathematician" + 3 padding
00 00 00 0a 70 72 6f ... 72 00 00      "programmer" + 2 padding
00 00 00 01                     active: 1 (bool as u32)
```

## Other notable encodings

### postcard (66 bytes)

```
2a                              id: varint(42)
0c                              name length: varint(12)
41 64 61 20 4c 6f 76 65 6c 61 63 65
01                              email Some
15                              email length: varint(21)
61 64 61 40 ... 65              "ada@..."
ae 1c                           birth_year: zigzag-varint(1815)
02                              tags count: varint(2)
0d 6d 61 74 ...                 "mathematician"
0a 70 72 6f ...                 "programmer"
01                              active
```

### NBT (135 bytes uncompressed)

```
0a 00 00                        TAG_Compound, name ""
04 00 02 69 64 00 00 00 00 00 00 00 2a       TAG_Long "id" = 42
08 00 04 6e 61 6d 65 00 0c 41 64 61 ...      TAG_String "name" = "Ada Lovelace"
08 00 05 65 6d 61 69 6c 00 15 61 64 61 ...   TAG_String "email" = "ada@..."
03 00 0a 62 69 72 74 68 5f 79 65 61 72 00 00 07 17  TAG_Int "birth_year" = 1815
09 00 04 74 61 67 73 08 00 00 00 02 ...      TAG_List "tags" of String, 2 entries
01 00 06 61 63 74 69 76 65 01                TAG_Byte "active" = 1
00                              TAG_End
```

### Hessian 2 (~125 bytes single instance)

The class definition is emitted before the first instance:

```
43 06 50 65 72 73 6f 6e 96      class def: 6 fields named "Person"
   02 69 64 04 6e 61 6d 65 ...  field name strings
60                              instance reference (first class)
ba                              id: 42 (compact long)
0c "Ada Lovelace"               name (short string)
15 "ada@analytical.engine"      email
59 17 07                        birth_year (3-byte int form)
58 96 0d "math..." 0a "prog..."  tags list
54                              active = true
```

## How to read this appendix

The bytes above are the same Person record. Reading vertically
through the formats, you can see the design choices in their
purest form: which formats spend bytes on field names, which on
length prefixes, which on alignment padding, which on type
tags, which on framing, and which manage to encode the value in
near-minimal size by trading off all of these.

The single most useful exercise the appendix supports is to
look at any specific design choice across formats. *How does
each format encode 42?* Look at the first byte after each
format's header. *How does each format encode the boolean
true?* Look at the last byte before the terminator. *How does
each format encode the absence of email?* Take the present
encodings and visualize the diff against the absent ones; for
some formats, it's a missing key; for others, a flipped bit; for
others, a discriminant byte that changes from 1 to 0.

The appendix is a compact view of the book's central thesis:
the same value, encoded by formats with different design
priorities, produces dramatically different bytes. The bytes
are the design.
