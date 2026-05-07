# ASN.1 (BER/DER/PER)

ASN.1 is the format that runs the world without anyone noticing. The
TLS handshake on the connection that loaded this page used ASN.1.
The cellular signaling protocol that delivered the request to the
server used ASN.1. The X.509 certificate that authenticated the
server uses ASN.1. The LDAP directory queries that authorized the
user used ASN.1. SNMP packets, Kerberos tickets, smart card
protocols, MPEG audio metadata, and biometric standards all use
ASN.1 in some encoding. The format predates almost every other
format in this book, and it is, by a large margin, the most
deployed binary serialization technology on Earth.

This chapter is harder to write than most because ASN.1 is not one
format. It is a *schema language* with a family of *encoding rules*
that produce dramatically different bytes for the same logical
value. BER, DER, CER, PER, OER, XER, and JER are all encoding
rules; the schema is one ASN.1 module that any of these encoders
can consume. The right way to read ASN.1 is to understand the
schema language first, then pick whichever encoding rule is
relevant to your context.

## Origin

ASN.1 (Abstract Syntax Notation One) was specified by the ITU-T
(at the time CCITT) in 1988 as part of the OSI protocol stack.
The motivation was that the various OSI layer specifications all
needed a way to describe message structures, and the committee
wanted a single notation that could be used across protocols
rather than each protocol inventing its own. ASN.1 was the
notation. The original encoding rule, BER (Basic Encoding Rules),
was the wire format. Subsequent encoding rules were added over the
following decades as the OSI vision faded but ASN.1 turned out to
be useful for non-OSI work: DER (Distinguished Encoding Rules) was
specified for cryptographic uses where deterministic encoding was
required; PER (Packed Encoding Rules) was specified for bandwidth-
constrained environments like cellular signaling; OER (Octet
Encoding Rules) was specified more recently to combine PER's
density with simpler encoder/decoder logic.

The ITU has continued to maintain the ASN.1 specifications. The
current spec is from 2021, with updates issued periodically. The
specifications themselves are dense, technical, and cover use
cases that range from the canonical (X.509 certificates, signed
with DER) to the obscure (PER-encoded layer 3 protocol messages
in 5G NAS signaling). Reading the specs is a serious undertaking;
fortunately, most users of ASN.1 do not need to read the specs,
because the format is consumed through battle-tested compilers and
runtime libraries that hide the details.

ASN.1's adoption inside cellular networks is essentially universal.
3GPP's LTE and 5G specifications define every wire-format message
in ASN.1 with PER encoding rules. The signaling layers in these
networks — RRC, NAS, S1AP, NGAP, and many more — exchange billions
of ASN.1-PER-encoded messages per second across the world's mobile
infrastructure. Inside cryptography, X.509 (the format of every
TLS certificate), PKCS#7 (the format of signed and encrypted
documents in S/MIME), and OCSP (online certificate status protocol)
are all ASN.1 with DER encoding. Inside identity, Kerberos and
LDAP both use ASN.1 with BER. Inside operations, SNMP uses ASN.1
with BER for every query and response. Inside multimedia, MPEG-7
metadata is ASN.1.

The deployment scale is, in literal terms, larger than any other
format in this book. The fact that most engineers never knowingly
touch ASN.1 is testament to the format's success at being
infrastructure: it works, the libraries exist, and the
specifications are stable enough that the wire format on a 2024
LTE signaling channel is interoperable with the wire format on a
1995 GSM signaling channel.

## The format on its own terms

ASN.1 has two parts that need to be discussed separately: the
schema language and the encoding rules.

The *schema language* is similar in role to Protobuf's `.proto` or
Avro's JSON schemas: it describes the types and structures that
will be encoded. ASN.1's type system is rich. Primitive types
include INTEGER (arbitrary precision), BOOLEAN, REAL, BIT STRING,
OCTET STRING, UTF8String, NumericString, PrintableString, OBJECT
IDENTIFIER (a sequence of integers identifying a global named
entity), and several others. Constructed types include SEQUENCE
(an ordered fixed list of fields), SET (an unordered fixed set of
fields), SEQUENCE OF and SET OF (variable-length lists), and
CHOICE (tagged union). Optional fields are indicated by the
keyword `OPTIONAL`; default values by `DEFAULT`. Extension points
are indicated by `...` and allow new fields to be added in
backward-compatible ways.

