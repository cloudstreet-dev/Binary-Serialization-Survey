# NBT

NBT is the format Minecraft uses, and that is approximately the
entire reason it deserves a chapter in this book. The format is
not technically remarkable, the design choices are conventional,
and outside Minecraft and its derivatives there is no production
deployment of any size. What makes NBT interesting is the
pedagogical value: it is a clean, complete, schemaless
self-describing binary format with a distinct flavor — Big-endian,
type-tagged, recursively nested, designed by a single person for
a single application — that has been deployed at scale, evolved
under pressure, and survived the application's transition from
hobby project to billion-user platform. Reading it is the right
way to understand what a hand-rolled binary format looks like
when its constraints are clear and its scope is bounded.

## Origin

NBT — *Named Binary Tag* — was created by Markus Persson, who
designed and built Minecraft's first several years. The format was
introduced in Minecraft Alpha around 2010 to replace the
previous world-storage format, which had been a packed binary
representation of block IDs that did not have room for the
metadata Persson wanted to add. NBT was designed to carry that
metadata: every value is tagged with a type, every value can have
a name, and compound values can nest arbitrarily.

Persson designed NBT in roughly an afternoon, by his own account,
based on the requirements for storing player inventory, world
chunks, and the various entity properties Minecraft wanted to
persist. The format is documented in a short specification on
Mojang's wiki. It has been stable since 2010, with one notable
addition (Long Array, added in 2017 for representing block-light
data more efficiently). The wire format itself has not changed.

NBT's deployment is, in raw byte terms, enormous. Every Minecraft
world is millions of NBT files; every Minecraft server replicates
NBT-encoded chunks to every player; every Minecraft plugin that
wants to persist state writes NBT. The Minecraft modding community
has built tooling around NBT — viewers, editors, converters — that
constitutes a substantial subculture of the wider Minecraft
ecosystem. The Bedrock Edition of Minecraft uses a slightly
different NBT variant (little-endian, with some structural
adjustments) but is recognizably the same format.

Outside Minecraft, NBT has been adopted by a few derivatives and
fan projects. Some Minecraft-adjacent tools use it for shared
configurations. A handful of voxel-game engines have borrowed it
for their save formats, on the reasoning that the format is
well-known to the modding community. Nothing else of consequence.

## The format on its own terms

NBT is a tree of named, tagged values. Every value in an NBT
document begins with a one-byte tag indicating its type. Every
value (with one exception, noted below) is preceded by a
two-byte big-endian length and that many UTF-8 bytes of name. The
value's bytes follow.

The exception to the name rule is values inside a TAG_List. List
elements share a single type (declared in the list header) and
have no individual names; they are just values, concatenated.

The thirteen tags:

```
TAG_End         (0)  - terminates a TAG_Compound; no name, no value
TAG_Byte        (1)  - 1-byte signed integer
TAG_Short       (2)  - 2-byte signed integer (BE)
TAG_Int         (3)  - 4-byte signed integer (BE)
TAG_Long        (4)  - 8-byte signed integer (BE)
TAG_Float       (5)  - 4-byte IEEE 754 float (BE)
TAG_Double      (6)  - 8-byte IEEE 754 double (BE)
TAG_Byte_Array  (7)  - 4-byte length, then that many bytes
TAG_String      (8)  - 2-byte length, then that many UTF-8 bytes
TAG_List        (9)  - 1-byte element type, 4-byte count, elements
TAG_Compound    (10) - sequence of named values, terminated by TAG_End
TAG_Int_Array   (11) - 4-byte length, then that many BE int32s
TAG_Long_Array  (12) - 4-byte length, then that many BE int64s
```

Big-endian throughout. No varint encoding. Strings use modified
UTF-8 (Java's variant, where U+0000 is encoded as two bytes
0xC0 0x80 to allow null-termination — a Java-isms detail that
trips up non-Java implementations regularly). Compound values are
schemaless maps where the keys are names and the values are
typed; lists are homogeneous arrays of one element type.

The format has no schema language. The structure of an NBT
document is whatever the producer chose to emit; consumers walk
the tree, dispatch on tags, and extract values they care about.
This is the schemaless model in its purest form, with the
addition of a type tag system that gives more type information
than JSON does.

The data model has one notable peculiarity: lists declared as
empty must declare their element type, and the canonical choice
for a "list of unknown type" is TAG_End (which is otherwise the
compound terminator). This produces the occasional bug in
implementations that don't expect TAG_End to appear as a list's
element type.

The compression story is part of the format. NBT files are almost
always compressed: gzip is the default, zlib is also common. The
Minecraft world file format wraps each NBT chunk in zlib
compression and stores them in custom region files. A bare
uncompressed NBT file is rare in production; consumers must
detect the compression and inflate first.

## Wire tour

Encoding our Person record. The natural NBT representation:

