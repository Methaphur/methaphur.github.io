---
title: "Lean for the Working Mathematician"
date: 2026-11-23T00:00:00+05:30
draft: false
description: "Notes from an introductory Lean workshop at NISER, and what Lean's kernel does and does not guarantee about the Lean proof of the Riemann Hypothesis."
summary: "We ran an introductory Lean workshop at NISER, the same week a Lean proof of the Riemann Hypothesis was released. Here is what we covered, from Curry–Howard to tactics, and a longer answer to the best question of the day: who checks the checker?"
tags: ["Lean", "Formal Verification", "Theorem Proving", "Riemann Hypothesis", "Workshop"]
categories: ["Notes"]
showHero: true
heroStyle: "big"
featureimagecaption: "The Riemann Hypothesis, as stated in Mathlib."
alt: "Mathlib's Lean definition of RiemannHypothesis: for every complex s with riemannZeta s = 0, s not a trivial zero -2(n+1), and s ≠ 1, the real part of s is 1/2."
---

{{< katex >}}

*The title is my friend Aneesh's idea, a reference to Mac Lane's* Categories for the Working Mathematician.

I was halfway through writing up our Lean workshop when [Levent Alpöge](https://alpo.ge/) released a proof of the Riemann Hypothesis.

On 20 November 2026 [he announced it](https://alpo.ge/remark.pdf) as a Lean library of about 17 million lines of code, written with Claude. A day later, the Lean and Mathlib communities published a [joint statement](https://leanprover-community.github.io/blog/posts/riemann-hypothesis/), endorsed by the Lean FRO and the Mathlib Initiative:

> We have full confidence that Alpöge's proof of the Riemann Hypothesis is correct.

and, a little further down:

> We hereby certify that both the statement and the proof of the Riemann Hypothesis are correct.

We have all watched the huge amount of maths and CS results that have come out recently with the help of AI (or rather, LLMs). One name kept coming along with them, and it isn't unfamiliar by now: **Lean**.

I've seen countless versions of the same objection: *"17 million lines of AI-generated code, so obfuscated that no mathematician can read it. How do we know it isn't hallucinated? What's the guarantee?"*

The answer is that Lean *is* the guarantee. By the nature of Lean, if it checks, it is true. We can be pedantic about whether we actually proved RH or just a syntactically correct statement that isn't RH, and that's a fair worry. I'll come back to it [later in this post](#did-we-actually-prove-rh).

## The workshop

To make Lean a little less alien to our friends and faculty at [NISER](https://niser.ac.in), Aneesh Ukidve (B22 Int. MSc), [Dr.Anamay Tengse](https://anamay.bitbucket.io/) and I ran an introductory Lean workshop on Saturday, 14 November, in the SMS Seminar Hall. There were three sessions, each building on the one before:

1. **Programs and types** (Anamay)
2. **Why Lean Works?** (Aneesh)
3. **Hands-on Lean** (me)

This post walks through what we covered, and then answers the best question we got on the day.

## Let's talk about Lean

So how exactly does Lean prove things? Before we get there, we need a bit of functional programming and type theory, which is where Anamay started.

### Two ways to compute

In the 1930s there were two answers to the question *"what does it mean to compute something?"*

Turing gave us the machine: a tape, a head, a table of rules. That's the model most of us grew up with, and it's roughly how the computer in front of you works.

At the same time, Alonzo Church came up with the **λ-calculus**, where everything is a function, functions take functions as inputs, and computing means simplifying expressions. It turned out to be exactly as powerful as Turing's machines. It is also the root of functional programming, and of Lean.

Add **types** to it: every value has a type, and functions are values too, with types like `ℕ → ℕ`. Now you have the setting Lean lives in.

### Logic World

Aneesh then switched to logic. He started with Hilbert-style deductive systems: a few axioms and modus ponens. Correct, but painful to use.

Then he moved to **natural deduction**, which is much closer to how we actually write proofs. Every connective comes with two kinds of rule:

- an **introduction rule**, which says how to *prove* it;
- an **elimination rule**, which says how to *use* it.

In Gentzen's notation, a rule is a line: the premises sit above it and the conclusion below. To prove \(A \land B\), prove \(A\) and prove \(B\). To use \(A \land B\), take out either half:

$$
\frac{A \qquad B}{A \land B}\;(\land\text{I})
\qquad\qquad
\frac{A \land B}{A}\;(\land\text{E}_1)
\qquad\qquad
\frac{A \land B}{B}\;(\land\text{E}_2)
$$

Implication works the same way. To prove \(A \to B\), assume \(A\) and derive \(B\), then discharge the assumption \([A]\). To use \(A \to B\), feed it a proof of \(A\), which is just modus ponens:

$$
\frac{\begin{matrix}[A] \\ \vdots \\ B\end{matrix}}{A \to B}\;(\to\text{I})
\qquad\qquad
\frac{A \to B \qquad A}{B}\;(\to\text{E})
$$

Rules stack into trees, and a proof is a tree with the theorem at the root. Here is \(A \land B \to B \land A\):

$$
\dfrac{\dfrac{\dfrac{[A \land B]}{B}\;(\land\text{E}_2) \qquad \dfrac{[A \land B]}{A}\;(\land\text{E}_1)}{B \land A}\;(\land\text{I})}{A \land B \to B \land A}\;(\to\text{I})
$$

Aneesh then drew the bridge between the logic world and the proof world with the **Curry–Howard correspondence**: *propositions are types, and proofs are programs.* The rules above are exactly the rules for building and taking apart pairs, tagged unions and functions in a typed programming language. \(\land\)I builds a pair, \(\land\)E projects out of one, \(\to\)I writes a function, and \(\to\)E applies it.

| Logic | Type |
| --- | --- |
| `A ∧ B` | pairs, `A × B` |
| `A ∨ B` | tagged union, `A ⊕ B` |
| `A → B` | functions `A → B` |
| `False` | the empty type, `Empty` |
| `∀ x, P x` / `∃ x, P x` | Π-types / Σ-types |
| `¬A` | `A → False` |

## The hands-on session

My session put all of that into Lean.

### Propositions really are types

We started by going back to Curry–Howard with a tiny example, to convince everyone that propositions really are types:

```lean
def     iden (n : Int)   : Int   := n

theorem th   (h : 2 = 2) : 2 = 2 := h   -- same shape as iden!
```

`iden` takes an integer and returns it. `th` takes a proof of `2 = 2` and returns it. In Lean, *"`h` has type `2 = 2`"* and *"`h` is a proof of `2 = 2`"* mean the same thing. A `theorem` is just a `def` whose type happens to be a statement.

Break either one and you get the *same* "type mismatch" error:

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr)); gap: 0 1rem;">
<div>

```lean
def iden (n : Int) : Int := "five"
```

```text
error: type mismatch
  "five"
has type
  String : Type
but is expected to have type
  Int : Type
```

</div>
<div>

```lean
theorem th (k : 3 = 3) : 2 = 2 := k
```

```text
error: type mismatch
  k
has type
  3 = 3 : Prop
but is expected to have type
  2 = 2 : Prop
```

</div>
</div>

A wrong proof is just a type error. The type checker that checks programs also checks proofs.

Then a couple of real ones:

```lean
theorem and_swap (p q : Prop) : p ∧ q → q ∧ p := fun h => ⟨h.2, h.1⟩
theorem or_swap (p q : Prop) : p ∨ q → q ∨ p := fun h =>
  match h with
  | Or.inl hp => Or.inr hp
  | Or.inr hq => Or.inl hq
```

A proof of *and* is a pair, so to swap the statement you swap the pair. A proof of *or* is tagged left or right, so you look at the tag and re-tag it on the other side.

Modus ponens is just function application:

```lean
theorem modus_ponens (p q : Prop) : p → (p → q) → q := fun hp hpq => hpq hp
```

A proof `hpq` of `p → q` is a function that turns proofs of `p` into proofs of `q`, so feeding it `hp` gives a proof of `q`. *Using* a theorem literally means *calling* a function. (These examples are from my [slides](https://github.com/Methaphur/NISER-Lean-Workshop/blob/main/Slides/Lean-Workshop-S3.pdf).)

### The kernel is small

We only touched on this briefly, but it matters for the rest of the post. What you type into Lean goes through the **elaborator**, which fills in implicit arguments, runs tactics, and so on. The elaborator is huge and clever and might have bugs, and **you don't have to trust it**.

What comes out the other end is a plain proof term, like the ones above. Only the **kernel** decides whether that term is accepted, and the kernel is deliberately small: small enough for us to independently check and verify. People can read and audit it line by line, write their own kernels to re-check the same proofs, and even formally verify a kernel, as we'll see [later](#what-changed-more-kernels-some-of-them-verified). This design is called the *de Bruijn criterion*.

It's also why it doesn't matter who wrote the proof: you, a tactic, or an AI. They all have to get past the same small kernel.

### Tactics

Writing proof terms by hand gets painful fast, so Lean lets you build them one step at a time with **tactics**. Tactics are programs (proof automation tools) that construct the proof term for you.

We went through ten of them, one card each: `exact`, `intro`, `apply`, `constructor`, `obtain`, `left`/`right`, `rcases`, `exists`, `rw` and `suffices`. When we actually prove *Read the goal after every step*: the infoview always tells you what's left to prove.

Here's `and_swap` again, written with tactics:

```lean
theorem and_swap' (p q : Prop) : p ∧ q → q ∧ p := by
  intro h
  obtain ⟨hp, hq⟩ := h
  constructor
  · exact hq
  · exact hp
```

Run `#print and_swap'` and you'll see a term again:

```lean
theorem and_swap' : ∀ (p q : Prop), p ∧ q → q ∧ p :=
fun p q h => And.casesOn h fun hp hq => ⟨hq, hp⟩
```

The arguments `(p q : Prop)` have become a `∀`, and each tactic left its mark on the term. `intro h` became `fun ... h =>`. `obtain ⟨hp, hq⟩ := h` became `And.casesOn h fun hp hq => ...`, which takes the pair apart. `constructor` with the two `exact`s built the new pair `⟨hq, hp⟩`. The tactics were just a nicer way of writing it, and the kernel checks the term, not the tactics.

Unfortunately we didn't have time to build the natural numbers from scratch or get into Mathlib, which is where the real maths lives. That's something for a future session.

## "But who checks the checker?"

The best question of the day came from one of the professors in the audience. [TODO: name them, or keep it anonymous? Check with them first.]

If everything rests on the kernel, how do we know the kernel is type-checking correctly?

We talked about the **Collatz incident** from this summer, which I'll summarise below. Their follow-up was a sharp one. Collatz was a glamorous result, so lots of people looked at it closely and the bug was found. But what about some ordinary, boring lemma that gets "proven", maliciously or innocently, through a kernel bug? Nobody is going to stare at that.

Here's my longer answer, mostly drawn from two posts by [Leo de Moura](https://leodemoura.github.io/), who created Lean.

### What happened with Collatz

On 25 July 2026, Ramana Kumar published a repository with a `sorry`-free "disproof" of the Collatz conjecture, produced with AI assistance. It wasn't a disproof. It exploited a bug in how the kernel handled a corner case of nested inductive types.

Three days later Kiran Gopinathan cut it down to a short proof of `False` and reported it. A fix was pushed within an hour. [Leo's postmortem](https://leodemoura.github.io/blog/2026-8-1-postmortem-for-kernel-soundness-bug-14576/) is worth reading in full, but three things stand out.

**1. It was an implementation bug, not a flaw in the logic.** In Leo's words:

> This is an implementation bug, not a hole in Lean's meta-theory.

It was also only reachable through metaprogramming, by handing a declaration straight to the kernel. You wouldn't hit it by writing ordinary proofs.

**2. It needed two different bugs at once.** The proof also passed a week-old version of **nanoda**, an independent Lean checker written in Rust by Chris Bailey. But it got through because of a *second, unrelated* bug in nanoda, which had already been reported and fixed by then. Fooling both checkers took two separate mistakes in two separate implementations, and the current version of nanoda rejected it.

Afterwards, Daniel Selsam at OpenAI, working with the Lean FRO, used a security-focused AI to hunt for more kernel bugs. It found several, all fixed and all metaprogramming-only, and Leo notes that *"All of them were caught by nanoda."*

**3. Leo rejected "lock down the elaborator" as the fix:**

> The elaborator is untrusted by design. Soundness cannot depend on an untrusted component refusing to build a bad term.

Nobody has to behave nicely for the checking to work. Only the kernel needs to be right.

### What changed: more kernels, some of them verified

In October, Leo announced that [Lean now ships with verified kernels](https://leodemoura.github.io/blog/2026-10-15-lean-now-ships-with-verified-kernels/). Since Lean v4.35:

- `lake check` exports your project and re-checks it, in a sandbox, with `leanchecker`, the official kernel without everything else around it.
- `lake check --paranoid` re-checks it with **four more kernels**: nanoda, lean4lean, con-leche and con-ron. It also adds a build of the official kernel with a hardened memory allocator, so memory corruption becomes a crash instead of a silently wrong answer.

Two of these are new and special:

- **con-leche** is a Lean kernel written in Lean and *proven consistent*: every environment it accepts has a model in set theory, following Mario Carneiro's model of Lean's type theory. It was written by Claude, under heavy supervision from Joachim Breitner at the Lean FRO.
- **con-ron** is a Rust translation of con-leche, with a proof (using a tool called Aeneas) that the Rust version refines con-leche's checking algorithm.

To be precise: the official C++ kernel is small and auditable, but it isn't formally verified itself. lean4lean's verification is expected "in the next few months". nanoda is independent but not verified. And Leo is upfront that even con-leche "is not bulletproof, because there might be bugs in the compiler and runtime used to build it".

That's exactly why you run more than one:

> It is very unlikely that the same bug also exists in the Rust compiler or runtime, so by checking a proof with both con-leche and con-ron, you gain a high level of protection against this class of bugs.

### So, the boring lemma

**Checking is mechanical, and that includes the boring stuff.** A referee might skim a dull lemma. `lake check --paranoid` doesn't know what's dull. The forgettable lemma is checked exactly as carefully as RH, by the same small kernel and four independent ones, written in different languages and two of them formally verified. Nobody has to find the result interesting for it to get checked.

**A false proof would have to fool every checker in the same way.** Bugs happen; Collatz shows that. But a bad proof has to get past several implementations, each with its own bugs, *in exactly the same way*. Even Collatz needed two unrelated bugs lined up at the right time, and current versions caught it. The chance isn't zero, but it's small enough that "beyond reasonable doubt" means something precise here.

The Xena Project blog took on exactly this worry in [*…and the proof is checked in Lean*](https://xenaproject.wordpress.com/2026/11/12/and-the-proof-is-checked-in-lean/), posted on 12 November. It doesn't pretend kernel bugs can't happen:

> This can happen. There was a soundness bug in Lean's kernel which was fixed four hours after it was discovered, and a subsequent review of Lean's code base by OpenAI found several more, all of which were fixed before they were announced publicly. There may in theory be more soundness bugs in Lean's kernel. However, this does not matter.

The key move is *"this does not matter"*. The post doesn't claim the official kernel is bug-free. Its reason is the one from the previous section: Lean now has many kernels, written by different people in different languages, and a malicious proof would have to exploit undiscovered bugs in all of them at once. So what you have to trust is not that one implementation is perfect, but that **many independent implementations won't all fail on the same proof in the same way**. That is a much weaker assumption, and a much easier one to believe.

3Blue1Brown just put out a video, [*"What if AI solved the Riemann Hypothesis?"*](https://youtu.be/zqzk43bSdDI), made from interviews recorded over the past year, before any of this happened. I highly recommend watching it to hear what the community has to say about AI proving things with Lean. In an interview from May 2026, Leo de Moura described almost exactly the procedure that was followed this week:

> We have a curated statement of the Riemann hypothesis. So the first thing we would do is run the comparator and check … there is no exploits. Suppose the comparator says, yes, this is a proof for the Riemann hypothesis, and I checked it with two, three different kernels. They all agree. This is amazing.

That is more or less what happened on 21 November. Which brings us to the comparator.

## Did we actually prove RH?

Back to the pedantic worry from the start. The kernel guarantees that the *proof* proves the *statement*. It does not guarantee that the statement says what you think it says.

The Xena post breaks "it's checked in Lean" into four separate tests:

1. Does the Lean statement match the informal statement?
2. Does the code compile?
3. Does the code prove *that* statement?
4. Does it exploit a kernel bug?

Only (4) is about the kernel, and we just covered it. The others are where things can quietly go wrong.

**Statements can lie.** Write Fermat's Last Theorem naively over `ℕ`, as "for all `a b c n` with `n > 2`, `a^n + b^n ≠ c^n`", and it's false, because in Lean `ℕ` contains 0 and `0^3 + 0^3 = 0^3`. (The Xena post uses this same example.) Lean will happily accept a proof of the *negation*.

The workshop materials have a "spot the bug" exercise with more of these (we didn't get to it on the day). All of the following are true in Lean:

```lean
example : (5 : ℕ) - 7 + 7 = 7 := by norm_num -- subtraction on ℕ truncates: 5 - 7 = 0
example : (1 : ℝ) / 0 = 0 := by simp         -- division by zero is defined to be 0
```

These aren't bugs. They are conventions that make the library easier to work with. But they mean you have to *read* the statement.

**Proofs can cheat.** How do you know a 17-million-line proof didn't quietly redefine `^` somewhere, or add an axiom that makes the theorem true? That is what **`lake comparator`** is for. It checks that the theorem stated in a trusted file and the theorem proved through untrusted imports are *identical*, and that only the standard axioms are used. On a smaller scale, `#print axioms` shows which axioms any proof relies on. A clean answer is short: according to [Andrew Sutherland's checks](https://math.mit.edu/~drew/grh/), the main theorems of the RH proof depend only on Lean's three standard axioms, `propext`, `Classical.choice` and `Quot.sound`.

**How RH did on the four tests.** From the community statement:

1. The statement is **Mathlib's own `RiemannHypothesis`**, a definition that was already in the library and had been vetted by experts in Lean and number theory.
2. Dozens of people compiled the library and reproduced the same proof export.
3. `comparator` and `con-leche --compare` both confirm that the theorem proved is exactly Mathlib's statement.
4. The official kernel, con-leche, con-ron and several third-party checkers all accept the proof. They ran on an inert export, out of reach of any malicious code in the sources.

You don't have to take anyone's word for this. Andrew Sutherland (MIT) has posted [every check he ran](https://math.mit.edu/~drew/grh/), with the logs. A sandboxed build reproduced the published export byte for byte. nanoda, lean4lean, con-leche and con-ron all accept the proof, and `con-leche --compare` and `con-ron --compare` confirm that the theorem proved is exactly the one stated.

So yes: this is a proof of *the* Riemann Hypothesis, not of a lookalike.

### What Lean does not tell you

There are two things all of this does *not* give you.

**It says nothing about any written argument.** The Xena post again: even if the Lean code passes every test, *"this does not mean that any corresponding pdf also contains a correct proof!"*

**It doesn't tell you why RH is true.** The community statement ends by saying it looks forward to *"all the activity in the mathematical community that will help explain the new ideas in this proof."* Right now, nobody understands those ideas.

What it means to *prove* something is a very philosophical question, and I'm not going to settle it in this post. Lean does guarantee the proof of RH. But did we, as a society, actually gain anything from it?

My personal answer is **yes and no**.

*Yes*, because in a few years (or months) AI will get good at untangling this Lean mess into a proof humans can understand, and that could open up entirely new areas of mathematics.

*No*, because at the end of the day, why are we doing maths? If a magical oracle answered every theorem with a yes or a no but never explained why, we wouldn't be getting anything. Why would we even do maths at that point?

## Why now? Lean is not a black box

I think this is the best time to get comfortable with Lean, and not because AI will make Lean disappear behind a chat box. It's the opposite.

AI is going to keep producing Lean, more and more of it, faster than anyone can read. As that happens, the work that matters shifts to the parts that a human has to own:

- **reading the statement**, and knowing what the definitions actually mean (is your `ℕ` allowed to be 0?);
- **knowing which statements to trust**: Mathlib's curated definitions are where this starts, and RH was literally one of them;
- **running the checks**: `#print axioms`, comparator, `lake check --paranoid`.

None of these is a programming skill. They are mathematician's skills: reading definitions carefully and being suspicious of statements. You don't need to write 17 million lines of Lean. You need to be able to read the twenty that state the theorem.

Lean isn't a magical black box that only AIs talk to. It's a language you can read, and its guarantees come from tools you can run yourself.

One practical warning, from the Xena post: *"do not run random Lean code on your local machine unless you sandbox it first."* Mathlib is human-curated. A random AI-generated repository isn't.

## Where to go next

If the workshop, or this post, got you curious:

- [Natural Number Game](https://adam.math.hhu.de/#/g/leanprover-community/nng4): build ℕ from scratch, as a game. This is the part we didn't get to.
- [*Mathematics in Lean*](https://leanprover-community.github.io/mathematics_in_lean/): real maths with Mathlib.
- [*Theorem Proving in Lean 4*](https://lean-lang.org/theorem_proving_in_lean4/): how Lean works underneath, including Curry–Howard properly.
- [*Functional Programming in Lean*](https://lean-lang.org/functional_programming_in_lean/): Lean as a programming language.
- [Lean Zulip](https://leanprover.zulipchat.com): the community, which is very friendly to beginners.
- [Workshop materials](https://github.com/Methaphur/NISER-Lean-Workshop), all in one repository:
  - [Slides](https://github.com/Methaphur/NISER-Lean-Workshop/blob/main/Slides/Lean-Workshop-S3.pdf) for the hands-on session.
  - [Tactic cheat sheet](https://github.com/Methaphur/NISER-Lean-Workshop/blob/main/Slides/Tactic-Cheatsheet.pdf): every core tactic on one page, with the goal before and after.
  - Follow-along files: [plain Lean](https://github.com/Methaphur/NISER-Lean-Workshop/blob/main/Workshop/S3_follow.lean) and [with Mathlib](https://github.com/Methaphur/NISER-Lean-Workshop/blob/main/Workshop/S3_follow_mathlib.lean). Replace each `sorry` with a proof.
  - Build the natural numbers yourself: [exercises](https://github.com/Methaphur/NISER-Lean-Workshop/blob/main/Workshop/MyNat_ex.lean) and [solutions](https://github.com/Methaphur/NISER-Lean-Workshop/blob/main/Workshop/MyNat_sol.lean).


## Thanks

Thanks to Anamay and Aneesh for co-hosting, and for the two sessions that made mine possible. Thanks to everyone who came and spent a Saturday on proofs.

## Sources

- Lean and Mathlib communities, [Lean verification of the Riemann Hypothesis](https://leanprover-community.github.io/blog/posts/riemann-hypothesis/) (21 Nov 2026)
- Levent Alpöge, [announcement](https://alpo.ge/remark.pdf) (20 Nov 2026)
- Andrew Sutherland, [Verification files for the Lean proof of the Riemann Hypothesis](https://math.mit.edu/~drew/grh/) (last updated 22 Nov 2026)
- Leo de Moura, [Postmortem for Kernel Soundness Bug #14576](https://leodemoura.github.io/blog/2026-8-1-postmortem-for-kernel-soundness-bug-14576/) (1 Aug 2026)
- Leo de Moura, [Lean Now Ships with Verified Kernels](https://leodemoura.github.io/blog/2026-10-15-lean-now-ships-with-verified-kernels/) (15 Oct 2026)
- Xena Project, […and the proof is checked in Lean](https://xenaproject.wordpress.com/2026/11/12/and-the-proof-is-checked-in-lean/) (12 Nov 2026)
- 3Blue1Brown, [Since last year, we asked: "What if AI solved the Riemann Hypothesis?"](https://youtu.be/zqzk43bSdDI) (22 Nov 2026)
- Mathlib, [`RiemannHypothesis`](https://leanprover-community.github.io/mathlib4_docs/Mathlib/NumberTheory/LSeries/RiemannZeta.html#RiemannHypothesis)