A schema looks like this:

```asn1
Person ::= SEQUENCE {
    id          INTEGER,
    name        UTF8String,
    email       UTF8String OPTIONAL,
    birthYear   INTEGER,
    tags        SEQUENCE OF UTF8String,
    active      BOOLEAN
}
```

The schema language allows constraints on values (e.g., INTEGER
(0..99) for an integer in 0-99), constraints on string lengths,
and constraints on the size of SEQUENCE OF. These constraints are
not just documentation; PER and OER use the constraints to encode
fields more compactly. A constrained INTEGER (0..99) is encoded
in PER as 7 bits; the unconstrained INTEGER takes more.

The *encoding rules* take a schema and a value and produce bytes.
Different encoding rules produce different bytes:

**BER** (Basic Encoding Rules) is TLV: every value is encoded as
a Tag, a Length, and a Value. Tags are bytes that include a
class (universal, application, context-specific, or private), a
form bit (primitive or constructed), and a tag number. Lengths
are encoded as a single byte for short values (0-127) or a
length-of-length followed by length bytes for longer values.
Values are encoded according to their type: INTEGERs as
two's-complement big-endian bytes, OCTET STRINGs as raw bytes,
SEQUENCEs as the concatenation of their components.

**DER** (Distinguished Encoding Rules) is a strict subset of BER:
the same tags, lengths, and values, but with additional
constraints that produce a unique byte representation for each
value. DER mandates the smallest possible length encoding, the
shortest INTEGER representation (no leading zeros except for
sign), the canonical TRUE encoding (`FF`, all bits set), and a
specific ordering for SET fields. DER bytes are deterministic;
the same value always produces the same bytes.

**PER** (Packed Encoding Rules) is fundamentally different. PER
does not encode tags or lengths in the bytes for fixed-position
fields; instead, the encoder and decoder both know the schema
and use it to determine where each field is. Optional fields are
indicated by a single bit at the start of a SEQUENCE (a
"preamble") that says which optional fields are present. Integers
are encoded in the minimum number of bits required by their
constraint range. PER produces dramatically smaller bytes than
BER for the same schema, at the cost of being uninterpretable
without the schema.

**OER** (Octet Encoding Rules) is a newer encoding designed to
combine PER's density with simpler implementation. OER aligns to
octet boundaries, uses fixed-width integers where the schema's
constraints permit, and skips the preamble bit-packing
optimization that PER uses. OER is denser than BER and slower than
PER but simpler than both; it has been adopted in some 5G
specifications and in some LWM2M deployments.

**XER** (XML Encoding Rules) and **JER** (JSON Encoding Rules)
encode ASN.1 values as XML and JSON respectively. They are useful
for debugging and for interop with non-ASN.1 systems; they are
not what you would choose for production wire bytes.

## Wire tour

Encoding our Person record. The schema:

```asn1
Person ::= SEQUENCE {
    id          INTEGER,
    name        UTF8String,
    email       UTF8String OPTIONAL,
    birthYear   INTEGER,
    tags        SEQUENCE OF UTF8String,
    active      BOOLEAN
}
```

DER encoding:

```
30 4c                                        SEQUENCE, length 76
   02 01 2a                                  INTEGER, length 1, value 42
   0c 0c 41 64 61 20 4c 6f 76 65 6c 61 63 65 UTF8String, length 12, "Ada Lovelace"
   0c 15 61 64 61 40 61 6e 61 6c 79 74 69 63
        61 6c 2e 65 6e 67 69 6e 65           UTF8String, length 21, "ada@analytical.engine"
   02 02 07 17                               INTEGER, length 2, value 1815
   30 1b                                     SEQUENCE OF, length 27
      0c 0d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e   UTF8String, length 13, "mathematician"
      0c 0a 70 72 6f 67 72 61 6d 6d 65 72            UTF8String, length 10, "programmer"
   01 01 ff                                  BOOLEAN, length 1, TRUE (0xff per DER)
```

