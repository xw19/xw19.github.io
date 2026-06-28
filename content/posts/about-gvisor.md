---
title: "Down the gVisor Rabbit Hole: A Sunday Afternoon Deep Dive"
date: 2026-06-28
draft: false
tags: ["kubernetes", "cri-o", "gvisor", "linux", "security"]
categories: ["infrastructure", "security"]
description: "Reflecting on gVisor's architecture, syscall interception, and the road to making it work with CRI-O after a long break from open-source."
---

It’s a quiet Sunday afternoon. I've got a hot cup of cha on the desk, a terminal window open, and finally some headspace to write a new post—especially now that I'm starting to make some real headway on a new CRI-O feature.

This whole journey started because I wanted to get back into open-source. I’d taken a pretty long hiatus after my daughter was born, and between changing jobs and moving to a new city, my focus was just elsewhere. But once the moving boxes were finally out of the way and my routine settled, I felt the itch to contribute again.

I reached out to Peter Hunt ([@haircommander](https://github.com/haircommander)), who helps lead the CRI-O project, to see if there was anything interesting to look into. He pointed me toward a long-standing integration puzzle: getting gVisor to play nicely with CRI-O, specifically around how the runtime handles multi-container pods.

That was all the spark I needed. I accepted the challenge and started digging.

---

## What actually is gVisor?

If you haven't run into gVisor before, it’s a security tool that sits in a really interesting sweet spot.

Normally, containers aren't virtual machines; they share the host operating system's kernel. That's great for performance, but if a containerized app exploits a kernel vulnerability, it can break out. You can use heavy VMs to isolate things, but they are slow and resource-heavy.

gVisor virtualizes the kernel itself in user space. It’s essentially a user-space kernel written in Go. When a container tries to make a system call, gVisor intercepts it and handles it in a safe, non-privileged sandbox. The host kernel never even sees it.

To do this, it splits the work. You have the **Sentry**, which is the actual user-space kernel doing the syscall interception. Then there’s the **Gofer**, a helper process that handles file system access so the Sentry never has to touch host files directly. Finally, **runsc** wraps it all up so it looks like a standard container runtime to Kubernetes and CRI-O.

---

## The Specific Challenge: Issue #10313

The issue Peter pointed me to ([gVisor Issue #10313](https://github.com/google/gvisor/issues/10313)) already had some great analysis by [@hbhasker](https://github.com/hbhasker) and a few others. They did the heavy lifting of mapping out the problem, but even with their notes, looking at the gVisor codebase for the first time was pretty intimidating. Navigating a user-space kernel is a whole different beast.

The bug itself is a classic logical mismatch. gVisor’s multi-container design expects a single, long-running "root" container to host the Sentry. But in Kubernetes, if a Pod uses `initContainers`, the very first container CRI-O creates is that init container.

Naturally, gVisor assumes this first container is the "root" container. But init containers are designed to do a quick job, exit, and destroy themselves. Once it exited, gVisor’s lifecycle management got confused because its "root" was gone, leaving the actual application containers stranded. Because the CRI spec doesn't explicitly flag what is or isn't an init container, CRI-O and gVisor were just talking past each other.

---

## Where are we using gVisor?

While reading up on all of this, I got curious about who actually uses gVisor in production. It turns out it powers some massive cloud services. Google Cloud Run V1 uses it to isolate customer containers running on shared serverless infrastructure, and GKE uses it for their sandbox feature.

But the most interesting modern use case is definitely AI. When platforms like ChatGPT or local LLM agents execute dynamically generated Python code on the fly, they run it inside gVisor sandboxes. It’s perfect for that because it starts instantly (unlike a VM) but ensures the untrusted code can't touch the host. AI! interesting but not really!

---

## The "Aha!" Moment

While trying to wrap my head around this translation layer, it clicked. This is the exact same design pattern we see in some of the coolest tech around.

Think about WSL 1 translating Linux calls to Windows NT APIs on the fly. Or Apple's Rosetta 2 translating x86 instructions to ARM64 so legacy apps run on Apple Silicon. Even Wine and Proton do this, translating Windows DirectX and Win32 calls to Linux and Vulkan so we can play Windows games.

It's the same elegant trick: intercept the interface, emulate the behavior, and map it to a different backend. gVisor is just doing it for security.

---

## What’s Next?

I went from _"let's see what this is"_ to _"I need to make this work"_ very quickly.

In fact, I've already gone ahead and implemented the PRs to fix this! It was a steep learning curve after being away from the codebase for so long, but it felt great to get it working.

If this successfully goes through, it will easily be my biggest contribution to the AI ecosystem. In the world of LLMs and autonomous AI agents—where systems are constantly generating and executing untrusted Python code on the fly—secure, lightweight sandboxing is everything. Making gVisor work seamlessly with CRI-O unlocks this capability for enterprise-grade Kubernetes and OpenShift clusters.

In my next post, I'll dive into the actual implementation details, the code changes, and how it all came together.

Time to pour another cup of cha. ☕