```
0a 00 00                                     TAG_Compound, name "" (length 0)
                                             — root compound
   04 00 02 69 64                             TAG_Long, name "id" (length 2)
   00 00 00 00 00 00 00 2a                    value 42 (BE i64)

   08 00 04 6e 61 6d 65                       TAG_String, name "name"
   00 0c                                       string length 12
   41 64 61 20 4c 6f 76 65 6c 61 63 65         "Ada Lovelace"

   08 00 05 65 6d 61 69 6c                   TAG_String, name "email"
   00 15                                       string length 21
   61 64 61 40 61 6e 61 6c 79 74 69 63
      61 6c 2e 65 6e 67 69 6e 65               "ada@analytical.engine"

   03 00 0a 62 69 72 74 68 5f 79 65 61 72    TAG_Int, name "birth_year"
   00 00 07 17                                value 1815 (BE i32)

   09 00 04 74 61 67 73                      TAG_List, name "tags"
   08                                          element type: TAG_String
   00 00 00 02                                 count: 2
   00 0d 6d 61 74 68 65 6d 61 74 69 63 69 61 6e
                                              "mathematician" (length-13 UTF-8)
   00 0a 70 72 6f 67 72 61 6d 6d 65 72         "programmer"

   01 00 06 61 63 74 69 76 65                TAG_Byte, name "active"
   01                                          value 1 (true encoded as byte)

   00                                         TAG_End: closes root compound
```

135 bytes uncompressed. Compressed with gzip, the output drops
to about 100 bytes, depending on the compressor settings.
Compared to MessagePack's 104 bytes for the same record, NBT
uncompressed is about 30% larger, primarily because every value
carries a name (a 2-byte length plus the UTF-8 bytes) and every
integer is full-width. Compressed NBT is competitive on size with
MessagePack, but the comparison is misleading; comparing compressed
NBT to compressed MessagePack would close the gap further.

The Person record's email field needed a workaround. NBT compounds
do not have a native concept of *optional*; a field is either in
the compound or it is not, with no marker for absence. The
straightforward representation of "absent email" is to omit the
TAG_String for email entirely. The straightforward representation
of "present email" is to include it. This is the same approach
JSON, MessagePack, and CBOR use, and the schema-evolution
implications are the same: the application has to know which
fields it expects and handle the missing-key case.

The boolean `active` is encoded as TAG_Byte with value 1. NBT
does not have a TAG_Bool; the convention is to use TAG_Byte for
booleans, with 0 meaning false and 1 meaning true. Some NBT
documents use other byte values; consumers should be liberal in
what they accept (anything non-zero) and strict in what they
emit (0 or 1).

## Evolution and compatibility

NBT has no formal evolution story. Adding a field means starting
to emit a new key in the compound; consumers that don't know
about the key skip it. Removing a field means stopping emission;
consumers that expect the key handle it as missing. Renaming a
field is a breaking change that requires application-level
coordination. Type changes are coordinated similarly.

In practice, Minecraft's evolution has been managed through a
combination of strict versioning and pragmatic tolerance. The
*data version* number embedded in every save file lets the game
tell which schema version produced the data, and Mojang ships
*data fixers* that migrate older NBT structures to the current
format on load. The data fixer codebase is substantial — thousands
of lines of Java handling the cumulative migrations across more
than a decade — and is the operational equivalent of what a
schema-evolution-aware format would handle in the format itself.

The deterministic-encoding question for NBT is unsettled. The
format itself is mostly deterministic: integer widths are fixed,
string encodings are fixed, list element ordering is preserved.
The non-deterministic parts are compound member ordering (NBT
compounds are unordered maps, and most encoders preserve insertion
order but the spec does not require it) and the modified-UTF-8
encoding's edge cases (rare in practice). Hashing NBT bytes for
content addressing is workable if you control both ends and
mandate sorted compound members; otherwise it is a known source
of bugs.

## Ecosystem reality

NBT's ecosystem is concentrated entirely in Minecraft and its
derivatives. The canonical Java implementation is in Mojang's own
codebase (closed source for the game itself, but the wire format
is documented and several open-source implementations exist).
Open-source implementations of varying quality exist in Java
(e.g., the Querz NBT library), Python (NBT-py), Rust (the `nbt`
and `fastnbt` crates), JavaScript (prismarine-nbt), Go, C#, and a
handful of others. The Bedrock variant has its own implementations
that handle the little-endian and structural differences.

The tooling worth knowing about: NBTExplorer is a long-standing
GUI tool for inspecting and editing NBT files; mcedit and its
successors include NBT editors; the Minecraft commands ecosystem
uses *SNBT* (Stringified NBT) for in-game configuration of
entities and items, with a JSON-shaped textual syntax that maps
to NBT semantically. SNBT is what most Minecraft players actually
type when they configure things; the binary NBT is what the
game stores.

