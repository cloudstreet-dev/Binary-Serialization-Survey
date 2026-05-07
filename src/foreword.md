# Foreword

There is a recurring scene in the working life of a software engineer. Two
systems need to exchange data. Someone has chosen a format. The choice is
defended by a sentence, sometimes a phrase: *we use Protobuf*, *it's all
JSON*, *the analytics team wanted Parquet*, *Kafka uses Avro*. The sentence
is then treated as if it had ended the conversation. It has not ended the
conversation. It has only deferred the parts of the conversation that turn
out, six months or six years later, to be load-bearing.

This is a book about those parts.

The thesis is straightforward and, I think, slightly subversive. Binary
serialization formats are not interchangeable, and the differences between
them are not stylistic. Each format is a coherent answer to a question of
the form: *what do you want to optimize for, and what are you willing to
give up in exchange?* The formats look similar from a distance — they all
turn structured values into bytes and back — but the moment you ask anything
about cost, evolution, ambiguity, or correctness, they diverge violently.
A choice that is fine for one system is malpractice for another. A property
that one format treats as foundational, another treats as a bug. None of
them is wrong about this. They are answering different questions.

You can read every binary format's specification as a contract. Like any
contract, the interesting parts are not the parts that read smoothly. They
are the clauses you wouldn't notice unless you were specifically looking
for them. *Field tags are stable across versions.* *Map ordering is
unspecified.* *Float NaN bit patterns are not preserved.* *Unknown fields
are dropped silently.* *The schema must be available to the reader at
decode time.* Any one of these clauses, taken alone, sounds reasonable.
Each one is also a hand grenade with a fifteen-year fuse. Reading the spec
is the difference between knowing this and being surprised by it.

The aim of this book is to teach that kind of reading. By the end, the
chapter on Protobuf shouldn't make you a Protobuf expert. It should leave
you able to pick up the spec for a format I haven't covered — there are
always more formats — and locate, within an hour, the four or five clauses
that determine what you would and would not bet a system on.

A word on what this book is not. It is not a benchmark shootout. There is
an appendix on benchmark methodology, but its main argument is that almost
every published benchmark is misleading, often through no fault of its
authors, and that the *speed* of a serialization format is one of the
least interesting things about it once you've narrowed the field to a
handful of plausible candidates. Speed is a property of an implementation,
not a format. Two CBOR libraries can differ from each other by ten times.
A bad Protobuf implementation will lose to a good MessagePack implementation
even though Protobuf, in principle, encodes much smaller. The interesting
questions — *will this format hurt me when I add a field, when I remove a
field, when a producer and a consumer skew, when a language we hadn't
planned for shows up, when the data must be read on disk thirty years
from now* — are not benchmark questions.

Nor is this book a recommendation engine. There is no chapter that ends
with *and so you should use X*. There is one chapter on decision frameworks,
which is mostly a list of questions to ask yourself in the right order,
and a chapter on anti-patterns, which is mostly a list of decisions whose
costs are more often paid than acknowledged. The honest answer to *which
format should I use* is almost always *that depends on six things you
haven't told me about your system yet*, and the goal here is to give you
enough vocabulary to know what those six things are.

Finally, this book is not exhaustive, and I want to apologize in advance
to the people whose favorite format I have either skipped or treated
briefly. The formats that get full chapters were chosen because each one
illustrates a distinct point on the design space. MessagePack and CBOR are
both self-describing schemaless binary formats, and so they share a chapter
worth of commentary, but they get separate chapters because the small
differences between them are exactly the kind of small differences this
book is meant to sensitize you to. FlatBuffers and Cap'n Proto are both
zero-copy, but their answers to *what zero-copy actually means* are
incompatible enough to be worth contrasting at length. ASN.1 is older
than most readers and most authors, and is still the format that runs
the cellular network you are reading this on, and so it gets a chapter
even though almost nobody chooses it for new work. The omissions are
defensible; the inclusions are deliberate.

The book is organized along six axes that, between them, separate any
two binary formats you are likely to compare. They are introduced
properly in Chapter 2. Briefly: whether the format requires a schema;
whether the encoded bytes describe themselves; whether records are
laid out in rows or columns; whether the encoding can be read without
parsing; whether bindings are generated at build time or constructed
at runtime; and what kind of compatibility — forward, backward, both,
neither — the format guarantees when the schema changes. Most formats
take a clear position on each axis. The formats that try to take both
positions on an axis at once tend to be the formats with the most
exciting failure modes.

A note on the exemplar. Every format chapter encodes the same record:
a person named Ada Lovelace, with an integer ID, an optional email,
a birth year, a couple of tags, and an active flag. The values were
chosen to be small enough to walk through byte by byte and varied
enough to exercise the parts of each format that differ. The same
record encoded in MessagePack is around forty-five bytes; in BSON,
about a hundred and ten; in Protobuf, around fifty; in Avro with the
schema available, under fifty; in CBOR, similar to MessagePack; in
SBE, fixed and surprising. Lining these up, side by side, in the
appendix is more pedagogically useful than any table of microbenchmark
numbers. You can see the design in the bytes.

A note on voice. I have opinions, and rather than launder them through
the passive voice I have stated them. Where I think a format's design
is an excellent solution to its actual problem, I say so. Where I think
a format is being used outside the problem it was designed to solve,
I say that too. The opinions are mine; the formats are not on trial,
and their authors have generally done excellent work. The point is to
help you think about your own system, not to litigate someone else's
fifteen-year-old design decisions.

A note on what to expect from each chapter. There is a fixed shape:
a short history of who built the format and what they were trying to
fix; an explanation of how the format thinks about the world, in its
own terms, before any comparison is made; a wire tour, where the
exemplar is encoded and every byte is annotated; the rules for
schema evolution and version skew, because that is where the format's
real costs hide; the state of the ecosystem, including the languages
and tools and the gotchas that don't appear in the spec; my honest
view of when to choose this format and when not to; and a single
sentence at the end summarizing the whole thing, because I have found
over the years that a one-sentence summary is the part you actually
remember six months later when you need it.

The book is released into the public domain under CC0. If parts of it
are useful to you in writing your own documentation, your own training
material, or your own decision memos, copy them. If you find an error,
the repository is on GitHub and it accepts pull requests; serializers
have surface area, and surface area means I have certainly gotten
something wrong.

Read the table of contents. Pick a part. The first three chapters
build the vocabulary; after that, you can read in any order. The
formats are old; the systems built on them will outlast most careers;
the questions they answer are not going away. The least we can do
is read the contracts before we sign them.

— D.L.