78 bytes total. Slightly larger than Protobuf for this payload,
slightly smaller than the schemaless self-describing formats. The
bytes are deterministic by spec: the same value always produces
exactly these bytes, in this order, with these lengths. This is
the property that makes DER the wire format for X.509 certificates
and other signed documents — the bytes can be hashed and signed,
and the signature verifies against the canonical encoding from any
producer.

A few details worth noting in the DER tour. The INTEGER for `id`
takes one byte (0x2a) because 42 fits in a single signed byte.
The INTEGER for `birthYear` takes two bytes (0x07 0x17) because
1815 doesn't fit in one signed byte (the high bit must be 0 for a
positive number). The BOOLEAN encodes TRUE as 0xff (all bits set);
BER permits any non-zero byte to mean TRUE, but DER requires 0xff
specifically. The SEQUENCE OF uses tag 0x30 (SEQUENCE in BER
terminology — ASN.1 conflates the encoding of SEQUENCE and
SEQUENCE OF, distinguishing them only at the schema level).

PER encoding (assuming the schema declares no constraints, so we
use *unaligned* PER) for the same record:

```
80                                           preamble: bit 1 = email present
00 00 00 00 00 00 00 2a                     id: encoded as full INTEGER...
```

Wait — that doesn't work. PER's INTEGER encoding for unconstrained
values uses a length-prefixed representation. For 42:

```
01 2a                                        length 1, value 42
```

The full PER encoding is roughly:

```
80                                           preamble (1 bit): email present
01 2a                                        id: length 1, value 42
0c 41 64 61 20 4c 6f 76 65 6c 61 63 65       name: length 12, "Ada Lovelace"
15 61 64 61 40 61 6e 61 6c 79 74 69 63 61
   6c 2e 65 6e 67 69 6e 65                   email: length 21, "ada@..."
02 07 17                                     birthYear: length 2, value 1815
02                                           tags count: 2
0d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e   tag 1: length 13, "mathematician"
0a 70 72 6f 67 72 61 6d 6d 65 72             tag 2: length 10, "programmer"
01                                           active: 1 bit, value true... 
                                              padded to byte
```

Approximately 73 bytes — modestly smaller than DER, primarily
because PER does not emit the type tag for each field. With
*aligned* PER and explicit constraints in the schema (e.g.,
`birthYear INTEGER (1800..2200)`), the savings increase: a
constrained integer in that range encodes in 9 bits, not 16, and
the boolean encodes in a single bit packed into the preamble.
For schemas with many constrained fields, PER routinely produces
encodings 30-50% smaller than DER.

If `email` were absent, the DER encoding would skip the email
TLV (24 bytes) and the SEQUENCE length would shrink accordingly.
The PER encoding would change the preamble bit to 0 and skip
the email value, saving 22 bytes plus the preamble shift.

## Evolution and compatibility

ASN.1's evolution mechanism is the *extension marker* (`...`),
which appears in a SEQUENCE definition to indicate that future
versions may add fields after the marker. Old decoders that
encounter an extended message skip the post-marker fields they
don't recognize; new encoders include them. The effect is
analogous to Protobuf's tag-based skipping but is built into the
schema language explicitly.

A schema with an extension marker:

```asn1
Person ::= SEQUENCE {
    id          INTEGER,
    name        UTF8String,
    email       UTF8String OPTIONAL,
    birthYear   INTEGER,
    tags        SEQUENCE OF UTF8String,
    active      BOOLEAN,
    ...
}
```

A future version that adds a `country` field after the marker
remains compatible with old decoders; old decoders see the
extension and skip it. New decoders see the new field and
process it.

The schema language also supports *version brackets* — explicit
groupings of fields by schema version — which give finer control
over how extensions are handled. Telecom protocols use these
extensively; cryptographic protocols generally do not.

