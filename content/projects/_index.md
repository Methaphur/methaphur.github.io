---
title: "Projects"
description: "Research code, teaching material, interactive explainers, and community software by Harisankar B, from verified replicated data types in Lean 4 to student event platforms."
layout: "content-only-list"
showTableOfContents: true
---

## Research code

{{< icon "check" >}} **Tombstone-free sequence MRDT in Lean 4**  
Contributed a tombstone-free RGA and its successor **EmbedRGA** to [Sal](https://github.com/fplaunchpad/sal), the Lean 4 verification framework for replicated data types developed at FP Launchpad, IIT Madras. The tombstone-free variant was proved RA-linearizable and then found to reorder text on delete regardless; EmbedRGA repairs this with immutable coordinates. [Write-up](/blogs/replicated-growable-array) · [Slides](/talks/RGA-Talk.pdf)

---

## Teaching material

{{< icon "graduation-cap" >}} **NISER Lean Workshop**  
Materials for an introductory Lean workshop at NISER, aimed at mathematicians new to proof assistants. Includes slides for the hands-on session, a one-page tactic cheat sheet showing the goal before and after each tactic, and follow-along exercise files where each `sorry` is left for you to fill in. [GitHub Repo](https://github.com/Methaphur/NISER-Lean-Workshop) · [Write-up](/blogs/lean-workshop-niser)

{{< icon "file-lines" >}} **Topology Assignment Solutions (2026)**  
A class-wide LaTeX compilation of worked solutions to the Topology problem sets, with solutions contributed by over twenty students. I set up the shared preamble, build, and per-problem file layout, and assemble each contribution into a single credited PDF. [GitHub Repo](https://github.com/Methaphur/Topology-Assignment-Solutions-2026)

{{< icon "code" >}} **CS142 Lab Solutions**  
My solutions to the lab assignments for CS142, the data structures and algorithms course at NISER, written with explanatory comments and organised by week. They cover recursion and sorting, divide and conquer, backtracking (subset sum, N-Queens, Sudoku), linked lists, binary trees and hashing. [GitHub Repo](https://github.com/Methaphur/CS142-NISER)

### Outreach demos

Interactive demos built for the NISER Open Day, where they were presented to visiting high-school students.

{{< icon "search" >}} **Path-Finding Visualisation** · *Open Day 2024*  
An interactive grid on which you draw walls or generate a random maze, then watch Dijkstra's algorithm and A\* search for a route, with or without diagonal moves. Used to show students how an algorithm turns an idea into something that works. [GitHub Repo](https://github.com/Methaphur/Path-Finding-Visualization)

{{< icon "eye" >}} **Pose Detection** · *Open Day 2025*  
Live computer-vision demos built on MediaPipe and YOLO: full-body and multi-person pose tracking from a webcam, and a hand-tracking "air paint" canvas. [GitHub Repo](https://github.com/Methaphur/Pose-Detection)

---

## Explainers and visualisations

{{< icon "lightbulb" >}} **How QR Codes Work**  
An interactive, animated walkthrough of a QR code from end to end: how text becomes bits, how the bytes snake into the grid, how Reed–Solomon codes over a field of 256 elements let a damaged code heal itself, and how a scanner finds and un-skews the symbol at any angle. [View Website](https://methaphur.github.io/How-do-QR-Codes-Work/) · [GitHub Repo](https://github.com/Methaphur/How-do-QR-Codes-Work)

{{< icon "wand-magic-sparkles" >}} **Catacaustic Curve Explorer**  
Shine light at a mirror and watch the caustic form. Choose a reflective curve (or type in your own *y = f(x)*), move the light source or drag a point source, and see the envelope of reflected rays trace out curves such as the nephroid of a circular mirror. [View Website](https://methaphur.github.io/catacaustic-curves/) · [GitHub Repo](https://github.com/Methaphur/catacaustic-curves)

---

## Community and student projects

{{< icon "code" >}} **QR Coupon for Event Management**  
A lightweight website for sending QR codes for event participation through email with Google Apps Script, plus a static verification page for organizers. [GitHub Repo](https://github.com/Methaphur/QR-Coupon-Scanner)

{{< icon "star" >}} **Umang '25 Fest Website**  
The official site for Umang, the annual intra-college fest at NISER, held 23–26 October 2025. Alongside event listings and a schedule organised by club, it ran a live results board that sorted events into completed, ongoing, and upcoming, refreshing itself every five minutes so winners appeared as each event wrapped up rather than after the fest. [View Website](https://methaphur.github.io/Umang/)

{{< icon "list-check" >}} **Problem of the Week Platform**  
A website for weekly competitive programming problems with automated leaderboard management for community coding engagement. [View Project](https://sdgniser.github.io/problem-of-the-week) · [GitHub Repo](https://github.com/Methaphur/problem-of-the-week)

{{< icon "worktree" >}} **Switcheroo Website**  
A Flask-based web server for a coding event where teams collaborate by switching systems and solving problems together, with integrated timing and scoring. [GitHub Repo](https://github.com/Methaphur/switcheroo)

{{< icon "mug-hot" >}} **NISER Mess and Canteen Menus**  
Websites that present the weekly NISER mess menu and the campus canteen menus, with staff uploads through Google Forms and automatic display for students. On the canteen menu I added image support, the night canteen, and the move back to a Google Forms backend. [Mess Menu](https://methaphur.github.io/messmenu/) · [Canteen Menu](https://methaphur.github.io/canteenmenu/)

{{< icon "bell" >}} **Event Management**  
An event scheduling system concept that integrates with Google Calendar to reduce scheduling conflicts and support a public NISER club calendar. [Project Link](https://sdgniser.github.io/event-management)

{{< icon "scale-balanced" >}} **Timetable Generator**  
A work-in-progress system for minimizing course conflicts using constraint satisfaction, graph coloring, and scheduling algorithms.

{{< icon "code" >}} **Competition Website**  
A work-in-progress coding competition platform using Django and React for NISER students, including leaderboards, point tracking, and team participation. [GitHub Repo](https://github.com/sugar-syrup/NiserCodeLab)

---

## Smaller things

- [**ROSALIND**](https://github.com/Methaphur/ROSALIND): solutions to bioinformatics problems, with each algorithm implemented from scratch rather than taken from a library.
- [**Rubik's Cube**](https://github.com/Methaphur/Rubik-s-Cube): a model of the cube as a permutation of its 54 facelets, with each face turn written as a product of 4-cycles.
