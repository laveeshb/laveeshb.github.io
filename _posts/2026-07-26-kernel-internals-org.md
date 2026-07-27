---
layout: post
title: "kernel-internals.org: Learning the Why Behind the Linux Kernel"
date: 2026-07-26
order: 8
categories: [linux]
tags: [linux-kernel, documentation, open-source, learning-in-public]
excerpt: "A side project where I try to make sense of why the Linux kernel is built the way it is. I'm not a kernel developer — this is learning in public, with everything traced back to primary sources."
---

Let me get the disclaimer out of the way first: I'm not a kernel developer. I work on cloud infrastructure, and the Linux kernel has always been the layer I *depend on* without really understanding. <a href="https://kernel-internals.org" target="_blank">kernel-internals.org</a> is my attempt to fix that — for myself, first — by writing down the *why* behind the kernel's design as I dig into it, and organizing it so the next curious person doesn't have to start from scratch.

So think of the site less as an authoritative reference and more as careful notes, taken in public. This post is about why I started it and where it stands.

## The gap I kept running into

When I started reading, the material felt oddly polarized. <a href="https://docs.kernel.org" target="_blank">docs.kernel.org</a> is authoritative but tells you *what* and *how* — it's reference documentation, not a narrative. The classic books ("Understanding the Linux Kernel," Robert Love's) are wonderful and badly dated — most stop around the 2.6 era, before folios, EEVDF, io_uring, or modern reclaim existed. <a href="https://lwn.net" target="_blank">LWN</a> is the gold standard for the *why*, but it's news-shaped — tied to a particular patch series — and not organized as something you can sit down and browse by subsystem.

What I kept wishing for was a current, organized place that explained the *reasoning*: the trade-offs, the alternatives that got rejected, the mailing-list arguments that produced the design we ended up with. I couldn't find it, so I started assembling it as I learned. It's meant to complement docs.kernel.org, not compete with it.

## Where it stands today

It's grown more than I expected — somewhere around 375 pages across 28 subsystems, built as a static <a href="https://squidfunk.github.io/mkdocs-material/" target="_blank">MkDocs</a> site. Memory management is the deepest part (the page allocator, reclaim, folios, huge pages, `get_user_pages`, the kernel's address-space layout), with broader coverage of the scheduler, networking, the architecture layer, io_uring, locking, filesystems, and more.

Two principles I try to hold to — precisely *because* I'm not the expert in the room:

- **Every claim should trace to a primary source** — a specific commit, an LWN article, an lore.kernel.org thread. The authority lives in those sources, not in me. If I can't cite it, it doesn't belong on the site; and if I've paraphrased something wrong, you can go read the original and catch me.
- **The cross-references stay honest** — there's CI that fails the build on any broken internal link or anchor, so as the site grows the links between pages don't quietly rot.

## What I try to do on a page

The goal isn't to list the right structs and functions — docs.kernel.org and the source already do that better than I could. What I aim for is an honest attempt at *why*. Take `get_user_pages`: the part I found worth understanding wasn't the function signature, it was why the kernel ended up with two nearly-identical APIs, why an ordinary reference count couldn't express "this page is being used for DMA," and how that ambiguity led to a long tail of subtle bugs. I don't always get this right, and some pages are thinner or shakier than others.

## Where it's going, and a request

It's open source, and there's a lot of surface left — scheduler extensions, Rust in the kernel, the mainlining of PREEMPT_RT, and plenty of subsystems that are still thin.

More than anything, I'd welcome corrections. I *will* get things wrong — I'm learning this as I go — and a confidently wrong explanation of *why* is worse than admitting I don't know. If you understand this better than I do (many of you will), please tell me where I've missed. Take a look at <a href="https://kernel-internals.org" target="_blank">kernel-internals.org</a>, and be generous with the red ink.
