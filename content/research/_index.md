---
title: "Research"
description: "Research interests, internships, and publications of Harisankar B, covering formal verification, automated reasoning, and theoretical computer science."
layout: "content-only-list"
showTableOfContents: true
---

My research interests lie at the intersection of **automated reasoning**, **formal methods**, **theoretical computer science**, and **machine learning**. I am particularly interested in developing systems that can reason about, verify, and synthesize correct software and mathematical artifacts. My current work focuses on **automated theorem proving**, proof assistants such as [Lean](https://leanprover-community.github.io/), and the **formal verification of distributed systems**. More broadly, I am fascinated by the **foundations of computation**, **logic**, **programming languages**, and the ways in which formal reasoning techniques can be combined with modern machine learning methods to build more reliable and trustworthy intelligent systems.

---

## Publications

**Harisankar Binod**, T. Shithij, A. Tichy, and S. Mishra. *Artificial Intelligence in Oral Health*. In **Artificial Intelligence for Oral Health Care: Applications and Future Prospects**, eds. F. Schwendicke, P.K. Chaudhari, K. Dhingra, S.E. Uribe, M. Hamdan. Springer Nature Switzerland, Cham, 2025, pp. 1-21. [https://doi.org/10.1007/978-3-031-84047-0_1](https://doi.org/10.1007/978-3-031-84047-0_1)

---

## Talks

**Convergence is Not Enough: Verifying a Tombstone-Free Sequence MRDT in Lean 4**  
*July 2026 · FP Launchpad, IIT Madras*

Closing presentation for my internship under [Dr. KC Sivaramakrishnan](https://kcsrk.info/). The talk builds from collaborative text editing and why replication makes it hard, through CRDTs and RGA, to **RA-linearizability** as a statement of what "correct" should mean for a sequence — convergence alone is not enough. It then presents a tombstone-free RGA mechanized in Lean 4 with the [Sal](https://kcsrk.info/papers/sal_jan26.pdf) framework: proved RA-linearizable, and then shown to reorder text on delete regardless. It closes with EmbedRGA, which repairs this using immutable coordinates.

[Slides](/talks/RGA-Talk.pdf) · [Write-up](/blogs/replicated-growable-array)

**Transformers Learn Shortcuts to Automata**  
*April 2025 · Advanced Machine Learning course, NISER · with Yash Chauhan*

A talk on [Liu et al. (2022)](https://arxiv.org/abs/2210.10749), which asks how shallow, non-recurrent Transformers manage computations that appear to require sequential processing. Rather than iterating a transition function step by step, Transformers learn *shortcut solutions* that compress the whole sequence into a few parallel layers. The talk covers the construction behind this — Krohn-Rhodes decomposition and results from circuit complexity — along with why such shortcuts are brittle and generalize poorly to longer inputs.

[Slides](/talks/transformers-shortcuts-to-automata-slides.pdf) · [Report](/talks/transformers-shortcuts-to-automata-report.pdf) · [Paper](https://arxiv.org/abs/2210.10749)

---

## Research and Internships

### Research Intern at FP Launchpad, IIT Madras
*May–July 2026*

I was a Research Intern at [FP Launchpad](https://fplaunchpad.org/) at the [Indian Institute of Technology Madras](https://www.iitm.ac.in/), working under the guidance of [Dr. KC Sivaramakrishnan](https://kcsrk.info/). My research focused on **formal verification** of replicated data types, using *[Lean](https://leanprover-community.github.io/)* to verify correctness of **distributed systems**. ([Write-up](/blogs/replicated-growable-array))

### Summer Research Program, IMSc Chennai
*Summer 2025*

In the summer of 2025, I participated in the [Summer Research Program](https://www.imsc.res.in/summer_research_programme) at the [Institute of Mathematical Sciences](https://www.imsc.res.in/) (IMSc), Chennai, under the guidance of [Dr. Meena Mahajan](https://www.imsc.res.in/~meena/), Professor in the Theoretical Computer Science group. My work focused on **automated reasoning**, where I explored topics in **formal logic**, experimented with **Lean**, and studied automated theorem proving.

### Winter Internship, IISc Bangalore
*Winter 2023*

In the winter of 2023, I did a small reading project under the guidance of [Dr. L Sunil Chandra](https://www.csa.iisc.ac.in/~sunil/students.html) at the [Indian Institute of Science](https://www.iisc.ac.in/) (IISc), Bangalore. The project involved reading and summarizing a research paper on **Graph Theory**.

### AI in Oral Health, NISER
*Summer 2023*

In the summer of 2023, I completed an internship under the guidance of [Dr. Subhankar Mishra](https://niser.ac.in/~smishra/) on a research project related to **Artificial Intelligence in Oral Health**. This work contributed to a [book chapter](https://doi.org/10.1007/978-3-031-84047-0_1) on applications of AI in diagnosis, treatment, and patient care in dentistry.
