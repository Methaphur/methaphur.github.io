---
title: "Collaborative text editing with RGA"
date: 2026-07-30
description: "Notes on a tombstone-free RGA, proved it RA-linearizable in Lean 4 with zero sorries, and then discovered it reorders your text on delete"
summary: "A tombstone-free sequence MRDT, machine-checked in Lean 4 — and a machine-checked negative result about it. On why convergence is a substrate rather than a correctness criterion, and why a specification has to be external to the implementation's own fold."
categories: ["fpl","Notes"]
tags:
  - "Lean"
  - "Formal Verification"
  - "CRDT"
  - "MRDT"
  - "RGA"
  - "Collaborative Editing"
  - "Distributed Systems"
showTableOfContents: true
showAuthor: true
showDate: true
showReadingTime: true
# showWordCount: true
showTaxonomies: true
showEdit: false
showHero: false
---

{{< katex >}}

A warning up front: this is going to be a long and technical post. It is also the one I most wanted to write.

This summer I had the opportunity to work with **Prof. KC Sivaramakrishnan** at
**FP Launchpad, IIT Madras**. The project lived in [**Sal**](https://kcsrk.info/papers/sal_jan26.pdf): the Lean 4 port of the F\* framework **Neem** (Soundarapandian, Nagar, Rastogi, Sivaramakrishnan, OOPSLA 2025), for verifying state-based CRDTs and Mergeable Replicated Data Types (MRDTs).

My work was with RGA, the sequence CRDT behind Automerge and
Yjs.It has to keep every character you ever deleted, forever. An MRDT's merge gets to see the
common ancestor of the two states it is merging. Could that extra information pay for the
graveyard?

I spent the summer answering that. The short version is that we got a design, proved it correct in Lean with zero `sorry`s, and then proved that it is wrong. Not wrong in the
proof; wrong in what the proof was about. A single backspace reorders text the user never
touched, and the theorem could not see it, because it was checking the
implementation against *itself*.

That turned out to be the most useful thing I learned all summer, so it gets its own
section rather than a footnote. The rest of the post is the road there and the fix on the
other side.

The arc is:

1. Why collaborative editing is hard, and why *indices* are the wrong vocabulary.
2. RGA, and what its tombstones are actually paying for.
3. MRDTs: a merge that gets to see the common ancestor, and the question that opens.
4. A tombstone-free RGA that *rehomes* survivors, and its proof.
5. The defect, as a theorem. Why the proof could not see it. Why it is not patchable.
6. **EmbedRGA**: make position an immutable birth constant, and everything gets easier.

---

## 1. The problem

<!-- Google Docs, Overleaf, Notion, Apple Notes. Two authors typing in the same paragraph at
the same instant is not an edge case; it is Tuesday. Nobody accepts a lock, a *document
is busy* dialog, or a `<<<<<<< HEAD` marker in the middle of a sentence, and nobody
accepts losing the paragraph they wrote on a plane. -->

I don't need to mention why collaborative text editing is useful in today's world, especially as students or researchers. Google Docs and Overleaf is almost indispensable if you're working with a team. Those late night edits before an assignment deadline, watching the cursor lobby around the document and seeing the changes made by your teammates in real time is a great experience. But what happens when you are working on a document and your internet connection drops? Or when two people are editing the same paragraph at the same time? How do we ensure that everyone sees the same document, and that no edits are lost? It would be incredibly frustrating to lose your work because of a network issue.

So the product requirement is: **always writable, never lost, everyone eventually sees the
same document**, and it should be the document people meant.

Each client is a full replica. The edit is applied to the local copy *first*, with no
round trip on the critical path. So documents will diverge, but we have a decent way to reconcile. They just need to _eventually_ converge. **Strong _Eventual_ Consistency**

{{< mermaid >}}
sequenceDiagram
    participant A as Replica A (plane, offline)
    participant B as Replica B (office)
    participant C as Replica C (phone)

    Note over A,C: all three replicas hold AB

    A->>A: ins X — applied locally, 0 ms
    B->>B: ins Y — applied locally, 0 ms
    C->>C: del B — applied locally, 0 ms

    B->>C: ins Y
    C->>B: del B
    Note over B,C: B and C reconcile with each other<br/>while A is still in the air

    A->>B: ins X (hours late)
    A->>C: ins X
    B->>A: ins Y, del B (one batch, on reconnect)

    Note over A,C: the same three edits, a different arrival order at every replica<br/>SEC: all three must still read the same document
{{< /mermaid >}}

Four facts make this genuinely hard, and they are all consequences of choosing
availability:

- **No global order.** No clock and no leader that both replicas trust. *"Which edit came
  first?"* often has no answer at all.
- **Arbitrary delays.** Tuesday's edit can arrive after Friday's.
- **Local-first.** You cannot wait for consensus. Availability during a partition is the
  whole point.
- **Concurrency is genuine.** Any rule that resolves two mutually unaware edits must give
  the *same* answer at every replica, in any arrival order.

Sequential reasoning (*"apply the operations in order"*) is precisely what we are not
allowed to assume.

## 2. Indices are not identities

The obvious protocol is to ship what the editor already knows: a position and a
character.

```text
insert(index = 2, char = 'X')
delete(index = 0)
```

This is the vocabulary of `String.insert` and `splice`, the API every editor has. It is
also broken, because an index is *a coordinate in a document that is changing underneath
it*. It means something only relative to one replica's state at one instant.

### Break #1: two documents, forever

{{< mermaid >}}
flowchart TD
    S["AB — both replicas start here"]
    S -->|"A: ins(1,'X')"| L["AXB"]
    S -->|"B: ins(1,'Y')"| R["AYB"]
    L -->|"then applies ins(1,'Y')"| L2["AYXB"]
    R -->|"then applies ins(1,'X')"| R2["AXYB"]
{{< /mermaid >}}

Same two operations, same causal history, two different documents, and the replicas will
never agree again.

### Break #2: the index stops referring to anything

This one is worse, and it is a *different* failure: there are not two answers, there is no
answer at all.

{{< mermaid >}}
flowchart TD
    S["CAT — insert positions 0…3"]
    S -->|"A: del(0), del(0)"| L["T — insert positions 0…1"]
    S -->|"B: ins(3,'S') — meaning at the end"| R["CATS"]
    L -->|"now ins(3,'S') arrives at A"| X["position 3 does not exist"]
    X -->|"reject it"| X1["T — the author's S is silently lost"]
    X -->|"clamp to the end"| X2["TS — a position nobody asked for"]
{{< /mermaid >}}

`ins(3,'S')` was perfectly legal when it was issued. By the time it lands, `T` has two
positions. The operation is not wrong so much as *meaningless*.

{{< alert >}}
**Every repair here is a guess about intent.** A rule that forces the replicas to agree
just makes them agree on the guess. This is the first hint that **convergence was never
the goal.**
{{< /alert >}}

### The fix in principle: name characters, don't count them

| broken | fixed |
| --- | --- |
| `insert` *at position 2* | `insert` *after character* `id₇` |
| position 2 of *whose* document, *when*? | `id₇` means the same thing at every replica, forever |

Give every inserted character a globally unique, immutable identifier: a
`(timestamp, replica id)` pair, minted locally with no coordination. An insertion names
its **anchor**: the character it goes *after*. The operation is now *context-free*: it can
be applied at any replica, in any order, and still mean the same thing.

The document is no longer a string. It is a **tree of identities**, read out by a
deterministic traversal. That is what a sequence CRDT is.

## 3. CRDTs and RGA

A state-based CRDT is a state space with a merge

$$\mathsf{merge} : \Sigma \to \Sigma \to \Sigma$$

that is commutative, associative and idempotent, i.e. a join-semilattice. It is easy to see that this merge buys
**Strong Eventual Consistency**: any two replicas that have received the same set of
updates are in the same state, regardless of order, duplication or delay. 

**RGA** (Replicated Growable Array; Roh, Jeon, Kim, Lee, *JPDC* 71(3), 2011) is the
sequence CRDT underlying Automerge and Yjs. Its state is a grow-only set of records

$$(\mathit{id},\; \mathit{char},\; \mathit{afterId})$$

plus a grow-only set of **tombstones**: ids that have been deleted. `Insert` adds a
record anchored at an existing id; `Remove` adds an id to the tombstone set; `merge` is
componentwise union. Both components are grow-only, so merge is trivially a semilattice
join.

**All the interesting content is in the read.** The read is a depth-first traversal from
the root, where among *siblings* sharing an anchor, the **newest id goes first**:

{{< mermaid >}}
flowchart TD
    root(("⊥")) --> n1["1 · 'A'"]
    n1 --> n6["6 · 'Y'"]
    n1 --> n5["5 · 'X'"]
    n1 --> n2["2 · 'B'"]
{{< /mermaid >}}

*(Arrows read "is the anchor of"; the stored pointer runs the other way.)* Ids `6 > 5 > 2`
are all anchored at `'A'`, so the document reads `A Y X B` at **every** replica. Notice
that this is exactly what settles Break #1's race, deterministically and without a
tie-breaking authority.

In Sal, that order is an inductive relation `visible_lt` with four rules
(`RGA_ReadSide.lean`):

| rule | meaning |
| --- | --- |
| `parent_child` | an anchor comes before its children |
| `sibling` | among siblings, the larger id comes first |
| `left_desc_of_sib` | a sibling's whole subtree stays together |
| `trans` | transitive closure |

and three kernel-checked intent theorems on top of it, on both the CRDT and the MRDT:
`causal_order_visible_lt`, `tombstone_monotone_under_remove`,
`concurrent_insert_tiebreak_deterministic`.

### Real documents are rich text

Real documents are not plain text, it is rich text with **Bold**, _Italics_, [links](https://youtu.be/dQw4w9WgXcQ?si=T503gzS-bbNLpJOi), <span title="Anonymous Reviewer: like this one — You should click on the previous link" style="background-color:rgba(194,102,42,0.18);border-bottom:1px dashed #C2662A;cursor:help">comments</span>. So *formatting by index range* breaks for exactly the
reason that *inserting by index* breaks: concurrent inserts shift every index, so "bold
characters 5 to 14" denotes a different span at each replica.

**Peritext** (Litt, Lim, Kleppmann, van Hardenberg, CSCW 2022) fixes the span the same way
RGA fixes the insert: a mark is not a range of positions, it is a **pair of anchors**, each
naming a character *and a side*.

<figure>
<svg viewBox="0 0 700 112" role="img" aria-label="The character sequence 'The fox jumped' as tiles, each with an identity, with a bold mark drawn as a span bar anchored before character 5@A and after character 14@B" style="width:100%;height:auto">
  <g font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="14">
    <g fill="none" stroke="currentColor" stroke-opacity="0.35">
      <rect x="12"  y="42" width="30" height="40" rx="3"/>
      <rect x="48"  y="42" width="30" height="40" rx="3"/>
      <rect x="84"  y="42" width="30" height="40" rx="3"/>
      <rect x="120" y="42" width="30" height="40" rx="3"/>
      <rect x="156" y="42" width="30" height="40" rx="3"/>
      <rect x="192" y="42" width="30" height="40" rx="3"/>
      <rect x="228" y="42" width="30" height="40" rx="3"/>
      <rect x="264" y="42" width="30" height="40" rx="3"/>
      <rect x="300" y="42" width="30" height="40" rx="3"/>
      <rect x="336" y="42" width="30" height="40" rx="3"/>
      <rect x="372" y="42" width="30" height="40" rx="3"/>
      <rect x="408" y="42" width="30" height="40" rx="3"/>
      <rect x="444" y="42" width="30" height="40" rx="3"/>
      <rect x="480" y="42" width="30" height="40" rx="3"/>
    </g>
    <g fill="currentColor" text-anchor="middle">
      <text x="27"  y="68">T</text>
      <text x="63"  y="68">h</text>
      <text x="99"  y="68">e</text>
      <text x="171" y="68">f</text>
      <text x="207" y="68">o</text>
      <text x="243" y="68">x</text>
      <text x="315" y="68">j</text>
      <text x="351" y="68">u</text>
      <text x="387" y="68">m</text>
      <text x="423" y="68">p</text>
      <text x="459" y="68">e</text>
      <text x="495" y="68">d</text>
    </g>
    <g stroke="#4C63A8" stroke-width="2.5" stroke-linecap="round">
      <line x1="153" y1="32" x2="513" y2="32"/>
      <line x1="153" y1="24" x2="153" y2="40" stroke-width="1.5"/>
      <line x1="513" y1="24" x2="513" y2="40" stroke-width="1.5"/>
    </g>
    <g font-size="11" fill="#4C63A8" font-family="ui-sans-serif, system-ui, sans-serif">
      <text x="141" y="16">start: before 5@A</text>
      <text x="524" y="16" text-anchor="end">end: after 14@B</text>
    </g>
    <g font-size="10.5" fill="currentColor" fill-opacity="0.6" text-anchor="middle">
      <text x="27"  y="97">1@A</text>
      <text x="171" y="97">5@A</text>
      <text x="495" y="97">14@B</text>
    </g>
    <g font-size="11" fill="currentColor" fill-opacity="0.6" font-family="ui-sans-serif, system-ui, sans-serif">
      <text x="536" y="58">each character is a tile</text>
      <text x="536" y="74">with an identity</text>
    </g>
  </g>
</svg>
<figcaption>

A mark as a pair of anchors. The **side bit** (*before* vs *after*) is what decides
whether text typed at the boundary joins the span, and it is why bold expands while a
link contracts.

</figcaption>
</figure>

```json
{ "action":   "addMark",
  "markType": "bold",
  "start":    { "type": "before", "opId": "5@A" },
  "end":      { "type": "after",  "opId": "14@B" } }
```

Peritext is then just **RGA + a grow-only set of marks**. Alice's bold is the mark above,
on `fox jumped`; say Bob concurrently italicises `The fox`. The two overlap and simply merge;
the render is a left-to-right fold over reading order, opening and closing marks:

```text
[ {text: "The ",    format: {italic}},
  {text: "fox",     format: {bold, italic}},
  {text: " jumped", format: {bold}} ]
```

The merge is where the anchoring earns its keep. Alice bolds the whole sentence, and
concurrently Bob types a word into the middle of it:

<figure>
<svg viewBox="0 0 700 452" role="img" aria-label="Four rows of character tiles. Row one: 'The fox jumped' with a bold span bar anchored before 1@A and after 14@B. Row two: Bob's state, 'The brown fox jumped', with the six characters of 'brown ' drawn in orange as fresh ids 20@B to 25@B, and no mark. Row three: the merged state, all twenty characters bold, because the inserted word lies between the two anchors. Row four: the same merge had the mark been the index range 1 to 14, where the bar stops after 'fox ' and the word 'jumped' is left plain." style="width:100%;height:auto">
  <g font-family="ui-sans-serif, system-ui, sans-serif" font-size="12">
    <text x="12" y="18" fill="currentColor" fill-opacity="0.6">Alice bolds the whole sentence — one mark added, not a character touched</text>
    <g font-size="11" fill="#4C63A8">
      <text x="12" y="38">start: before 1@A</text>
      <text x="428" y="38" text-anchor="end">end: after 14@B</text>
    </g>
    <g stroke="#4C63A8" stroke-width="2.5" stroke-linecap="round">
      <line x1="12" y1="48" x2="428" y2="48"/>
      <line x1="12" y1="42" x2="12" y2="54" stroke-width="1.5"/>
      <line x1="428" y1="42" x2="428" y2="54" stroke-width="1.5"/>
    </g>
    <g fill="none" stroke="currentColor" stroke-opacity="0.35">
      <rect x="12"  y="58" width="26" height="34" rx="3"/>
      <rect x="42"  y="58" width="26" height="34" rx="3"/>
      <rect x="72"  y="58" width="26" height="34" rx="3"/>
      <rect x="102" y="58" width="26" height="34" rx="3"/>
      <rect x="132" y="58" width="26" height="34" rx="3"/>
      <rect x="162" y="58" width="26" height="34" rx="3"/>
      <rect x="192" y="58" width="26" height="34" rx="3"/>
      <rect x="222" y="58" width="26" height="34" rx="3"/>
      <rect x="252" y="58" width="26" height="34" rx="3"/>
      <rect x="282" y="58" width="26" height="34" rx="3"/>
      <rect x="312" y="58" width="26" height="34" rx="3"/>
      <rect x="342" y="58" width="26" height="34" rx="3"/>
      <rect x="372" y="58" width="26" height="34" rx="3"/>
      <rect x="402" y="58" width="26" height="34" rx="3"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="14" font-weight="700">
      <text x="25"  y="81">T</text><text x="55"  y="81">h</text><text x="85"  y="81">e</text>
      <text x="145" y="81">f</text><text x="175" y="81">o</text><text x="205" y="81">x</text>
      <text x="265" y="81">j</text><text x="295" y="81">u</text><text x="325" y="81">m</text>
      <text x="355" y="81">p</text><text x="385" y="81">e</text><text x="415" y="81">d</text>
    </g>
    <g font-size="10.5" fill="currentColor" fill-opacity="0.6" text-anchor="middle" font-family="ui-monospace, monospace">
      <text x="25"  y="106">1@A</text>
      <text x="115" y="106">4@A</text>
      <text x="415" y="106">14@B</text>
    </g>
    <text x="12" y="140" fill="currentColor" fill-opacity="0.6">Bob, concurrently — types <tspan font-family="ui-monospace, monospace">brown </tspan> after 4@A, having never seen the mark</text>
    <g fill="none" stroke="currentColor" stroke-opacity="0.35">
      <rect x="12"  y="152" width="26" height="34" rx="3"/>
      <rect x="42"  y="152" width="26" height="34" rx="3"/>
      <rect x="72"  y="152" width="26" height="34" rx="3"/>
      <rect x="102" y="152" width="26" height="34" rx="3"/>
      <rect x="312" y="152" width="26" height="34" rx="3"/>
      <rect x="342" y="152" width="26" height="34" rx="3"/>
      <rect x="372" y="152" width="26" height="34" rx="3"/>
      <rect x="402" y="152" width="26" height="34" rx="3"/>
      <rect x="432" y="152" width="26" height="34" rx="3"/>
      <rect x="462" y="152" width="26" height="34" rx="3"/>
      <rect x="492" y="152" width="26" height="34" rx="3"/>
      <rect x="522" y="152" width="26" height="34" rx="3"/>
      <rect x="552" y="152" width="26" height="34" rx="3"/>
      <rect x="582" y="152" width="26" height="34" rx="3"/>
    </g>
    <g fill="#C2662A" fill-opacity="0.10" stroke="#C2662A" stroke-opacity="0.7">
      <rect x="132" y="152" width="26" height="34" rx="3"/>
      <rect x="162" y="152" width="26" height="34" rx="3"/>
      <rect x="192" y="152" width="26" height="34" rx="3"/>
      <rect x="222" y="152" width="26" height="34" rx="3"/>
      <rect x="252" y="152" width="26" height="34" rx="3"/>
      <rect x="282" y="152" width="26" height="34" rx="3"/>
    </g>
    <g text-anchor="middle" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="14">
      <g fill="currentColor">
        <text x="25"  y="175">T</text><text x="55"  y="175">h</text><text x="85"  y="175">e</text>
        <text x="325" y="175">f</text><text x="355" y="175">o</text><text x="385" y="175">x</text>
        <text x="445" y="175">j</text><text x="475" y="175">u</text><text x="505" y="175">m</text>
        <text x="535" y="175">p</text><text x="565" y="175">e</text><text x="595" y="175">d</text>
      </g>
      <g fill="#C2662A">
        <text x="145" y="175">b</text><text x="175" y="175">r</text><text x="205" y="175">o</text>
        <text x="235" y="175">w</text><text x="265" y="175">n</text>
      </g>
    </g>
    <line x1="132" y1="194" x2="308" y2="194" stroke="#C2662A" stroke-opacity="0.6"/>
    <text x="220" y="208" text-anchor="middle" fill="#C2662A" font-size="10.5">six fresh ids, 20@B … 25@B</text>
    <text x="12" y="242" fill="currentColor" fill-opacity="0.6">merge — <tspan font-family="ui-monospace, monospace">chars ∪ chars</tspan>, <tspan font-family="ui-monospace, monospace">marks ∪ marks</tspan>. Both anchors still exist, so the mark still means something</text>
    <g font-size="11" fill="#4C63A8">
      <text x="12" y="262">before 1@A</text>
      <text x="608" y="262" text-anchor="end">after 14@B</text>
    </g>
    <g stroke="#4C63A8" stroke-width="2.5" stroke-linecap="round">
      <line x1="12" y1="272" x2="608" y2="272"/>
      <line x1="12" y1="266" x2="12" y2="278" stroke-width="1.5"/>
      <line x1="608" y1="266" x2="608" y2="278" stroke-width="1.5"/>
    </g>
    <g fill="none" stroke="currentColor" stroke-opacity="0.35">
      <rect x="12"  y="282" width="26" height="34" rx="3"/>
      <rect x="42"  y="282" width="26" height="34" rx="3"/>
      <rect x="72"  y="282" width="26" height="34" rx="3"/>
      <rect x="102" y="282" width="26" height="34" rx="3"/>
      <rect x="312" y="282" width="26" height="34" rx="3"/>
      <rect x="342" y="282" width="26" height="34" rx="3"/>
      <rect x="372" y="282" width="26" height="34" rx="3"/>
      <rect x="402" y="282" width="26" height="34" rx="3"/>
      <rect x="432" y="282" width="26" height="34" rx="3"/>
      <rect x="462" y="282" width="26" height="34" rx="3"/>
      <rect x="492" y="282" width="26" height="34" rx="3"/>
      <rect x="522" y="282" width="26" height="34" rx="3"/>
      <rect x="552" y="282" width="26" height="34" rx="3"/>
      <rect x="582" y="282" width="26" height="34" rx="3"/>
    </g>
    <g fill="#C2662A" fill-opacity="0.10" stroke="#C2662A" stroke-opacity="0.7">
      <rect x="132" y="282" width="26" height="34" rx="3"/>
      <rect x="162" y="282" width="26" height="34" rx="3"/>
      <rect x="192" y="282" width="26" height="34" rx="3"/>
      <rect x="222" y="282" width="26" height="34" rx="3"/>
      <rect x="252" y="282" width="26" height="34" rx="3"/>
      <rect x="282" y="282" width="26" height="34" rx="3"/>
    </g>
    <g text-anchor="middle" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="14" font-weight="700">
      <g fill="currentColor">
        <text x="25"  y="305">T</text><text x="55"  y="305">h</text><text x="85"  y="305">e</text>
        <text x="325" y="305">f</text><text x="355" y="305">o</text><text x="385" y="305">x</text>
        <text x="445" y="305">j</text><text x="475" y="305">u</text><text x="505" y="305">m</text>
        <text x="535" y="305">p</text><text x="565" y="305">e</text><text x="595" y="305">d</text>
      </g>
      <g fill="#C2662A">
        <text x="145" y="305">b</text><text x="175" y="305">r</text><text x="205" y="305">o</text>
        <text x="235" y="305">w</text><text x="265" y="305">n</text>
      </g>
    </g>
    <text x="12" y="332" fill="#1F7A4C" font-size="11.5">✓ the six new characters landed between the anchors, so they are inside the span — bold, with nobody asking for it</text>
    <text x="12" y="366" fill="currentColor" fill-opacity="0.6">the same merge, had the mark been the index range <tspan font-family="ui-monospace, monospace">1–14</tspan> instead of a pair of anchors</text>
    <g stroke="#C43333" stroke-width="2.5" stroke-linecap="round">
      <line x1="12" y1="384" x2="428" y2="384"/>
      <line x1="12" y1="378" x2="12" y2="390" stroke-width="1.5"/>
      <line x1="428" y1="378" x2="428" y2="390" stroke-width="1.5"/>
    </g>
    <g fill="none" stroke="currentColor" stroke-opacity="0.35">
      <rect x="12"  y="394" width="26" height="34" rx="3"/>
      <rect x="42"  y="394" width="26" height="34" rx="3"/>
      <rect x="72"  y="394" width="26" height="34" rx="3"/>
      <rect x="102" y="394" width="26" height="34" rx="3"/>
      <rect x="132" y="394" width="26" height="34" rx="3"/>
      <rect x="162" y="394" width="26" height="34" rx="3"/>
      <rect x="192" y="394" width="26" height="34" rx="3"/>
      <rect x="222" y="394" width="26" height="34" rx="3"/>
      <rect x="252" y="394" width="26" height="34" rx="3"/>
      <rect x="282" y="394" width="26" height="34" rx="3"/>
      <rect x="312" y="394" width="26" height="34" rx="3"/>
      <rect x="342" y="394" width="26" height="34" rx="3"/>
      <rect x="372" y="394" width="26" height="34" rx="3"/>
      <rect x="402" y="394" width="26" height="34" rx="3"/>
      <rect x="432" y="394" width="26" height="34" rx="3"/>
      <rect x="462" y="394" width="26" height="34" rx="3"/>
      <rect x="492" y="394" width="26" height="34" rx="3"/>
      <rect x="522" y="394" width="26" height="34" rx="3"/>
      <rect x="552" y="394" width="26" height="34" rx="3"/>
      <rect x="582" y="394" width="26" height="34" rx="3"/>
    </g>
    <g text-anchor="middle" font-family="ui-monospace, SFMono-Regular, Menlo, monospace" font-size="14">
      <g fill="currentColor" font-weight="700">
        <text x="25"  y="417">T</text><text x="55"  y="417">h</text><text x="85"  y="417">e</text>
        <text x="325" y="417">f</text><text x="355" y="417">o</text><text x="385" y="417">x</text>
      </g>
      <g fill="#C2662A" font-weight="700">
        <text x="145" y="417">b</text><text x="175" y="417">r</text><text x="205" y="417">o</text>
        <text x="235" y="417">w</text><text x="265" y="417">n</text>
      </g>
      <g fill="currentColor" fill-opacity="0.75">
        <text x="445" y="417">j</text><text x="475" y="417">u</text><text x="505" y="417">m</text>
        <text x="535" y="417">p</text><text x="565" y="417">e</text><text x="595" y="417">d</text>
      </g>
    </g>
    <text x="12" y="444" fill="#C43333" font-size="11.5">✗ 1–14 is now <tspan font-family="ui-monospace, monospace">The brown fox </tspan> — the range never moved, so the last six characters fall out of the span</text>
  </g>
</svg>
<figcaption>

Nothing in Alice's operation mentions `brown`; it could not, the word did not exist when
she issued it. The mark names only its two endpoints, so whatever lands between them in
reading order is inside the span. Six characters in, six characters off the end: the
bottom row loses exactly what Bob inserted.

</figcaption>
</figure>

The merge itself has nothing to arbitrate: `chars` and `marks` are both grow-only, so it
is componentwise union, exactly as in §3. The delicate case is not this one but text typed
*at* a boundary rather than strictly inside it, which is what the side bits decide, and
why bold grows to swallow it while a link does not.

## 4. MRDTs: git, but for data types

Everything above is a CRDT: `merge` sees two states. An **MRDT**'s merge additionally sees
their **lowest common ancestor**:

$$\mathsf{merge} : \Sigma \to \Sigma \to \Sigma \to \Sigma, \qquad \mathsf{merge}\;\ell\;a\;b$$

That third argument is worth a lot. With `ℓ` in hand, the merge can distinguish *"x was
never there"* from *"x was there and was deleted"*, which is exactly the information a
CRDT has to encode in tombstones. So **the LCA can pay for the tombstones instead.**

The canonical demonstration is OR-Set, which drops its remove-set entirely:

$$\mathsf{merge}\;\ell\;a\;b \;=\; (\ell \cap a \cap b) \;\cup\; (a \setminus \ell) \;\cup\; (b \setminus \ell)$$

{{< mermaid >}}
flowchart TD
    L["ℓ = {(1,a)}"]
    L -->|"A: Add a @ ts 2"| A["A = {(1,a), (2,a)}"]
    L -->|"B: Rem a"| B["B = { }"]
    A --> M["merge = {(2,a)} — a is live, add wins"]
    B --> M
{{< /mermaid >}}

Term by term: `ℓ∩a∩b = {}` (B removed it), `a∖ℓ = {(2,a)}` (the new tag), `b∖ℓ = {}`.
Every add stakes a fresh, globally unique id that each replica mints for itself, so
`(2,a)` is a *different tag* from `(1,a)`. `(1,a)` was in `ℓ`, so B's remove **observed**
it. `(2,a)` was not, so B *cannot* have seen it, and it survives.

{{< alert >}}
**The question this project starts from.** If the LCA can replace OR-Set's tombstones,
can it replace RGA's?
{{< /alert >}}

## 5. What "correct" means: Neem, Sal, RA-linearizability

To reason about correctness, we need a framework to prove things about our Replicated Data Types. **Neem** (OOPSLA'25) reduces the correctness of a data type to a fixed, discharge-able set of
obligations. You supply a signature

$$\langle\, \Sigma,\; \sigma_0,\; \mathsf{do},\; \mathsf{merge},\; \mathsf{rc} \,\rangle$$

where `rc` resolves *non-commuting* operation pairs (`Fst_then_snd` / `Snd_then_fst` /
`Either`). Neem's metatheorem: discharge **24 verification conditions** (some algebraic properties) over `do`, `merge`
and `rc` (`rc_non_comm`, `no_rc_chain`, `merge_comm`, `merge_idem`, `base_1op`,
`base_2op`, `ind_lca_2op`, `inter_left_1op`, `lem_0op`, …) and *replication-aware
linearizability* (the notion of correctness we will be working with) follows for every execution.

**Sal** is the Lean 4 port, plus a multi-modal tactic that stages automation by
*trustworthiness* (`dsimp`+`grind` → `lean-blaster`/Z3 → interactive), and a
counterexample pipeline that turns an *invalid* VC into an inspectable execution trace
(Plausible + ProofWidgets). The suite currently holds **29 RDTs** (17 CRDTs, 12 MRDTs)
and **648 VCs** for state convergence, the vast majority kernel-checked.

### RA-linearizability, in words

> Every state a replica can ever hold must be **explainable** as the result of running its
> operations **one at a time, in some order**, an order that respects causality and obeys
> the data type's own tie-breaking rule.

{{< mermaid >}}
flowchart LR
    D["the replica's event set E<br/>a partial order under vis"]
    D -->|"∃ π: a permutation of E<br/>respecting vis and rc"| P["π = o₁ · o₃ · o₂"]
    P -->|"applySeq from σ₀, using do_"| S["s — the replica's actual state"]
{{< /mermaid >}}

So a merged state is never "some third thing": it is always a sequential history someone
*could* have executed. In Lean (`Sal/ConditionedMRDTs/Metatheory/Adequacy.lean`):

```lean
def IsRALinearizable3 (C : Configuration D) : Prop :=
  ∀ (v : Version) (s : D.State) (E : Set (Op D.AppOp)),
    C.ver v = some (s, E) →
    ∃ π : List (Op D.AppOp),
      listPermOf π E ∧                              -- π enumerates exactly E
      respects π (lo (Configuration.core C)) ∧      -- causality + rc
      applySeq D.toCRDTSig D.init π = s             -- replaying π gives s
```

It is quantified over *every version in the configuration*: replica heads **and**
historical versions used as LCAs. `lo` is visibility together with the `rc` arbitration;
its acyclicity is exactly what `no_rc_chain` gives you.

{{< alert >}}
**Remember this shape.** The reference sequence is the data type's **own** `do`-fold.
Section 8 is about what that does *not* buy.
{{< /alert >}}

### Convergence is not enough

`merge ℓ a b := ∅` is commutative, associative, idempotent, and satisfies **SEC**. It is also
useless. So is "ignore all operations". Convergence is a *necessary* condition that any
number of wrong data types satisfy.

For sequences the problem is sharper, and it is worth being blunt about where in the suite
the VCs carry real content:

| Tier | Character | Examples | The 24 VCs prove… |
| --- | --- | --- | --- |
| **A** | state *is* the semantic content | LWW-/Max-/Min-Register, PN-Counter | lattice join, arithmetic: **real content** |
| **B** | merge does real computation | MVR, LWW-Map, LWW-Element-Set | conflict collection: **real content** |
| **C** | state is a grow-only bag; **meaning lives in the read** | OR-Set, **RGA**, Add-Win-PQ, **Peritext** | grow-only union: *near-trivial* |

Roughly half the suite, and *all* the interesting data types, are Tier C. For RGA, the 24
VCs prove that two grow-only sets converge under union. They never mention the traversal
that turns those sets into a document.

Hence the house rule: every RDT in Sal carries a `*_ReadSide.lean` companion, and the
Tier-C ones carry **intent-preservation theorems matched to their papers**. The honest
claim is intent preservation against a published specification, not *"proven correct"*.

## 6. Tombstones, and why you cannot just delete the record

A tombstone is a deleted character we cannot throw away, because other characters are
positioned relative to it.

<figure>
<svg viewBox="0 0 700 96" role="img" aria-label="A bar in which a small green segment marks live characters and a very long grey segment marks tombstones" style="width:100%;height:auto">
  <g font-family="ui-sans-serif, system-ui, sans-serif" font-size="12">
    <rect x="8"  y="30" width="52"  height="34" fill="#1F7A4C" fill-opacity="0.35" stroke="currentColor" stroke-opacity="0.4"/>
    <rect x="60" y="30" width="632" height="34" fill="currentColor" fill-opacity="0.09" stroke="currentColor" stroke-opacity="0.4"/>
    <text x="34"  y="22" text-anchor="middle" fill="currentColor" fill-opacity="0.75">live</text>
    <text x="376" y="22" text-anchor="middle" fill="currentColor" fill-opacity="0.75">tombstones</text>
    <text x="350" y="84" text-anchor="middle" fill="currentColor" fill-opacity="0.55">what the reader sees   vs.   what the replica stores</text>
  </g>
</svg>
<figcaption>

A heavily revised paper is mostly graveyard: state grows with the *total number of edits*,
not with the document size.

</figcaption>
</figure>

Garbage collection would need to know that *every* replica has seen the delete, which
needs consensus, which is exactly what we gave up in §1. So the goal is: make `Remove`
physically remove the record, so state is bounded by *live* characters. And the MRDT's LCA
is the extra information that might pay for it.

Why not simply delete the record, then? Because a survivor anchored at the deleted node is
left pointing at an id that is not in the state: its position becomes undefined, and it
has nowhere to render.

{{< alert >}}
**Tombstones in OR-Set and in RGA carry different kinds of information.** OR-Set's
tombstone is a *boolean* (membership history), so an LCA can recompute it. RGA's tombstone
is a *graph node* (structural position), and it is load-bearing geometry.
{{< /alert >}}

## 7. The rehoming RGA

`Sal/MRDTs/RGA_Rehoming/`. State is `map ℕ (α × ℕ)`: id ↦ (element, anchor). Deletion
really removes the record. Two halves, deliberately parallel:

**In `merge`: climb the LCA.** Survivorship is the OR-Set formula, yielding a survivor set
`I`. A survivor whose recorded anchor is dead walks *up the ancestor chain the LCA already
knows*, until it reaches a node that also survives, or the root.

```lean
let ancL := fun y => anc l y   -- the ancestor function, read out of the LCA

def climb_aux (ancL) (I) : ℕ → ℕ → ℕ
  | 0,      x => x
  | fuel+1, x => if x = 0 || I x then x
                 else climb_aux ancL I fuel (ancL x)
```

<figure>
<svg viewBox="0 0 700 262" role="img" aria-label="Top row: the chain root, P, Q, R, Y as the LCA knows it. Bottom row: the same chain after the merge, where Q and R are not in the survivor set and are drawn faint; an arc from P to Y shows Y climbing past the two dead nodes so that P becomes its anchor." style="width:100%;height:auto">
  <defs>
    <marker id="arrC" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 1 L 9 5 L 0 9 z" fill="currentColor" fill-opacity="0.5"/>
    </marker>
    <marker id="climbC" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 1 L 9 5 L 0 9 z" fill="#C2662A"/>
    </marker>
  </defs>
  <g font-family="ui-sans-serif, system-ui, sans-serif" font-size="12">
    <text x="12" y="24" fill="currentColor" fill-opacity="0.6">what the LCA ℓ knows — every node on the chain still has a position</text>
    <g stroke="currentColor" stroke-opacity="0.5" fill="none" marker-end="url(#arrC)">
      <line x1="74"  y1="64" x2="169" y2="64"/>
      <line x1="205" y1="64" x2="300" y2="64"/>
      <line x1="336" y1="64" x2="431" y2="64"/>
      <line x1="467" y1="64" x2="562" y2="64"/>
    </g>
    <circle cx="56"  cy="64" r="18" fill="none" stroke="currentColor" stroke-opacity="0.45" stroke-dasharray="3 3"/>
    <g fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8">
      <circle cx="187" cy="64" r="18"/>
      <circle cx="318" cy="64" r="18"/>
      <circle cx="449" cy="64" r="18"/>
      <circle cx="580" cy="64" r="18"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="13">
      <text x="56"  y="69" fill-opacity="0.55">⊥</text>
      <text x="187" y="69">P</text>
      <text x="318" y="69">Q</text>
      <text x="449" y="69">R</text>
      <text x="580" y="69">Y</text>
    </g>
    <text x="580" y="98" text-anchor="middle" fill="currentColor" fill-opacity="0.55" font-size="10.5">Y's recorded anchor is R</text>
    <text x="12" y="130" fill="currentColor" fill-opacity="0.6">after the merge — only P and Y are in the survivor set I</text>
    <g text-anchor="middle" font-size="10.5" font-family="ui-monospace, monospace">
      <text x="187" y="152" fill="#1F7A4C">∈ I</text>
      <text x="318" y="152" fill="currentColor" fill-opacity="0.45">∉ I</text>
      <text x="449" y="152" fill="currentColor" fill-opacity="0.45">∉ I</text>
      <text x="580" y="152" fill="#1F7A4C">∈ I</text>
    </g>
    <line x1="74" y1="180" x2="169" y2="180" stroke="currentColor" stroke-opacity="0.5" fill="none" marker-end="url(#arrC)"/>
    <g stroke="currentColor" stroke-opacity="0.28" fill="none" stroke-dasharray="3 4">
      <line x1="205" y1="180" x2="300" y2="180"/>
      <line x1="336" y1="180" x2="431" y2="180"/>
      <line x1="467" y1="180" x2="562" y2="180"/>
    </g>
    <circle cx="56"  cy="180" r="18" fill="none" stroke="currentColor" stroke-opacity="0.45" stroke-dasharray="3 3"/>
    <g fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8">
      <circle cx="187" cy="180" r="18"/>
      <circle cx="580" cy="180" r="18"/>
    </g>
    <g fill="none" stroke="currentColor" stroke-opacity="0.28" stroke-dasharray="3 3">
      <circle cx="318" cy="180" r="18"/>
      <circle cx="449" cy="180" r="18"/>
    </g>
    <g fill="currentColor" text-anchor="middle" font-size="13">
      <text x="56"  y="185" fill-opacity="0.55">⊥</text>
      <text x="187" y="185">P</text>
      <text x="318" y="185" fill-opacity="0.3">Q</text>
      <text x="449" y="185" fill-opacity="0.3">R</text>
      <text x="580" y="185">Y</text>
    </g>
    <path d="M 199 195 Q 383 262 568 195" fill="none" stroke="#C2662A" stroke-width="1.6" marker-end="url(#climbC)"/>
    <text x="383" y="250" text-anchor="middle" fill="#C2662A" font-size="11">climb from Y: R ∉ I, then Q ∉ I, then P ∈ I — so P becomes Y's anchor</text>
  </g>
</svg>
<figcaption>

Arrows read *is the anchor of*, so `Y`'s ancestor chain is that path walked backwards:
`R`, `Q`, `P`, `⊥`, exactly the order `climb_aux` tries. `merge` reads no operation and no
path: the dead nodes' positions are still in `ℓ`, and *that* is what pays for the
tombstone.

</figcaption>
</figure>

**In `do` there is no LCA.** A single replica applying an operation has no third state to
consult. So the operation *carries* its target's ancestor chain in its payload, and
`resolve` takes the first entry still live. Both halves together give `rc = Either` at
every reachable state (`rc_non_comm'`).

### It was proved, and the schema had to be widened

Commutation here holds only on **reachable** states: the operation's claimed path must
really be its ancestor chain (`accurate`), timestamps must be fresh, the root is never
stored. That is a *conditioned* commutativity, and the plain 24 VCs quantify over all
states. (A prefix-free variant of the design is outright impossible; see
`RGA_PrefixFree_Impossible.lean`.)

So we built the conditioned metatheory and proved the theorem directly:

{{< alert >}}
`rga_ra_linearizable3_eq`: RA-linearizability **up to observational ≈** at *every
reachable configuration*, under a single honest-delivery assumption. **Kernel-clean, 0
`sorry`.**
{{< /alert >}}


## 8. …and it is still wrong

Four operations on one replica, with no concurrency and no merge anywhere. Just a
backspace.

<figure>
<svg viewBox="0 0 720 326" role="img" aria-label="A birth tree with nodes 2, 1 and 3 reading as 2, 1, 3; after deleting node 1, node 3 rehomes to the root, where it outranks node 2 and becomes the root's first child, so the document reads 3, 2 instead of 2, 3" style="width:100%;height:auto">
  <defs>
    <marker id="arrD" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 1 L 9 5 L 0 9 z" fill="currentColor" fill-opacity="0.55"/>
    </marker>
    <marker id="climbD" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 1 L 9 5 L 0 9 z" fill="#C2662A"/>
    </marker>
  </defs>
  <g font-family="ui-sans-serif, system-ui, sans-serif" font-size="12">
    <text x="12" y="20" fill="currentColor" fill-opacity="0.6">before —
      <tspan font-family="ui-monospace, monospace">ins 1 after ⊥ ; ins 2 after ⊥ ; ins 3 after 1</tspan></text>
    <g stroke="currentColor" stroke-opacity="0.45" fill="none">
      <line x1="131" y1="64" x2="72" y2="79"/>
      <line x1="131" y1="108" x2="72" y2="93"/>
      <line x1="229" y1="116" x2="163" y2="116"/>
    </g>
    <circle cx="56"  cy="86"  r="16" fill="none" stroke="currentColor" stroke-opacity="0.45" stroke-dasharray="3 3"/>
    <circle cx="146" cy="56"  r="16" fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8"/>
    <circle cx="146" cy="116" r="16" fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8"/>
    <circle cx="246" cy="116" r="16" fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8"/>
    <g fill="currentColor" text-anchor="middle" font-size="13">
      <text x="56"  y="91" fill-opacity="0.55">⊥</text>
      <text x="146" y="61">2</text>
      <text x="146" y="121">1</text>
      <text x="246" y="121">3</text>
    </g>
    <rect x="360" y="68" width="150" height="36" rx="3" fill="none" stroke="currentColor" stroke-opacity="0.35"/>
    <text x="435" y="91" text-anchor="middle" fill="currentColor" font-family="ui-monospace, monospace">reads [2, 1, 3]</text>
    <text x="526" y="82" fill="currentColor" fill-opacity="0.6" font-size="11">at ⊥: ids 2 &gt; 1,</text>
    <text x="526" y="98" fill="currentColor" fill-opacity="0.6" font-size="11">so 2 comes first</text>
    <line x1="146" y1="142" x2="146" y2="178" stroke="currentColor" stroke-opacity="0.55" marker-end="url(#arrD)"/>
    <text x="160" y="164" fill="currentColor" font-family="ui-monospace, monospace">Del 1</text>
    <text x="215" y="164" fill="currentColor" fill-opacity="0.6">— one backspace</text>
    <text x="12" y="200" fill="currentColor" fill-opacity="0.6">after — record 1 is physically gone, so 3 rehomes to ⊥ and becomes its <tspan font-style="italic">first</tspan> child</text>
    <line x1="71" y1="268" x2="131" y2="290" stroke="currentColor" stroke-opacity="0.45"/>
    <line x1="71" y1="258" x2="131" y2="240" stroke="#C2662A" stroke-width="1.6" marker-end="url(#climbD)"/>
    <circle cx="56"  cy="262" r="16" fill="none" stroke="currentColor" stroke-opacity="0.45" stroke-dasharray="3 3"/>
    <circle cx="146" cy="236" r="16" fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8"/>
    <circle cx="146" cy="296" r="16" fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8"/>
    <g fill="currentColor" text-anchor="middle" font-size="13">
      <text x="56"  y="267" fill-opacity="0.55">⊥</text>
      <text x="146" y="241">3</text>
      <text x="146" y="301">2</text>
    </g>
    <text x="172" y="232" fill="#C2662A" font-size="11">3's anchor was 1; the climb</text>
    <text x="172" y="248" fill="#C2662A" font-size="11">rehomes it to ⊥</text>
    <rect x="360" y="204" width="150" height="34" rx="3" fill="#C43333" fill-opacity="0.07" stroke="#C43333" stroke-opacity="0.6"/>
    <text x="435" y="226" text-anchor="middle" fill="currentColor" font-family="ui-monospace, monospace">reads [3, 2]</text>
    <rect x="360" y="250" width="150" height="34" rx="3" fill="#1F7A4C" fill-opacity="0.07" stroke="#1F7A4C" stroke-opacity="0.6"/>
    <text x="435" y="272" text-anchor="middle" fill="currentColor" font-family="ui-monospace, monospace">should be [2, 3]</text>
    <text x="526" y="222" fill="currentColor" fill-opacity="0.6" font-size="11">⊥'s children are</text>
    <text x="526" y="238" fill="currentColor" fill-opacity="0.6" font-size="11">now {2, 3} —</text>
    <text x="526" y="254" fill="currentColor" fill-opacity="0.6" font-size="11">and 3 &gt; 2</text>
    <text x="526" y="278" fill="#C43333" font-size="11.5" font-weight="600">the survivors</text>
    <text x="526" y="294" fill="#C43333" font-size="11.5" font-weight="600">swapped</text>
  </g>
</svg>
<figcaption>

`3` was *buried* under `1`, so its timestamp never competed with `2`'s. The splice
promotes it to the root, where it wins the tiebreak, and text the user never touched
changes order.

</figcaption>
</figure>

This is a theorem rather than an anecdote, and it is stated against a specification
written *independently of the implementation*: a delete removes its target and nothing
else moves.

```lean
-- The independent intent spec.
def DeleteOrderPreserving : Prop :=
  ∀ (s : concrete_st) (t r x : ℕ) (p ids : List ℕ),
    List.Pairwise (· > ·) ids → (∀ i, contains s i = true → i ∈ ids) →
    document (do_ s (t, r, .Del p x)) (ids.filter (· ≠ x))
      = (document s ids).filter (· ≠ x)

theorem tombstone_free_violates_delete_order : ¬ DeleteOrderPreserving := by
  -- witness: s_bac with the complete, strictly-descending candidate list [3,2,1]
  ...  exact absurd key (by native_decide)
```

The candidate list is guarded **descending** and **complete**, so the discrepancy cannot be
an artifact of how we chose to read. There is a companion,
`del_a_breaks_survivor_order`, and, importantly, the positive contrast is a theorem too:
`remove_preserves_visible_lt` *holds* for the tombstoned RGA.

{{< alert >}}
**The rehoming RGA converges, is RA-linearizable, is kernel-clean, and does not implement
a text buffer.** A single backspace reorders text the user never touched.
{{< /alert >}}

### Why the proof did not see it

{{< mermaid >}}
flowchart LR
    DO["do_ — the step function"]
    DO -->|"merge ℓ a b"| M["what the replica computed"]
    DO -->|"fold of a witness sequence π"| F["the certified reference"]
    M -.->|"the VC checks only this ="| F
{{< /mermaid >}}

*Both sides are computed with the same `do_`. If `do_` is wrong, both sides are wrong the
same way, and the VC passes.*

Own-fold RA-linearizability certifies **convergence and self-consistency**. It does **not**
certify *fidelity to the intended sequence semantics*. This is the verification analogue of
*"your tests pass but the code is wrong"*: a proof checks itself against the spec you
wrote, and nothing checks the spec.

Catching it requires a specification independent of the implementation's own fold; here,
the published RGA's traversal order. We had hit the same failure mode once before, in
Peritext: `in_span_boundary` encoded the *opposite* of the paper's §3.3, and roughly 400
lines of perfectly correct proofs about it had to be deleted.

### Is it patchable? No, and that is informative

A fooling-pair argument settles it. Consider two executions whose *live* characters and
live geometry agree, but whose *deleted* history differs. RGA's traversal distinguishes
them. A tombstone-free state cannot.

{{< alert >}}
**No *bounded* tombstone-free state can preserve RGA order.** If the geometry you need is the dead nodes, you must either keep them, or
stop needing them.
{{< /alert >}}

There is also a structural reason the obvious repair, "let `merge` notice the orphan and
rewrite its anchor", cannot be made to work inside the framework, and the framework says
so. The VC

```lean
lem_0op : eq (merge (do_ l ol) (do_ a ol) (do_ b ol))
             (do_ (merge l a b) ol)
```

fails for `ol = Remove X` under *any* rehab scheme whose orphan test consults the merged
domain, because `do` and `merge` disagree about who is an orphan. Delete first, and `X` is
absent from the merged domain, so `Y` *is* an orphan and gets rewritten. Merge first, and
all three inputs still have `X`, so `Y` is *not* an orphan, and the outer delete then
strands it. The lattice-join merges the 24 VCs were designed around touch `ℓ` only through
*set algebra*; rehabilitation is a *structural query* on `ℓ`, and that is precisely what an
outer `do` can invalidate.

So the culprit is not the merge. It is the **representation**:

| | |
| --- | --- |
| `Y` stores a **mutable anchor**: *"my parent is X"* | a claim that gets **rewritten** |
| `Y` stores an **immutable coordinate**: *"I am here"* | nothing to rewrite |

Delete reorders survivors *because* survivors' positions are stored **relatively**, and a
relative position must be recomputed when its reference dies. Rewriting a position *is*
reordering.

**Fix the representation, not the merge. Make position a birth constant.**

## 9. EmbedRGA: immutable birth coordinates

`Sal/MRDTs/RGA_Embed/`. State is `map ℕ (α × List Bool)`: id ↦ (element, **coordinate**).
On insert, mint the newcomer's coordinate from its anchor's, once and forever:

$$\mathsf{coord}(t) \;=\; \mathsf{coord}(a) \,\Vert\, \Gamma.\mathsf{enc}(t-a)$$

where `a` is the anchor id, `t` the new timestamp, `t − a` the **delta**, and `Γ` any
*order-preserving, prefix-free* code.

<figure>
<svg viewBox="0 0 720 344" role="img" aria-label="The same document with absolute coordinates: node 2 is coordinate 2, node 1 is coordinate 1, node 3 is coordinate 1 comma 2. After deleting node 1, node 3's coordinate is untouched and the document reads 2, 3" style="width:100%;height:auto">
  <defs>
    <marker id="arrE" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 1 L 9 5 L 0 9 z" fill="currentColor" fill-opacity="0.55"/>
    </marker>
  </defs>
  <g font-family="ui-sans-serif, system-ui, sans-serif" font-size="12">
    <text x="12" y="20" fill="currentColor" fill-opacity="0.6">the same document, with <tspan font-style="italic">absolute</tspan> coordinates</text>
    <g stroke="currentColor" stroke-opacity="0.45" fill="none">
      <line x1="131" y1="64" x2="72" y2="79"/>
      <line x1="131" y1="108" x2="72" y2="93"/>
      <line x1="229" y1="116" x2="163" y2="116"/>
    </g>
    <circle cx="56"  cy="86"  r="16" fill="none" stroke="currentColor" stroke-opacity="0.45" stroke-dasharray="3 3"/>
    <circle cx="146" cy="56"  r="16" fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8"/>
    <circle cx="146" cy="116" r="16" fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8"/>
    <circle cx="246" cy="116" r="16" fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8"/>
    <g fill="currentColor" text-anchor="middle" font-size="13">
      <text x="56"  y="91" fill-opacity="0.55">⊥</text>
      <text x="146" y="61">2</text>
      <text x="146" y="121">1</text>
      <text x="246" y="121">3</text>
    </g>
    <g fill="#C2662A" font-size="11.5" font-family="ui-monospace, monospace">
      <text x="168" y="52">⟨2⟩</text>
      <text x="146" y="150" text-anchor="middle">⟨1⟩</text>
      <text x="268" y="120">⟨1,2⟩</text>
    </g>
    <rect x="380" y="68" width="150" height="36" rx="3" fill="none" stroke="currentColor" stroke-opacity="0.35"/>
    <text x="455" y="91" text-anchor="middle" fill="currentColor" font-family="ui-monospace, monospace">reads [2, 1, 3]</text>
    <line x1="146" y1="166" x2="146" y2="196" stroke="currentColor" stroke-opacity="0.55" marker-end="url(#arrE)"/>
    <text x="160" y="186" fill="currentColor" font-family="ui-monospace, monospace">Del 1</text>
    <text x="215" y="186" fill="currentColor" fill-opacity="0.6">— no path, no rehoming</text>
    <text x="12" y="210" fill="currentColor" fill-opacity="0.6">after — the record is gone; every survivor's coordinate is a birth constant</text>
    <g stroke="currentColor" stroke-opacity="0.28" fill="none" stroke-dasharray="2 3">
      <line x1="71"  y1="284" x2="131" y2="318"/>
      <line x1="162" y1="312" x2="230" y2="312"/>
    </g>
    <line x1="131" y1="262" x2="72" y2="274" stroke="currentColor" stroke-opacity="0.45"/>
    <circle cx="56"  cy="278" r="16" fill="none" stroke="currentColor" stroke-opacity="0.45" stroke-dasharray="3 3"/>
    <circle cx="146" cy="248" r="16" fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8"/>
    <circle cx="146" cy="312" r="16" fill="none" stroke="currentColor" stroke-opacity="0.28" stroke-dasharray="3 3"/>
    <circle cx="246" cy="312" r="16" fill="#4C63A8" fill-opacity="0.12" stroke="#4C63A8"/>
    <g fill="currentColor" text-anchor="middle" font-size="13">
      <text x="56"  y="283" fill-opacity="0.55">⊥</text>
      <text x="146" y="253">2</text>
      <text x="146" y="317" fill-opacity="0.28">1</text>
      <text x="246" y="317">3</text>
    </g>
    <g fill="#C2662A" font-size="11.5" font-family="ui-monospace, monospace">
      <text x="168" y="244">⟨2⟩</text>
      <text x="268" y="316">⟨1,2⟩</text>
    </g>
    <text x="330" y="316" fill="#1F7A4C" font-size="11.5">3's coordinate is untouched</text>
    <rect x="380" y="246" width="150" height="36" rx="3" fill="#1F7A4C" fill-opacity="0.07" stroke="#1F7A4C" stroke-opacity="0.6"/>
    <text x="455" y="269" text-anchor="middle" fill="currentColor" font-family="ui-monospace, monospace">reads [2, 3]</text>
  </g>
</svg>
<figcaption>

The same witness that broke the rehoming design. `Del` carries no path and rehomes
nothing; `merge` never climbs, and is just OR-Set survival with values copied. The delete
is a **filter**.

</figcaption>
</figure>

A pleasant side effect: `do` on `Ins` **never reads the state**, so every operation pair
commutes unconditionally enough that `rc = Either` discharges with no path algebra at all.

### Sorting coordinates *is* the RGA traversal

Compare two coordinates as delta sequences (`chainBefore`):

1. If one is a **prefix** of the other, the prefix comes first.
   → this is RGA's `parent_child` rule.
2. Otherwise, at the **first differing delta**, the **larger** delta comes first.
   → this is RGA's `sibling` rule (a larger delta under a shared anchor means a newer
   timestamp).

Reading the figure above: `⟨2⟩ ≺ ⟨1⟩ ≺ ⟨1,2⟩`, so the document is `[2,1,3]`; after the
delete, `[2,3]`. **Order preserved.**

Mechanically it is *one lexicographic compare*. Concatenate the codewords, append a
terminator symbol above the digit alphabet, and sort descending:

```lean
def key (c : coord) : List ℕ :=
  (c.map fun b => if b then 2 else 1) ++ [3]
```

The terminator is the trick that makes rule 1 fall out of rule 2: where one key ends, it
shows a `3`, which beats any digit the longer key can offer, so a prefix always sorts
ahead of its extensions. The supporting theorems:

| theorem | what it says |
| --- | --- |
| `display_iff_chainBefore` | the key comparison computes *exactly* `chainBefore` |
| `coordOf_inj` | prefix-freeness gives unique decodability: distinct birth chains, distinct coordinates |
| `subtree_convex` | a subtree is a coordinate prefix, hence a *contiguous* block of the display: **non-interleaving, for free** |
| `before_do_stable` | **axiom-free** step stability; at `Del` this *is* general delete-order preservation, the clause the rehoming RGA refutes |
| `before_merge_stable` | pairwise display order is stable at merges too, from value immutability alone |


### The two capstones


**1. Convergence.** `embed_ra_linearizable3`: RA-linearizability *per version* at every
honestly reachable configuration, with **strict `=`**, not `≈`.

Route: the *mergeable-queue* hook, whose join needs canonical states unique per event set.
Supplied by `e_fold_canon`: *any two well-formed enumerations of one event set fold to the
same state.*

**2. Fidelity.** `rga_read_eq_embed_read`, **the compaction theorem.** On honest
executions, EmbedRGA's read equals the *published, tombstoned* RGA's read, **element for
element.**

The second one is the point of the whole exercise: **the specification is now external.**
It is RGA's own `visible_lt`, not our fold. A by-product the published side never proved
falls out along the way. `visible_lt_total` says RGA's relational order *is* total on the
birth tree.

Two further notes on the engineering:

- The development is **parametric in the code**. **Unary**, **binary-δ** and **Elias-δ**
  instances are all proved, with zero new proof content each: every datatype theorem
  consumes only `OrderedPrefixCode` (monotone + prefix-free). Coordinates cost roughly
  1.4 to 1.8 bits per level on real traces.
- Axioms used: `propext`, `Classical.choice`, `Quot.sound`. No `sorryAx`.

**Tombstone-free, and provably the same document as RGA.**

## 10. Rich text inherits the fix, and inherited the defect

The *fused* design puts the mark boundaries **into** the sequence, so rich text is **one**
RGA:

```lean
inductive PeritextElt
  | char  : ℕ → PeritextElt
  | bound : ℕ → Mark → Bool → PeritextElt
--          markId  mark   isStart
```

Because the payload is *opaque* to the kernel, convergence is a one-line instantiation
with no new proof:

```lean
theorem peritextEmbed_ra_linearizable3
    (hReach : EReach Γ C) : IsRALinearizable3 C :=
  embed_ra_linearizable3 hReach
```

And the defect of §8 reappears here in the form a *user* would actually notice. Delete one
plain character on the rehoming kernel, and an untouched character silently changes
formatting:

<figure>
<svg viewBox="0 0 700 224" role="img" aria-label="A fused Peritext sequence: bold-open, X, bold-close, P, C. After deleting the plain character P, the character C moves inside the bold span and becomes bold." style="width:100%;height:auto">
  <defs>
    <marker id="arrP" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 1 L 9 5 L 0 9 z" fill="currentColor" fill-opacity="0.55"/>
    </marker>
  </defs>
  <g font-family="ui-sans-serif, system-ui, sans-serif" font-size="12">
    <text x="12" y="55" fill="currentColor" fill-opacity="0.6">before</text>
    <g fill="none">
      <rect x="60"  y="34" width="46" height="40" rx="3" stroke="#4C63A8" stroke-opacity="0.7"/>
      <rect x="114" y="34" width="30" height="40" rx="3" stroke="currentColor" stroke-opacity="0.35"/>
      <rect x="152" y="34" width="46" height="40" rx="3" stroke="#4C63A8" stroke-opacity="0.7"/>
      <rect x="206" y="34" width="30" height="40" rx="3" stroke="currentColor" stroke-opacity="0.35"/>
      <rect x="244" y="34" width="30" height="40" rx="3" stroke="currentColor" stroke-opacity="0.35"/>
    </g>
    <g text-anchor="middle" font-family="ui-monospace, monospace">
      <text x="83"  y="59" fill="#4C63A8" font-size="11">⟨b⟩</text>
      <text x="129" y="59" fill="currentColor" font-size="13">X</text>
      <text x="175" y="59" fill="#4C63A8" font-size="11">⟨/b⟩</text>
      <text x="221" y="59" fill="currentColor" font-size="13">P</text>
      <text x="259" y="59" fill="currentColor" font-size="13">C</text>
    </g>
    <g text-anchor="middle" font-size="10" fill="currentColor" fill-opacity="0.55">
      <text x="83"  y="88">1</text><text x="129" y="88">2</text><text x="175" y="88">4</text>
      <text x="221" y="88">3</text><text x="259" y="88">5</text>
    </g>
    <line x1="110" y1="24" x2="148" y2="24" stroke="#4C63A8" stroke-width="2.5" stroke-linecap="round"/>
    <text x="300" y="55" fill="currentColor" font-size="11.5">renders: X <tspan font-weight="700">bold</tspan>; P, C plain — the span is exactly {X}</text>
    <line x1="129" y1="100" x2="129" y2="128" stroke="currentColor" stroke-opacity="0.55" marker-end="url(#arrP)"/>
    <text x="142" y="119" fill="currentColor" font-family="ui-monospace, monospace">Del P</text>
    <text x="196" y="119" fill="currentColor" fill-opacity="0.6">— one plain character</text>
    <text x="12" y="169" fill="currentColor" fill-opacity="0.6">after</text>
    <g fill="none">
      <rect x="60"  y="148" width="46" height="40" rx="3" stroke="#4C63A8" stroke-opacity="0.7"/>
      <rect x="114" y="148" width="30" height="40" rx="3" stroke="currentColor" stroke-opacity="0.35"/>
      <rect x="152" y="148" width="30" height="40" rx="3" stroke="currentColor" stroke-opacity="0.35"/>
      <rect x="190" y="148" width="46" height="40" rx="3" stroke="#4C63A8" stroke-opacity="0.7"/>
    </g>
    <g text-anchor="middle" font-family="ui-monospace, monospace">
      <text x="83"  y="173" fill="#4C63A8" font-size="11">⟨b⟩</text>
      <text x="129" y="173" fill="currentColor" font-size="13">X</text>
      <text x="167" y="173" fill="currentColor" font-size="13">C</text>
      <text x="213" y="173" fill="#4C63A8" font-size="11">⟨/b⟩</text>
    </g>
    <g text-anchor="middle" font-size="10" fill="currentColor" fill-opacity="0.55">
      <text x="83"  y="202">1</text><text x="129" y="202">2</text>
      <text x="167" y="202">5</text><text x="213" y="202">4</text>
    </g>
    <line x1="110" y1="138" x2="186" y2="138" stroke="#4C63A8" stroke-width="2.5" stroke-linecap="round"/>
    <text x="300" y="169" fill="#C43333" font-size="11.5">renders: X <tspan font-weight="700">bold</tspan>, C <tspan font-weight="700">bold</tspan> — an untouched character</text>
    <text x="300" y="185" fill="#C43333" font-size="11.5">changed formatting</text>
  </g>
</svg>
<figcaption>

`C` rehomes onto `X`, out-ranks the close boundary (`5 > 4`) and leapfrogs *inside* the
span. In Lean: `fused_delete_reformats_survivor`.

</figcaption>
</figure>

On the embed kernel there is nothing to say, which is the point: `renderIds_del` states
that the post-delete render is the pre-delete render **minus exactly the deleted entries**,
so the formatting of every survivor is preserved bitwise.

Both shapes are in the repo: marks in a separate set (as published), and marks fused into
the sequence. The fused one rides the embed kernel for free.


## 12. What we learned

**1. Indices are not identities.** Collaborative editing works because characters are
*named*, and the document is a tree read by a deterministic traversal.

**2. Convergence is a substrate, not correctness.** "All replicas agree on the empty
document" satisfies SEC. For grow-only sequence state, the 24 VCs prove that union
commutes; the *meaning* lives entirely in the read.

**3. The specification must be external.** Own-fold RA-linearizability cannot see a broken
`do`, because both sides of *merge = fold* use it. Fidelity needed a theorem against the
**published** RGA.

**4. Mechanization pays in refutations.** It produced a machine-checked *negative*: a
proved, converging, RA-linearizable tombstone-free RGA that reorders text under a single
backspace, plus seven over-strong premises and a criss-cross countermodel we would never
have found on paper.

**5. Design for the proof.** Making position an *immutable birth constant* turned `Del`
into a filter, `merge` into a lattice join, and the read into a sort. Only then did both a
strict-equality convergence proof *and* read-equivalence with the published RGA become
available.

## Pointers

| | |
| --- | --- |
| the framework | [Sal](https://github.com/fplaunchpad/sal), a Lean 4 port of **Neem** (F\*, OOPSLA 2025) |
| the paper | [kcsrk.info/papers/sal_jan26.pdf](https://kcsrk.info/papers/sal_jan26.pdf) |
| the designs | `Sal/MRDTs/RGA_with_tombstones/`, `RGA_Rehoming/`, `RGA_Embed/` |
| the defect | `Sal/MRDTs/RGA_Rehoming/RGA_Tombstone_Free_SPOT.lean` |
| the fix | `RGA_Embed_ChainLex.lean`, `RGA_Embed_ReadEquiv.lean` |
| the capstones | `Sal/ConditionedMRDTs/MRDT_Instances/EmbedRGA/` |

**References.** Roh, Jeon, Kim, Lee, *Replicated abstract data types*, JPDC 71(3), 2011 ·
Shapiro, Preguiça, Baquero, Zawirski, *A comprehensive study of CRDTs*, INRIA RR-7506,
2011 · Litt, Lim, Kleppmann, van Hardenberg, *Peritext*, CSCW 2022 · Soundarapandian,
Nagar, Rastogi, Sivaramakrishnan, OOPSLA 2025.

---

*With thanks to Prof. KC Sivaramakrishnan for the supervision, and to Vimala
Soundarapandian and Pranav Ramesh. FP Launchpad, IIT Madras.*
