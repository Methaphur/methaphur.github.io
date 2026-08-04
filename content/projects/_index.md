---
title: "Projects"
description: "Research code and community software projects by Harisankar B, from verified replicated data types in Lean 4 to student event platforms."
layout: "content-only-list"
showTableOfContents: true
---

## Research code

{{< icon "check" >}} **Tombstone-free sequence MRDT in Lean 4**  
Contributed a tombstone-free RGA and its successor **EmbedRGA** to [Sal](https://github.com/fplaunchpad/sal), the Lean 4 verification framework for replicated data types developed at FP Launchpad, IIT Madras. The tombstone-free variant was proved RA-linearizable and then found to reorder text on delete regardless; EmbedRGA repairs this with immutable coordinates. [Write-up](/blogs/replicated-growable-array) · [Slides](/talks/RGA-Talk.pdf)

---

## Community and student projects

{{< icon "code" >}} **QR Coupon for Event Management**  
A lightweight website for sending QR codes for event participation through email with Google Apps Script, plus a static verification page for organizers. [GitHub Repo](https://github.com/Methaphur/QR-Coupon-Scanner)

{{< icon "star" >}} **Umang '25 Fest Website**  
The official site for Umang, the annual intra-college fest at NISER, held 23–26 October 2025. Alongside event listings and a schedule organised by club, it ran a live results board that sorted events into completed, ongoing, and upcoming, refreshing itself every five minutes so winners appeared as each event wrapped up rather than after the fest. [View Website](https://methaphur.github.io/Umang/)

{{< icon "list-check" >}} **Problem of the Week Platform**  
A website for weekly competitive programming problems with automated leaderboard management for community coding engagement. [View Project](https://sdgniser.github.io/problem-of-the-week)

{{< icon "worktree" >}} **Switcheroo Website**  
A Flask-based web server for a coding event where teams collaborate by switching systems and solving problems together, with integrated timing and scoring. [GitHub Repo](https://github.com/Methaphur/switcheroo)

{{< icon "mug-hot" >}} **NISER Mess Menu Website**  
A website that presents the weekly NISER mess menu, with staff uploads through Google Forms and automatic display for students. [View Website](https://methaphur.github.io/messmenu/)

{{< icon "bell" >}} **Event Management**  
An event scheduling system concept that integrates with Google Calendar to reduce scheduling conflicts and support a public NISER club calendar. [Project Link](https://sdgniser.github.io/event-management)

{{< icon "scale-balanced" >}} **Timetable Generator**  
A work-in-progress system for minimizing course conflicts using constraint satisfaction, graph coloring, and scheduling algorithms.

{{< icon "code" >}} **Competition Website**  
A work-in-progress coding competition platform using Django and React for NISER students, including leaderboards, point tracking, and team participation. [GitHub Repo](https://github.com/sugar-syrup/NiserCodeLab)
