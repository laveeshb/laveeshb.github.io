---
layout: post
title: "kernel-internals.org: The Why Behind Linux Kernel Design"
date: 2026-07-26
order: 8
categories: [linux]
tags: [linux-kernel, documentation, open-source]
excerpt: "A project I've been building — a reference for the design rationale behind the Linux kernel, not just what it does but why it's built that way."
---

For a while now I've been building <a href="https://kernel-internals.org" target="_blank">kernel-internals.org</a> — a growing reference for the *design rationale* behind the Linux kernel. Not "here's the struct and here's the function," but "here's the problem, here's what they tried first, and here's why the current design won — and what it costs."

This post is about why the site exists and where it stands today.

## The gap

If you want to understand the Linux kernel, the material is oddly polarized. <a href="https://docs.kernel.org" target="_blank">docs.kernel.org</a> is authoritative but tells you *what* and *how* — it's reference documentation, not a narrative. The classic books ("Understanding the Linux Kernel," Robert Love's) are excellent and hopelessly out of date — most stop around the 2.6 era, before folios, EEVDF, io_uring, or modern reclaim even existed. <a href="https://lwn.net" target="_blank">LWN</a> is the gold standard for the *why*, but it's news-shaped — tied to a specific patch series or conference — and not organized as an evergreen, browsable reference.

So the thing I always wanted didn't really exist: a current, well-organized place that explains the *reasoning* — the trade-offs, the rejected alternatives, and the mailing-list arguments that produced the design we have. That's the gap kernel-internals.org tries to fill. It complements docs.kernel.org rather than competing with it.

## Where it stands today

The site is a static <a href="https://squidfunk.github.io/mkdocs-material/" target="_blank">MkDocs</a> site, and it's grown to roughly 375 documents across 28 subsystems. A few things I care about:

- **Memory management is the deepest section** — around 90 pages covering the page allocator, reclaim and MGLRU, DAMON, CXL memory tiering, folios, huge pages, the copy-on-write machinery, `get_user_pages`, and the kernel's virtual address-space layout. It's the area readers gravitate to, so it gets the most attention.
- **Broad coverage elsewhere** — the scheduler (EEVDF, deadline, energy-aware scheduling), networking (XDP, AF_XDP, netfilter), the architecture layer (x86-64 and arm64 page tables, syscall entry, Spectre/Meltdown mitigations), io_uring, locking and RCU, VFS, filesystems, cgroups, tracing and BPF, and more.
- **Everything aims to cite a primary source** — a specific commit, an LWN article, an lore.kernel.org thread. "Trust me" isn't good enough for kernel internals, and a claim you can't trace is a claim I don't want on the site.
- **The cross-references stay honest** — there's CI that fails the build on any broken internal link or anchor, so as the site grows the ~2,000 links between pages don't rot.

## How I think about a page

The test for a good page isn't whether it lists the right fields — it's whether it answers *why*. Take `get_user_pages`: the interesting story isn't the function signature, it's why the kernel split it into two nearly-identical APIs (`get_user_pages` vs `pin_user_pages`), why an ordinary reference count couldn't express "this page is being used for DMA," and why that ambiguity produced a decade of subtle bugs. The mechanism is a footnote; the reasoning is the point.

## Where it's going

It's open source, and there's plenty of surface left to cover — scheduler extensions (sched_ext), Rust in the kernel, the mainlining of PREEMPT_RT, and deeper coverage in the subsystems that are still thin. Corrections are especially welcome; getting the *why* wrong is worse than not covering it at all.

If any of this is your kind of thing, take a look at <a href="https://kernel-internals.org" target="_blank">kernel-internals.org</a> — and if you spot something wrong, tell me.
