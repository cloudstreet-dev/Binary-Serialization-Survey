# ROS msgs

ROS messages are the wire format of the Robot Operating System,
which despite the name is not an operating system but a publish-
subscribe middleware used to coordinate the components of a
robot — sensors, actuators, planners, perception modules — that
need to share data at high frequency. ROS has two major versions
that use different wire formats: ROS 1 uses a custom binary
format defined by the ROS team, and ROS 2 uses CDR (Common Data
Representation) from the OMG/DDS world. Both are interesting,
both are deployed widely in the robotics community, and both
illustrate a particular set of design choices that matter when
the consumer is a robot's wheels and the producer is a robot's
camera at sixty frames per second.

## Origin

ROS was created at Willow Garage in 2007 by Brian Gerkey, Morgan
Quigley, and others, as the framework underlying the PR2 robot
research platform. The design assumption was that a robot is a
collection of cooperating processes, possibly distributed across
multiple computers, exchanging data over a network at rates
ranging from once-an-hour (battery state) to a thousand-times-
a-second (control loops). The framework needed a wire format
that would handle this rate range, support a polyglot ecosystem
of researchers writing nodes in C++, Python, and other languages,
and allow message types to be defined by the application without
running a code-generation tool every time.

ROS 1's wire format was straightforward and homemade. A message
type was defined in a `.msg` file, the ROS build tools generated
language-specific bindings, and the wire format was a flat
concatenation of fields in declared order — little-endian, fixed
width for primitives, length-prefixed for variable-length
fields. The format was not particularly novel; what mattered
was the integration with the ROS framework's transport layer
(TCPROS and UDPROS), the topic-based pub/sub, and the
language bindings.

ROS 2 was a complete rewrite, started in 2014 and stabilizing
around 2017 with the first ROS 2 distribution. The motivation
for ROS 2 was largely about transport: the team wanted to move
to DDS (Data Distribution Service), an OMG standard for real-time
publish-subscribe with quality-of-service guarantees, multicast
discovery, and broad industrial adoption (DDS is used in
aerospace, defense, autonomous vehicles, and several other
domains where the requirements outstrip what TCPROS could
provide). Adopting DDS meant adopting CDR, the wire format DDS
specifies. ROS 2 messages are still defined in `.msg` files, but
they compile to OMG IDL definitions under the hood, and the bytes
on the wire are CDR.

The two formats are not interchangeable, and a ROS 1 node cannot
talk to a ROS 2 node directly. Bridges exist (the `ros1_bridge`
package translates messages), but they are operational seams in
the architecture rather than format-level interop.

## The format on its own terms

ROS 1's wire format is the simpler of the two and is worth
covering first.

A ROS 1 message is a sequence of fields, encoded in declaration
order, with no headers, no padding, and no per-field metadata.
Primitives are little-endian fixed-width: int32 is 4 bytes, int64
is 8 bytes, float32 is 4 bytes, and so on. Strings are encoded
as a 4-byte length prefix (uint32) followed by the UTF-8 bytes
with no padding. Variable-length arrays are encoded as a 4-byte
length prefix followed by the elements; fixed-length arrays are
encoded as the elements with no length prefix. Booleans are 1
byte (0 or 1).

There are no optional fields. ROS 1 messages have no concept of
absence; every field declared in the schema is present in every
encoded message. The convention for "no value" is sentinel
encoding: empty strings, NaN floats, sentinel integers (often
the value's wraparound or extreme representation). Schema
authors who want optionality have to layer it on top, typically
by emitting a separate "valid" field alongside the optional value.

The schema language is ROS's `.msg` syntax, which is similar to
flat C-struct declarations:

```
uint64 id
string name
string email
int32 birth_year
string[] tags
bool active
```

Each line declares a field with a type and a name. Types include
primitives (int8 through int64, uint8 through uint64, float32,
float64, string, bool, time, duration), arrays (using `[]` or
`[N]` for variable or fixed length), and references to other
message types defined in `.msg` files (allowing nested
messages). The build tools generate the language bindings from
the schema.

ROS 1's distinctive evolution mechanism is the *MD5 hash*. Every
message type's schema is hashed (the MD5 of a canonical text
representation of the schema and its dependencies), and the hash
is part of the topic registration. A subscriber's expected hash
must match the publisher's emitted hash; if they differ, the
subscription is rejected. This is a hard rejection, not a graceful
degradation; ROS 1 will not let a subscriber receive messages
from a publisher with a different schema.

The MD5 mechanism is what makes ROS 1 deployments operationally
manageable at modest scale and operationally painful at larger
scale. A schema change requires updating every node that uses
the message type, simultaneously, before the system can run
again with the new schema. ROS 1's tooling helps with this — the
build system propagates schema changes through dependent
packages — but the format itself imposes the hard requirement.

