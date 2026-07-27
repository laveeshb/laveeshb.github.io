---
layout: post
title: "kernel-internals.org: Learning the Why Behind the Linux Kernel"
date: 2026-07-26
order: 8
categories: [linux]
tags: [linux-kernel, documentation, open-source, learning-in-public]
excerpt: "Making sense of why the Linux kernel is built the way it is. I'm new to kernel development, and working through the why is a big part of how I'm learning — everything traced to primary sources."
---

Let me calibrate expectations first: I'm new to kernel development. Getting into it recently made me realize how much of the kernel I'd been depending on without really understanding — and that's a big part of why I started <a href="https://kernel-internals.org" target="_blank">kernel-internals.org</a>. It's where I work through the *why* behind the kernel's design as I dig into it, and organize it so the next curious person doesn't have to start from scratch.

So think of the site less as an authoritative reference and more as careful notes, taken in public. This post is about why I started it and where it stands.

## The gap I kept running into

When I started reading, the material felt oddly polarized. <a href="https://docs.kernel.org" target="_blank">docs.kernel.org</a> is authoritative but tells you *what* and *how* — it's reference documentation, not a narrative. <a href="https://lwn.net" target="_blank">LWN</a> is the gold standard for the *why*, but it's news-shaped — tied to a particular patch series — and not organized as something you can sit down and browse by subsystem.

What I kept wishing for was a current, organized place that explained the *reasoning*: the trade-offs, the alternatives that got rejected, the mailing-list arguments that produced the design we ended up with. I couldn't find it, so I started assembling it as I learned. It's meant to complement docs.kernel.org, not compete with it.

## Where it stands today

Today it spans roughly 375 pages across 28 subsystems, built as a static <a href="https://squidfunk.github.io/mkdocs-material/" target="_blank">MkDocs</a> site. Memory management is the deepest part (the page allocator, reclaim, folios, huge pages, `get_user_pages`, the kernel's address-space layout), with broader coverage of the scheduler, networking, the architecture layer, io_uring, locking, filesystems, and more.

Two principles I try to hold to — precisely *because* I'm not the expert in the room:

- **Every claim should trace to a primary source** — a specific commit on <a href="https://git.kernel.org" target="_blank">git.kernel.org</a>, an <a href="https://lwn.net" target="_blank">LWN</a> article, a <a href="https://lore.kernel.org" target="_blank">lore.kernel.org</a> thread. The authority lives in those sources, not in me. If I can't cite it, it doesn't belong on the site, and where I've summarized something the original is one click away.
- **The cross-references stay honest** — there's CI that fails the build on any broken internal link or anchor, so as the site grows the links between pages don't quietly rot.

## What I try to do on a page

The goal isn't to list the right structs and functions — docs.kernel.org and the source already do that better than I could. What I aim for is an honest attempt at *why*. Take <a href="https://kernel-internals.org/mm/gup/" target="_blank"><code>get_user_pages</code></a>: the part I found worth understanding wasn't the function signature, it was why the kernel ended up with two nearly-identical APIs, why an ordinary reference count couldn't express "this page is being used for DMA," and how that ambiguity led to a long tail of subtle bugs. I don't always get this right, and some pages are thinner or shakier than others.

## Where it's going

It's open source, and there's a lot of surface left to cover — scheduler extensions, Rust in the kernel, the mainlining of PREEMPT_RT, and plenty of subsystems that are still thin. I'll keep adding to it as I learn.

If any of this is your kind of thing, it's all at <a href="https://kernel-internals.org" target="_blank">kernel-internals.org</a>.