The most consequential ecosystem fact about NBT is that the
modding community has produced extraordinary tooling on top of
the format. The `Minecraft Wiki` documentation of NBT structures
for every block, entity, and item type is one of the most
detailed schema descriptions of any binary format I have seen,
maintained collaboratively by people who have read the game's
bytecode to figure out what each field means. This is what
schemaless documentation looks like when the community has
sufficient incentive: the format provides no help, and the
community provides everything else.

The ecosystem gotchas. First, the modified-UTF-8 encoding is a
common source of cross-language bugs; non-Java implementations
that use standard UTF-8 will produce subtly wrong bytes for
strings containing the null character. Second, the compression
detection: NBT files in the wild may be uncompressed, gzipped, or
zlibbed, and consumers must auto-detect (typically by examining
the first byte). Third, list-of-TAG_End: empty lists must declare
TAG_End as their element type, and consumers that don't expect
this will mis-parse.

## When to reach for it

NBT is the right choice when you are working in the Minecraft
ecosystem: writing a server plugin, a world-generation tool, a
modpack manager, or anything else that consumes Minecraft data.
Outside that ecosystem, NBT is rarely the right choice; CBOR or
MessagePack do the same job with more support and less specific
weirdness.

It is a defensible choice for hobby projects where the appeal is
*using the format that runs Minecraft*, but the practical reasons
are thin.

## When not to

NBT is the wrong choice for general-purpose typed binary
serialization. The format's eccentricities (modified UTF-8, the
ubiquitous compression assumption, the lack of a schema language,
the bare-tree data model) mean that integrating NBT into a
non-Minecraft system imports complexity for no real gain.

It is also the wrong choice for high-performance applications;
the per-value name string overhead and the integer-width
inflexibility produce encodings larger than modern alternatives
without offsetting benefits.

## Position on the seven axes

Schemaless. Self-describing. Row-oriented (NBT is fundamentally
a tree, but a single record is a row-shaped compound). Parse
rather than zero-copy. Runtime. Mostly deterministic in bytes
but not by spec. No formal evolution mechanism.

NBT's stance on the axes is essentially MessagePack's, with the
Java-specific UTF-8 quirk and the per-value name overhead being
the principal differences. The format is what you get when you
design a schemaless self-describing binary format in 2010
without studying the existing options too carefully — which is
fine, because NBT serves Minecraft's needs adequately, and the
broader ecosystem has not needed it to be more.

## A note on the modding-community schema documentation

Worth a brief detour because the pattern is unusual. The Minecraft
Wiki's documentation of NBT structures is, for many practical
purposes, the schema for the format. The game itself does not ship
a schema document; the modding community has reverse-engineered the
format by reading the game's bytecode, comparing emitted bytes
across versions, and writing up the results. The documentation
covers every block entity, every item, every entity type, every
chunk-format detail. Updates appear within hours of a new game
version.

This is what schemaless plus a sufficiently motivated community
looks like in steady state: the format provides nothing, the
community provides everything, and the documentation is sometimes
better than what a format with an in-band schema language could
produce. The lesson is not that schemaless formats can substitute
for schemas in general — they can't — but that *for a specific
single-vendor application*, the operational discipline of "the
community will document everything because they need to" can fill
in the gaps. Outside Minecraft, this pattern is rare.

The other lesson worth absorbing is that schema documentation,
when it exists outside the format itself, drifts from the format
in subtle ways that bite at the edges. The Minecraft Wiki has had
documented mismatches with the actual game over the years —
fields that were renamed without the wiki being updated, types
that were promoted, fields that were silently removed. Each
mismatch is debugged, each is fixed, and the system continues to
work. But the latency between a format change and a documentation
update is real, and consumers who depend on the documentation are
exposed to that latency. A format with an in-band schema avoids
the latency by definition.

## A note on the SNBT text format

SNBT, *Stringified NBT*, deserves a paragraph because it is the
text projection of NBT that most Minecraft players type without
realizing they are touching a binary format. SNBT looks like JSON
with a small set of differences: types are indicated by suffixes
(`42L` for a long, `1.5f` for a float), arrays of typed integers
are written `[I; 1, 2, 3]`, and compound and list syntax mirrors
JSON's. The Minecraft commands ecosystem uses SNBT extensively;
when a player types `/give @s diamond_sword{Damage:0,Enchantments:
[{id:"sharpness",lvl:5}]}`, the curly-brace block is SNBT, parsed
into NBT, and attached to the item.

SNBT is not a separate format; it is a textual encoding of NBT
values, equivalent in expressiveness. Tools that read NBT files
often have an SNBT mode for human-friendly output. The fact that
SNBT exists is part of why NBT survives despite its eccentricities:
the format has a usable text projection, and the text projection
is what most users actually interact with.

## Epitaph

NBT is the format Markus Persson wrote in an afternoon to give
Minecraft player inventories a place to live; thirteen tags, one
documentation wiki, and a billion users who have never heard of
binary serialization formats.
