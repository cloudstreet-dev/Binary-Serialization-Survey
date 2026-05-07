# rkyv

rkyv is the youngest format in this book, and the only one designed
specifically for the type system of a single language. It is a
zero-copy serialization framework for Rust, built around the
observation that Rust's type system, lifetime tracking, and
ownership rules are strong enough to make a zero-copy format
type-safe in ways that no language-agnostic format can be. The
result is a format that produces astonishingly ergonomic Rust APIs
on top of a wire encoding that is competitive with FlatBuffers and
Cap'n Proto. The cost is that the format is Rust-only and that its
schema is effectively *the source code*; sharing rkyv buffers
across language boundaries is not on offer.

## Origin

rkyv (pronounced *archive* — the name is a phonetic abbreviation)
was started by David Koloski in 2020. The motivation was the
observation that Rust's serde ecosystem, while excellent for
parse-style serialization, did not provide a path to zero-copy
deserialization, and the existing zero-copy formats (FlatBuffers,
Cap'n Proto) were uncomfortable to use from Rust because their
generated APIs did not match Rust's idioms. Specifically, both
FlatBuffers and Cap'n Proto produce accessor objects that hold
references into a buffer, and integrating those accessors with
Rust's lifetime and trait systems required either heavy
unsafe-code use or accepting a degraded API.

rkyv took a different approach: rather than generating accessor
objects from a schema, it derives an *archived form* of an
existing Rust type via a procedural macro. The archived form is a
sister type — `ArchivedPerson` for a `Person` — with the same
fields but with each field's type replaced by its archived
equivalent (`ArchivedString` for `String`, `ArchivedVec<T>` for
`Vec<T>`, and so on). The archived form is laid out in memory
exactly the way the bytes on disk are laid out, which means that
casting a byte buffer to `&ArchivedPerson` (with appropriate
alignment and validation) gives you direct access to the data
through Rust's normal field-access syntax. There is no parsing,
no copying, no allocation, and no API ceremony; the archived
struct *is* the buffer.

The format and library are both under active development; rkyv 0.7
is the version most production code targets, with rkyv 0.8 and
the move to rkyv 1.0 introducing significant breaking changes to
the archived layout and the API. This is one of the few places
in this book where I am writing about a moving target, and the
chapter accordingly emphasizes the durable design ideas rather
than the specific layout of any particular version.

## The format on its own terms

An rkyv archive is a contiguous byte buffer containing the
archived form of one or more values. Values are placed in the
buffer in dependency order: leaf values first (raw bytes,
fixed-width primitives), then containers that reference them
(strings, vectors, structs containing strings or vectors), and
finally the root value. A *root pointer* at a known position
(typically just before the end of the buffer) gives the offset of
the root value's location within the buffer.

References between values are encoded as *relative pointers*: a
4-byte signed offset from the location of the pointer to the
location of the pointed-to value. Relative pointers are
position-independent; an archive can be loaded at any address in
memory, or memory-mapped from disk, and the pointers continue to
resolve correctly. This is a stronger property than absolute
pointers (which require the buffer to be loaded at a specific
address) and a weaker property than absolute offsets (which are
what FlatBuffers and Cap'n Proto use); rkyv chose relative pointers
for the ergonomic reason that they map naturally onto Rust's
lifetime system.

The archived form of a primitive type is the same as its
in-memory form, with byte order chosen at archive time. The
archived form of a struct is the concatenation of its archived
fields, in declaration order, with appropriate alignment padding.
The archived form of a `String` is `ArchivedString`, which holds
a relative pointer to the UTF-8 bytes and a length; the bytes
themselves live elsewhere in the buffer. The archived form of a
`Vec<T>` is `ArchivedVec<T>`, a relative pointer plus a length,
with the elements (each in their archived form) stored contiguously
elsewhere. The archived form of an `Option<T>` is
`ArchivedOption<T>`, a discriminant byte plus the archived value
(if Some).

The schema is the Rust source code. The `#[derive(Archive,
Serialize, Deserialize)]` proc macro on a struct or enum generates
the archived form, the serialization function, and the
deserialization function (the latter being optional, since the
zero-copy access pattern usually obviates the need for full
deserialization). The schema is effectively a Rust type
declaration; there is no separate IDL.

Validation — checking that a byte buffer is in fact a well-formed
archive of the expected type — is a separate concern, handled by
the `bytecheck` crate. By default, rkyv assumes archived buffers
were produced by a trusted writer; reading an untrusted buffer
without validation can dereference invalid relative pointers,
read out-of-bounds memory, or trigger undefined behavior. The
`bytecheck`-based validation walks the buffer recursively,
checking that all relative pointers land within the buffer and
that all values are well-formed for their types. Validation is
substantially slower than zero-copy access (it touches every byte
of the buffer); the rkyv pattern is to validate once on input and
then access without further checks.

## Wire tour

Schema (Rust source):

```rust
use rkyv::{Archive, Serialize, Deserialize};

#[derive(Archive, Serialize, Deserialize)]
#[archive(check_bytes)]
struct Person {
    id: u64,
    name: String,
    email: Option<String>,
    birth_year: i32,
    tags: Vec<String>,
    active: bool,
}
```

The archive of our Person value is approximately 144 bytes,
depending on alignment choices and the specific rkyv version. The
high-level layout, with bytes addressed from the start of the
buffer:

```
0x00:  41 64 61 20 4c 6f 76 65 6c 61 63 65    "Ada Lovelace"
0x0c:  00 00 00 00                            padding
0x10:  61 64 61 40 61 6e 61 6c 79 74 69 63
       61 6c 2e 65 6e 67 69 6e 65 00 00 00    "ada@analytical.engine" + padding
0x28:  6d 61 74 68 65 6d 61 74 69 63 69 61
       6e 00 00 00                            "mathematician" + padding
0x38:  70 72 6f 67 72 61 6d 6d 65 72 00 00
       00 00 00 00                            "programmer" + padding
0x48:  d8 ff ff ff 0d 00 00 00                tags[0]: rel-ptr to "math…", len 13
0x50:  e8 ff ff ff 0a 00 00 00                tags[1]: rel-ptr to "programmer", len 10
0x58:  2a 00 00 00 00 00 00 00                Person.id: 42
0x60:  a8 ff ff ff 0c 00 00 00                Person.name: rel-ptr, len 12
0x68:  01 00 00 00 b0 ff ff ff 15 00 00 00    Person.email: Some, rel-ptr to "ada@…", len 21
0x74:  17 07 00 00                            Person.birth_year: 1815
0x78:  d0 ff ff ff 02 00 00 00                Person.tags: rel-ptr to tag list, len 2
0x80:  01 00 00 00                            Person.active: true (with padding)
0x84:  84 ff ff ff                            root pointer: rel-ptr back to Person at 0x58
```

Total: about 136-144 bytes depending on alignment. The exact bytes
of the relative pointers vary with the encoder's chosen layout
order; the important point is that `0x84` (the last four bytes of
the buffer) is the root pointer pointing back to the Person struct
at `0x58`. A reader does:

```rust
let archived = rkyv::access::<ArchivedPerson, _>(&buffer)?;
let id = archived.id;                  // direct field access
let name: &str = archived.name.as_str();
```

`archived.id` is a load from a known offset relative to the buffer.
`archived.name.as_str()` is a load of the relative pointer plus a
load of the length, then a slice operation. No parsing, no
allocation. The Rust borrow checker enforces that the returned
references do not outlive the buffer, which is exactly the
correctness guarantee the zero-copy pattern needs.

If `email` were `None`, the bytes would shrink by the email
string's bytes (about 24 bytes) and the discriminant byte would
read 0 instead of 1. The pointer slot would be empty.

## Evolution and compatibility

rkyv's evolution story is the youngest and least settled in this
book. The format is stable within a single version of rkyv and a
single version of the schema; cross-version evolution is an
ongoing area of work.

Within a single rkyv version, evolving the schema follows the
constraints of zero-copy formats generally. Adding a field to the
end of a struct works if the new schema's layout is preserved;
removing a field changes the layout and breaks old archives.
Reordering fields breaks old archives. Changing a field's type
breaks old archives.

The `rkyv_versioned` and `rkyv_dyn` crates exist as community
contributions to handle multi-version archives — by emitting a
schema version tag in the archive header and dispatching to the
appropriate archived-form definition — but the version-tagging
mechanism is not part of the rkyv core. Production deployments
that need long-lived archives usually layer their own versioning
on top: a discriminator at the start of each buffer, with explicit
migration logic from old archived forms to new ones.

This is the area where rkyv is most clearly a younger format than
its competitors. Cap'n Proto and FlatBuffers spent years working
out evolution stories; rkyv has spent most of its existence
working out the type-system gymnastics that let the archived form
be derived from arbitrary Rust types, which is genuinely difficult
and a major contribution. The evolution story will firm up over
time. For now, rkyv archives in production should be treated as
schema-stable: if the schema must change, the migration is
explicit.

The deterministic-encoding question for rkyv has the same
answer as for the other zero-copy formats. The wire format depends
on the encoder's layout choices, alignment decisions, and pointer
ordering; canonical encoding is achievable but is not the default.
Most rkyv deployments do not require byte-equality and do not
configure for it.

## Ecosystem reality

The rkyv ecosystem is Rust-only and concentrated. The reference
implementation lives at github.com/rkyv/rkyv and is maintained
actively. The `bytecheck` crate handles validation; the
`rkyv_dyn` crate handles trait-object archiving; the
`rkyv_typename` crate handles type identification across versions.
A handful of derived crates (`rkyv_with`, `rkyv_versioned`,
`rkyv_codec`) provide additional features.

The deployments that use rkyv are concentrated in a few areas.
Game development is the largest: several Rust-based game engines
and tools use rkyv for asset serialization, where the load-time
benefits are decisive. Database internals are another: a few
embedded databases and content-addressed stores use rkyv for
on-disk record formats. Server-side use is smaller; the canonical
RPC frameworks in the Rust ecosystem (tonic for gRPC, tarpc) do
not use rkyv, and most Rust services choose Protobuf or MessagePack
instead.

The ecosystem gotchas are typical of a young format. Pin-to-version
discipline matters: a buffer produced by rkyv 0.7 is not readable
by rkyv 0.8, and crates that expose rkyv archives in their public
API have to coordinate version bumps with consumers. Validation
is opt-in and easy to forget; archives from untrusted sources must
be validated, and the failure mode for unvalidated bad data is
undefined behavior, not a clean error.

The most subtle gotcha is alignment. Reading an `ArchivedPerson`
from a buffer requires the buffer to be aligned to the
struct's alignment requirement (8 bytes, in our example). Reading
an unaligned buffer produces undefined behavior on architectures
that require alignment, and a slow miss on architectures that
tolerate it. The `rkyv::AlignedVec` type exists to make
alignment automatic, but consumers of buffers from external
sources (mmap, network) have to handle alignment themselves.

## When to reach for it

rkyv is the right choice when you are working in pure Rust, the
workload benefits from zero-copy access, and the schema is
expected to be stable for the lifetime of the format. The classic
cases: game asset loading, on-disk record formats for
content-addressed storage, mmap-able caches.

It is the right choice when the alternative is FlatBuffers or
Cap'n Proto in Rust, and the developer ergonomics of those
formats are unsatisfying. The Rust APIs rkyv produces are
substantially nicer than the generated Rust bindings of either of
its competitors.

## When not to

rkyv is the wrong choice when cross-language compatibility is
required; the format is Rust-only and will remain so. It is the
wrong choice when long-lived archives across schema versions are
required and the migration story is unwilling to be explicit. It
is the wrong choice when small-buffer overhead is unacceptable;
the relative-pointer machinery has fixed overhead that does not
amortize for tiny payloads.

It is also the wrong choice when validation is required on a
hot path; bytecheck-based validation is fast but not free, and
the cost is meaningful when the buffer is large.

## Position on the seven axes

Schema-required (the schema is the Rust type). Not self-describing
(reading requires the matching Rust type to be defined; there is
no in-band schema descriptor). Row-oriented. *Zero-copy*. Codegen-
only, via proc macros. Non-deterministic by default; canonical
encoding achievable. Evolution constrained, with versioning
typically layered above the format.

The cell rkyv occupies — schema-required, zero-copy, derived from
existing source-language types rather than from an IDL — is the
unique consequence of building a zero-copy format inside a
language with strong type and lifetime systems. The format is the
strongest demonstration in this book of the principle that *the
right serialization format for a single language can be much nicer
than any cross-language format can be*, and the strongest argument
that cross-language compatibility is a feature with non-zero cost.

## A note on the broader serde-binary picture

rkyv exists in a Rust ecosystem with several other binary
serialization options, and the choice between them is worth
understanding even if you ultimately pick rkyv. *bincode* is the
default serde binary format and produces the densest output of
the parse-style binary formats; *postcard* is a no-std-friendly
varint-encoded serde format used in embedded contexts; *speedy* is
a serde-adjacent format that prioritizes decode speed; *abomonation*
is an unsafe zero-copy format that is *strictly* unsafe and
predates rkyv. Each has its constituency.

The dimension on which rkyv is unique among Rust formats is
zero-copy with a safe access pattern. bincode and postcard are
parse-style; speedy is parse-style with optimization; abomonation is
zero-copy but unsafe in a way Rust users have learned to avoid. rkyv's
contribution is to make zero-copy access type-safe in Rust, and the
type-safety is what justifies the relative-pointer machinery and
the derived archived form. The cost is the constraints and the
youth of the ecosystem; the benefit is that what you get works
inside the Rust type system in a way no other format does.

For users in the Rust ecosystem who don't need zero-copy, the
right choice is usually Protobuf via prost or tonic, MessagePack via
rmp-serde, or CBOR via ciborium. rkyv is not the default Rust binary
format; it is the format for the specific workloads where its
zero-copy access is decisive.

## Epitaph

rkyv is the format that asks what a zero-copy archive looks like
when designed inside Rust's type system rather than around it; the
ergonomics are the headline, and the wire format is the consequence.