ROS 2's wire format is CDR, the OMG's *Common Data Representation*.
CDR is very similar in spirit to XDR — fixed-width, byte-order-
prefixed, with rules for padding to natural alignment — but with
modern conveniences. CDR messages begin with a 4-byte
representation header: the first byte is reserved (0), the second
byte indicates byte order and encoding flavor (0x00 for big-endian
plain CDR, 0x01 for little-endian plain CDR, 0x02 and 0x03 for
parameter-list CDR with respective byte orders), and the next two
bytes are options. The body follows, encoded according to the
selected representation.

CDR's data types are similar to XDR's: primitives are fixed-width,
strings are length-prefixed (with a NULL terminator counted in
the length), sequences are length-prefixed arrays, structs are
the concatenation of fields. Padding is added between fields to
align each to its natural boundary; this is a slight elaboration
on XDR's blanket 4-byte alignment, allowing 8-byte fields to be
aligned to 8-byte boundaries.

CDR has an extension point, *XCDR2* (Extended CDR version 2),
which is the encoding most ROS 2 implementations actually use.
XCDR2 supports optional fields (via a parameter-list
representation), more flexible alignment, and a richer schema
evolution story than plain CDR. The OMG IDL files that ROS 2's
build tools generate from `.msg` files specify XCDR2 by default
in modern ROS 2 distributions.

The schema for ROS 2 lives in two places: the `.msg` file (which
the user writes) and the OMG IDL file (which the build tools
generate). The IDL file is the canonical wire-format spec; the
`.msg` file is the user-facing convenience.

## Wire tour

Schema (`Person.msg`):

```
uint64 id
string name
string email
int32 birth_year
string[] tags
bool active
```

ROS 1 encoding (little-endian throughout, no header):

```
2a 00 00 00 00 00 00 00                     id: u64 LE = 42
0c 00 00 00                                  name length: 12
41 64 61 20 4c 6f 76 65 6c 61 63 65          "Ada Lovelace"
15 00 00 00                                  email length: 21
61 64 61 40 61 6e 61 6c 79 74 69 63
   61 6c 2e 65 6e 67 69 6e 65                "ada@analytical.engine"
17 07 00 00                                  birth_year: i32 LE = 1815
02 00 00 00                                  tags count: 2
0d 00 00 00                                  tags[0] length: 13
6d 61 74 68 65 6d 61 74 69 63 69 61 6e       "mathematician"
0a 00 00 00                                  tags[1] length: 10
70 72 6f 67 72 61 6d 6d 65 72                "programmer"
01                                           active: 1 byte = 1 (true)
```

