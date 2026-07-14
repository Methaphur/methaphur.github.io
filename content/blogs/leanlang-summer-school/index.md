---
title: "LLM Proposes, Lean Disposes: A Week at the LeanLang Summer School"
date: 2026-07-14
description: "Notes from the LeanLang Summer School at IISc Bangalore, on Lean"
summary: "In July 2026 I spent a week at IISc Bangalore for the LeanLang Summer School, run by Emergence. Here is how I got there, what Lean actually is, why formally verified autonomy suddenly matters, and a tour of the week."
tags: ["lean", "formal verification", "theorem proving", "emergence", "IISc", "distributed systems" ,"agentic systems", "summer school", "LLM"]
categories: ["Notes"]
showHero: true
heroStyle: "big"
featureimagecaption: "The main building at the Indian Institute of Science, Bangalore."
alt: "The historic main building at the Indian Institute of Science, Bangalore"
---

{{< katex >}}

<!-- In the second week of July 2026, I found myself at the Indian Institute of Science in Bangalore for five days of the [LeanLang Summer School](https://www.emergence.ai/), run by [Emergence](https://www.emergence.ai/). It was fully funded, down to the accommodation at a rather nice hotel (JP Cosmos, Fortune Select). -->

I spent five days at the [LeanLang Summer School](https://www.emergence.ai/) at the Indian Institute of Science in Bangalore, run by [Emergence](https://www.emergence.ai/). The week was a mix of lab sessions and talks, all focused on Lean, formal verification, and the emerging ecosystem around it.

<!-- That sounds like an odd way to spend a summer, so let me try to explain why it is one of the more exciting places to be right now. The short version is a question that has quietly become urgent: what does it mean to actually *trust* a piece of software — and now that we are handing real decisions to AI agents, what would it mean to trust *them*? -->

This is one of the most exciting places to be right now, because it is grappling with a question that has quietly become urgent: what does it mean to actually *trust* a piece of software — and now that we are handing real decisions to AI agents, what would it mean to trust *them*?


## How I got here

I am an Int. MSc student at [NISER](https://www.niser.ac.in/), majoring in mathematics with a minor in computer science. A maths degree trains you to care about precise definitions and airtight arguments, and at some point that taste for rigour collided with computers.

My first real encounter with [Lean](https://lean-lang.org/) came during a summer internship at [IMSc](https://www.imsc.res.in/) under [Dr. Meena Mahajan](https://www.imsc.res.in/~meena/), where I studied some introductory logic and automated reasoning. That was the spark: the idea that you could take a mathematical argument and express it precisely enough that a machine could check it, no hand-wavy arguments.

These days I am continuing down that road as a summer intern in [FP Launchpad](https://fplaunchpad.org) under [Prof. KC Sivaramakrishnan](https://kcsrk.info/) at [IIT Madras](https://www.iitm.ac.in/), working on formal verification in distributed systems — specifically, using Lean for pragmatic verification of replicated data types *(blog post soon™)*  The work is done in Sal (PaPoC 2026), a framework for verifying replicated data types.  Sal’s multi-modal tactic stack is directly inspired by Loom, a multi-modal proof-orchestration layer built by [Ilya Sergey's](https://ilyasergey.net/) group at [VERSE Lab](https://verse-lab.github.io/). So when I saw that Sergey himself would be teaching, applying was not a difficult decision.

## Why this matters now: the promise of verified autonomy

It is worth pausing on *why* a company would fund a week of theorem proving in the first place.

{{< figure src="emergence-merch.jpeg" alt="An Emergence tote bag, ID card, and summer-school merchandise" caption="Fully funded, down to the tote bag and merch Emergence sent us home with." >}}

Emergence frames it bluntly: "the defining challenge of this moment in computing is not capability, it's control." Models are already powerful enough to plan, reason, and act at enterprise speed; what is missing, they argue, is "the critical infrastructure to ensure they act within verified bounds." Their [research lab in India](https://east.emergence.ai/) puts the thesis on a bumper sticker — *"solve scalable correctness, unlock scalable autonomy"* — and it rests on a bet that two things have quietly become true at the same time. First, "proof assistants like Lean have reached production maturity." Second, "LLMs can now translate natural language to formal specifications."

Put those together and you get a genuinely appealing architecture. A language model is a wonderful, creative, deeply *untrustworthy* proposer. A proof checker is a humourless, uncreative, utterly *trustworthy* judge. So you let the model do what it is good at — propose a plan, a patch, a move, a config change — and then you refuse to act on that proposal until it comes with a machine-checkable proof that it is safe. Thus defining the title of this post **The LLM proposes, and Lean disposes.**


## So what is Lean, anyway?

If you have not met it before: [Lean](https://lean-lang.org/) is two things at once. It is a functional programming language, and it is an interactive theorem prover. The trick that lets it be both is the [Curry–Howard correspondence](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence): propositions are types, and proofs are programs. Proving a theorem and writing a program that type-checks turn out to be the *same activity*. Once you accept that, a proof assistant is just a very pedantic compiler.

In practice, you rarely write proof terms by hand. You write **tactics** — small commands that transform the goal in front of you. Our morning lab sessions, led by [Siddhartha Gadgil](https://math.iisc.ac.in/~gadgil/) (a mathematics faculty member at IISc who has been an energetic advocate for Lean and a key collaborator with Emergence), were mostly spent walking through his teaching repository, [LeanLangur](https://github.com/siddhartha-gadgil/LeanLangur), and getting comfortable with the staples: `rfl`, `simp`, `rw`, `induction`, `cases`, `exact`, `decide`, and the heavier automation of `grind` and `aesop`. A recurring theme was that this toolbox is not fixed: Lean lets you define *your own* tactics with macros, extending the language as you go.

Underneath the tactics sit the ideas that make dependent type theory tick, and Prof. Gadgil's labs covered them in turn: polymorphism, structures, inductive types and type families, type-class inference, and **monads**. Monads show up as a single abstraction unifying wildly different things: lists and other collections, mutable `State`, `Option`/`Except` for failure, side effects, and long-running computation. The common shape is a type constructor `m` with a `pure : α → m α`, a functorial `map`, and the star of the show, *bind* (flat-map). Bind is what powers Lean's `do` notation — and Lean pushes that notation impressively far, letting you write `let mut`, `while` loops, and early returns that all desugar back down to this functional core. It is a lovely reminder that the friendly imperative syntax is monadic underneath.

If this has made you curious enough to try Lean yourself, there is no shortage of on-ramps:

- The [Lean community's *Learn* page](https://leanprover-community.github.io/learn) is the best starting hub — it lays out different routes depending on where you are coming from, whether that is mathematical theorem proving, computer science, or everyday programming.
- My personal favourite way to actually *start* is the [Natural Number Game](https://adam.math.hhu.de/#/g/hhu-adam/NNG4): you build the natural numbers up from the axioms and prove real theorems within minutes. The same group has sibling games for set theory, real analysis, and more.
- For something with a programming and CS flavour, the [*Hitchhiker's Guide to Logical Verification*](https://raw.githubusercontent.com/blanchette/logical_verification_2023/main/hitchhikers_guide.pdf) is an excellent book — and it comes with a companion GitHub repository of exercises you can work through as you read.


## The week itself

The rhythm was consistent: morning were mostly hands-on lab sessions with Prof. Gadgil where he walked us through the `LeanLangur` repository, and afternoons were talks, Q&A, and more lab sessions. The talks were a mix of theory and practice, with a strong emphasis on how Lean can be used to verify real-world systems.


{{< figure src="maths-building.jpg" alt="The mathematics department building at IISc Bangalore" caption="The Department of Mathematics at IISc, where the labs and talks were held." >}}

**Day 1** we had [Ra D'Souza](https://www.csa.iisc.ac.in/~deepakd/) on the foundations of **Hoare logic** — the original theory of what it means for a program to be correct. He laid out the intellectual lineage cleanly: inductive annotations come from Floyd, the proof system from Hoare, and the weakest-precondition calculus from Dijkstra. The central object is the Hoare triple \(\{P\}\,S\,\{Q\}\): if statement \(S\) starts in a state satisfying precondition \(P\), then it will not crash, and if it terminates it lands in a state satisfying postcondition \(Q\). He closed on Cook's 1974 completeness result — Hoare logic is complete provided the assertion language can express the weakest precondition for any program and postcondition ([slides here](https://www.csa.iisc.ac.in/~deepakd/fmse-2018/VCC/pav-floyd-hoare-2017.pdf)). 

Then **Ilya Sergey** picked up almost exactly where D'Souza left off, showing how that classical theory becomes something you can actually *run* in Lean through the **[Velvet](https://proofsandintuitions.net/2026/01/21/multi-modal-verification-velvet/)** framework from [VERSE Lab](https://verse-lab.github.io/). The clever move is that Velvet is not a standalone tool but a verifier *embedded* in Lean 4: unlike Dafny or Verus, which check programs in their own closed worlds, Velvet lets you verify genuinely imperative code — loops, mutable state, exceptions — from inside Lean's full mathematical universe. Under the hood it is Floyd-Hoare logic made mechanical: you annotate a program with a postcondition and loop invariants (which, as Sergey put it, play much the same role as induction hypotheses), and Velvet generates the verification conditions for you. By Rice's theorem there is no general way to infer those invariants automatically, so supplying them is the price of admission.

What makes it *multi-modal* is how the resulting proof obligations get discharged. The routine ones are handed to automated back-ends — SMT solvers like Z3 and cvc5, or Lean's own `grind` and `aesop`. The genuinely hard, mathematical ones are left to interactive Lean proof scripts.

He also pointed us to **[WybeCoder](https://facebookresearch.github.io/wybecoder/)**, a project from Meta's FAIR that pushes this to its extreme: an LLM generates the whole thing — the Velvet code, its loop invariants, and the proofs — refining against compiler feedback until Lean accepts the result, a loop its authors call **prove-as-you-generate**. It is the post's thesis in its purest form — the model proposes an entire program, and Lean decides whether to trust it.

Sergey framed the larger goal as a future where "rigorous program verification becomes accessible to a much wider audience of non-experts."

**Day 2** was a highlight for me personally, because the first afternoon talk was by my own summer advisor, [**KC Sivaramakrishnan**](https://kcsrk.info/), on the Neem and Sal work — pragmatic verification of replicated data types. I will be writing a blog post detailing the work.

Sergey then returned for a talk on **multi-modal verification of distributed systems** with [Veil](https://veil.dev/). He set it up with the classics — Lamport's *Time, Clocks, and the Ordering of Events*, the fault-tolerant lineage of Viewstamped Replication, Paxos and Raft, and the adversarial world of the Byzantine Generals Problem — plus a sobering pointer to a [running list of bugs found in *published* distributed protocols](https://github.com/dranov/protocol-bugs-list). The properties you care about are **safety** ("a bad thing will not happen") and **liveness** ("a good thing will eventually happen"), and it is the second kind, especially under Byzantine faults, where conventional testing and model checking start to run out of road.

The heart of the talk was a single chart: formalisation *effort* on one axis, *correctness* — how many bugs you would actually catch — on the other. Lightweight tools like TLA+ give fast feedback (model-check the protocol and get back a tick or a concrete counterexample) but plateau in what they can establish; the heavyweight interactive frameworks (IronFleet, Verdi, Velisarios, Igloo, LiDO) reach the top of the correctness axis at the cost of enormous manual effort and no fast feedback; and SMT-based tools like Ivy and mypyvy automate proofs only by staying inside a decidable first-order fragment. Sergey's "we want to be *here*" pointed at the corner everyone else misses — undecidable, higher-order properties *with* fast feedback — and **Veil** is the attempt to live there. It is a shallowly-embedded DSL in Lean that models a protocol as a **transition system** (states, transitions, and an *inductive invariant* that every step must preserve) and then attacks it from every mode at once: explicit-state and symbolic **model checking** for fast feedback and counterexamples, **external SMT solvers** to discharge the routine first-order goals, and **out-of-the-box interactive Lean proofs** for the genuinely hard, higher-order ones. On his feature-comparison slide, Veil was the only tool with a green tick in every column. He used Ring Leader Election as the running example; his group writes it all up beautifully at [proofsandintuitions.net](https://proofsandintuitions.net/).

**Day 3** brought KV Raghavan on **abstract interpretation** — the art of soundly approximating what a program does — alongside more lab.

**Days 4 and 5** We had project demos from the LeanLang Hackathon crowd and from Emergence, and nearly every one was some flavour of *AI proposes, Lean checks*:

- **FlowGuard** (Dev Soni), memorably subtitled *"I will, unfortunately, enter your house."* The point: the composition of agents need not be safe even when every individual capability was tested and works fine — it is the *union* that bites you. He drew on information flow in multi-agent systems, prompt injection (a term coined by [Simon Willison](https://simonwillison.net/)), the FIDES taint-tracking approach with a quarantined LLM, and AWS Cedar for policy.
- **"Who sacked my cluster!"** (Sachin Singh, a senior engineer at DigitalOcean, and the hackathon winner). This was the cleanest verified-autonomy story of the week. Managing Kubernetes clusters is painful, and an SRE *agent* to do it is non-deterministic and cannot be trusted to be safe. So he put a **Lean-checked admission gate** in front of it: the agent's proposed action is only executed if it comes with a machine-checkable proof, and when the proof fails, the structured rejection reason is fed back as an RL signal. Control theory meets proof theory.
- **Chess Provers in Lean** (Saachi, MARS team): feed in a position in FEN notation, let an LLM propose moves, and verify their legality in Lean — success only when the validated sequence ends in checkmate.
- **LeanDatabase** (Ajay Kumar Nair): a neat trick for trusting LLM-generated SQL. When you ask a model to turn a natural-language question into a SQL query, you have no easy way to know the query it hands back is correct. So instead of asking once, LeanDatabase asks several times and uses Lean to check whether the resulting queries are all *semantically equivalent* — Lean metaprogramming elaborates each SQL query into a Lean term capturing its meaning, and then the queries are proven to denote the same thing. The insight is probabilistic: a model might well produce a wrong query, but it is very unlikely to produce the *same* wrong query across several independent tries, so provable agreement between the samples is strong evidence the translation is right.
- A cluster of others in the same spirit: **PastaLean** (Anirudh Gupta), transpiling Python through its AST into Lean so you can "write in Python and get the guarantees of a proof assistant"; **Bulletproof Templates**, a templating approach that parses templates into a context-preserving tree to make XSS impossible by construction; and **VerifyTheVector**, checking that an LLM's claim actually matches the SVG it generated.

## What I'm taking home

More than any single talk, what I am taking home is the people. Over five days of coffee breaks and long lunches I got to talk with students, PhD students, postdocs, faculty, and folks from industry — each carrying a different perspective and a different problem they were excited about. Those conversations, as much as the lectures, made me realise just how genuinely I am invested in this field.

So a big thank you to Emergence for organising such a wonderful event, and to Siddhartha Gadgil, KC Sivaramakrishnan, Ilya Sergey, Deepak D'Souza, KV Raghavan, and everyone working behind the scenes for putting together such a generous and genuinely inspiring week. I came for the Lean; I left with a much clearer sense of why I want to keep doing this.


{{< figure src="certificate.jpg" alt="My LeanLang Summer School certificate of completion alongside the ID card" caption="_Proof_ of completion" >}}