The deterministic-encoding question for ASN.1 is settled by the
choice of encoding rules. DER is fully deterministic. CER is
also deterministic but with different rules suited to streaming.
PER is deterministic given a schema and a value, though aligned
PER's padding rules add some bytes that vary based on alignment.
BER is non-deterministic — multiple valid encodings of the same
value exist — and BER bytes should not be hashed.

## Ecosystem reality

ASN.1's ecosystem is split between specialized tooling and the
implementations buried inside the major standards. The
specialized tooling is professional-grade, often commercial: OSS
Nokalva, Marben, and a few others sell ASN.1 compilers that
handle the full standard with all extensions. Open-source
implementations exist (asn1c is the canonical C compiler, asn1tools
is the Python implementation, the Bouncy Castle Java library
includes ASN.1 support for cryptographic uses) and are mature for
their respective scopes.

Inside the major standards, ASN.1 is consumed without ceremony.
Every TLS implementation has a DER decoder for X.509 certificates;
the decoder is part of OpenSSL, BoringSSL, and every other TLS
library, and it just works. Every Kerberos implementation has a
BER decoder for tickets. Every LDAP server has a BER decoder for
queries. Every cellular base station has a PER encoder/decoder
for the relevant 3GPP messages.

The ecosystem gotcha that bites engineers when they first
encounter ASN.1 is that *the format depends on the encoding
rules*, and the bytes for a single value can differ dramatically
between BER, DER, and PER. Code that produces DER and code that
expects BER will disagree, even though both are processing the
same schema. Reading an ASN.1 deployment requires identifying
which encoding rule is in use, which is usually obvious from
context (X.509 is DER, LDAP is BER, 3GPP is PER) but is sometimes
documented obscurely.

A second gotcha is *implicit vs. explicit tagging*. ASN.1 schemas
can declare tags as IMPLICIT or EXPLICIT, and the choice affects
the wire bytes. This is one of the historically confusing parts
of ASN.1 and is the source of many implementation incompatibilities
between hand-rolled parsers and standards-conformant libraries.
Use a real ASN.1 compiler; do not write your own parser unless
you are interoperating with a known small subset of features.

## When to reach for it

ASN.1 is rarely the right choice for new general-purpose
protocols. It is the right choice when interoperating with an
existing ASN.1-using ecosystem: you are implementing X.509
extensions, you are writing a cellular base station, you are
extending an LDAP schema, you are working in MPEG-7 metadata.

It is a defensible choice when the requirements include extreme
schema stability, multi-decade specifications, and the operational
infrastructure of standards bodies. The OSI legacy continues to
matter in domains where international standardization is the
governance model: telecommunications, aviation, biometric
identification, government identity systems.

It is the right choice for cryptographic protocols specifically,
because DER's deterministic encoding is essential and the legacy
of X.509-and-friends means every implementation can read it.

## When not to

ASN.1 is the wrong choice for general-purpose typed binary
serialization in 2026. The schema language's expressive power
exceeds what most applications need; the encoding rules add
operational complexity; the tooling is more expensive than the
modern alternatives; and the engineering culture around the format
expects more rigor than most teams have appetite for.

It is also the wrong choice when human readability or hex-dump
debugging matter; the bytes are not meant to be read directly,
and the encoder/decoder is the only way in.

## Position on the seven axes

ASN.1's stance varies by encoding rule. The schema is always
mandatory. *DER and PER are not self-describing*; the bytes alone
are uninterpretable. BER is partially self-describing because the
TLV structure includes type tags that can be parsed without the
schema for some types, though full interpretation still requires
the schema. The format is row-oriented. Parse rather than
zero-copy. Codegen-first via mature compilers. *DER is fully
deterministic*; PER is mostly deterministic; BER is not. Evolution
via extension markers, with explicit per-version schema brackets
when needed.

The cell ASN.1 occupies — schema-rich, encoding-rule-pluggable,
extension-marker-evolved — is the deepest expression of "schema
language and wire format are separate concerns" in this book.
It is also the format with the longest production track record by
several decades.

## Epitaph

ASN.1 is the format that runs the cellular network and the public
key infrastructure; old, capacious, encoding-rule-pluggable, and
the strongest argument that schema languages and wire formats are
separate concerns.