89 bytes. Essentially identical to Borsh in layout (which is no
accident; both formats made the same conservative choices) with
the exception of the boolean width (Borsh's bool is 1 byte; ROS
1's is also 1 byte, so they match exactly here).

The Person record's email cannot be marked as absent in ROS 1
without an out-of-band convention. The cleanest convention is to
add a separate `bool email_present` field; the ad-hoc convention
is to use an empty string and have consumers treat empty as
absent. Neither is satisfying, and both are common in practice.

ROS 2 (XCDR2 little-endian) encoding:

```
00 01 00 00                                  CDR header: LE plain
2a 00 00 00 00 00 00 00                     id: u64 LE = 42
0d 00 00 00                                  name length: 13 (12 + null terminator)
41 64 61 20 4c 6f 76 65 6c 61 63 65 00       "Ada Lovelace\0"
00 00 00                                     padding to next 4-byte boundary
16 00 00 00                                  email length: 22 (21 + null)
61 64 61 40 61 6e 61 6c 79 74 69 63
   61 6c 2e 65 6e 67 69 6e 65 00             "ada@...\0"
00 00                                        padding to 4-byte boundary
17 07 00 00                                  birth_year: i32 LE = 1815
02 00 00 00                                  tags count: 2
0e 00 00 00                                  tags[0] length: 14
6d 61 74 68 65 6d 61 74 69 63 69 61 6e 00    "mathematician\0"
00 00                                        padding
0b 00 00 00                                  tags[1] length: 11
70 72 6f 67 72 61 6d 6d 65 72 00             "programmer\0"
00                                           padding to byte alignment
01                                           active: 1 byte = 1
```

About 100 bytes. The differences from ROS 1 are: the 4-byte
header at the start; the null terminators on strings (counted in
the length); the alignment padding between variable-length
fields. CDR's alignment overhead is real but modest for a
single-record payload.

## Evolution and compatibility

ROS 1's evolution story is the MD5-hash gate. Schemas can change,
but the change requires every node that uses the message type to
be rebuilt and redeployed before the topic can be subscribed
to again. There is no graceful degradation; mismatched schemas
mean no communication.

ROS 2 inherits from XCDR2 a more flexible story. Optional fields
can be added without breaking older subscribers; older subscribers
see the new fields as absent. Type promotions are supported.
Renaming and reordering are still breaking. The schema-hash
mechanism in ROS 2 is the *RIHS hash* (ROS Interface Hash Standard),
which is similar in role to ROS 1's MD5 but is computed
differently and supports the optional-field semantics.

The deterministic-encoding question for ROS messages is
straightforward for ROS 1 (the format is deterministic — fixed
widths, no choices) and slightly more nuanced for CDR-based ROS 2
(deterministic given the byte order, but the byte order can be
either; XCDR2's parameter-list mode allows optional fields whose
absence is encoded by omission, which can produce non-deterministic
ordering). Most ROS deployments do not care about byte-equality;
when they do, they fix the byte order and prohibit XCDR2's
parameter-list optional-field encoding.

## Ecosystem reality

The ROS ecosystem is concentrated entirely in robotics. Open
Robotics maintains both ROS 1 (now in maintenance mode) and ROS
2 (active). Major ROS 2 distributions ship every 1-2 years, with
LTS releases for production deployments.

The DDS layer underneath ROS 2 has multiple vendor implementations:
Eclipse Cyclone DDS (open source), eProsima Fast DDS (open source,
default in many ROS 2 distributions), RTI Connext DDS (commercial,
widely used in automotive and aerospace), and OpenDDS (open
source, less common). Each implementation has its own
characteristics — different transport options, different QoS
defaults, different debugging tools — and ROS 2 deployments
typically choose one based on their requirements.

The rosbag format (ROS's recording format for messages) is itself
a binary format that wraps ROS messages with timestamps and topic
metadata. Rosbag2 (the ROS 2 version) is significantly more
flexible than the original; both have their own ecosystems of
playback tools and analytics frameworks.

The ecosystem gotcha worth noting is *the bridge problem*. ROS 1
and ROS 2 cannot directly interoperate; the `ros1_bridge` package
exists but is operationally finicky, especially for complex
message types or high-rate topics. Many robotics organizations have
mixed ROS 1 / ROS 2 deployments, and the bridge is a substantial
operational burden. The migration story for ROS 1 → ROS 2 is, by
this measure, the long tail of the ecosystem's operational
attention.

A second gotcha is the *quality-of-service* configuration in
ROS 2 / DDS. Different QoS settings (best-effort vs. reliable,
volatile vs. transient-local, deadline guarantees) produce
silently different behavior, and mismatched QoS between publisher
and subscriber means no messages flow. The format itself is
fine; the surrounding configuration is the operational problem.

## When to reach for them

Use ROS messages if you are working in robotics. Otherwise, do
not. The formats are coupled tightly to their respective
frameworks (ROS 1's TCPROS, ROS 2's DDS), and using them outside
those frameworks loses the integration that justifies the
formats' existence.

If you specifically need CDR (the wire format under ROS 2)
without ROS, use a DDS implementation directly. CDR is widely
deployed outside robotics — in automotive infotainment systems,
aerospace control systems, and the broader DDS ecosystem — and is
a reasonable choice for any system where DDS's pub/sub semantics
are useful.

## When not to

ROS messages are the wrong choice for inter-service RPC, log
payloads, configuration, or any application that is not built
on a publish-subscribe middleware. The format choices are
optimized for the middleware integration, not for general
serialization.

## Position on the seven axes

ROS 1: schema-required, not self-describing, row-oriented, parse,
codegen, deterministic, evolution gated by MD5 hash. ROS 2 (CDR):
similar, with XCDR2 adding optional-field support and a more
nuanced determinism story.

Both formats illustrate a design point — robotics middleware
formats — that has substantial deployment but limited general
interest, and the right way to use either is through the surrounding
ROS framework rather than as a standalone serialization format.

## A note on rosbag and the message-replay use case

The persistence story for ROS messages deserves a paragraph
because it is one of the operational corners of the ecosystem
that engineers new to robotics often miss. *rosbag* (in ROS 1)
and *rosbag2* (in ROS 2) are file formats and tools for
recording the messages that flow over a system's topics, with
timestamps, so that the recording can be replayed later for
debugging or analysis. The recordings are essential for
robotics development: a lab robot's behavior in an unusual
situation is captured as a rosbag, sent to a developer, and
replayed in simulation to reproduce the bug.

The rosbag formats are themselves binary serialization formats —
wrappers around the underlying ROS messages with framing,
indexing, and metadata. rosbag1 has a custom format with its own
quirks; rosbag2 supports multiple storage backends (the default
is sqlite3, with mcap as a popular alternative). MCAP, the format
underlying many rosbag2 deployments, is itself an interesting
binary format: it stores any kind of timestamped messages, not
just ROS, and has gained adoption outside the ROS ecosystem in
robotics-adjacent applications (autonomous vehicle data logging,
drone telemetry, sensor fusion pipelines).

This is the second time in this book a format has spawned a
*recording format* of its own — the first was Avro Object
Container Files. The pattern is worth noting: serialization
formats that are used for high-rate streaming workloads tend to
grow companion formats for archival and replay, and the
companion formats often outlive the original. MCAP is now
specified as a generic timestamped-message container that ROS
happens to use; rosbag is now an interface, not a format.

## Epitaph

ROS messages are robotics' default wire format; ROS 1's homemade
binary serves nodes-talking-to-nodes adequately, ROS 2's CDR
piggybacks on DDS's industrial pedigree, and both are bought
together with the framework rather than chosen on their own merits.
