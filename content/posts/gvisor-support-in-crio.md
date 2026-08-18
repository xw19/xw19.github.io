---
title: "gVisor and CRI-O: It Finally Merged"
date: 2026-08-19T00:30:00+05:30
draft: false
tags: ["kubernetes", "containers", "cri-o", "gvisor", "linux", "security"]
categories: ["infrastructure", "security"]
description: "A 1 AM update on the gVisor PR that finally completes the CRI-O integration—and why this one means a lot to me."
---

It is 1 in the morning as I write this, and I am absolutely thrilled.

In my [last post](/posts/about-gvisor/), I wrote about falling down the gVisor rabbit hole and the strange little problem around CRI-O, init containers, and multi-container pods. At the time, I said I had started working on the fix.

The CRI-O side of that work, [PR #9974](https://github.com/cri-o/cri-o/pull/9974), merged a while back. But that was only one half of the story. The gVisor PR still had to make its way through review, and after almost two months, [PR #13279](https://github.com/google/gvisor/pull/13279) merged just a few moments ago.

Honestly, this is one of the most interesting things I have worked on in a long time.

It was not just about making a change in a config file. I had to understand how gVisor starts, how CRI-O starts runtimes, and the small assumptions each project makes about the other. I got to read code in two projects I really admire, make mistakes, ask questions, and slowly turn a confusing failure into something that actually works.

## What we changed

The two PRs had to meet in the middle. The CRI-O change made it possible to run gVisor through the right runtime path. The gVisor change taught it how to recognize CRI-O's sandbox and keep all the containers in a pod together, instead of treating them like separate little worlds.

We also fixed a case where gVisor could stop too aggressively when some optional information was not available, and added a simple guide for getting the two projects running together. None of it is flashy, but it is the kind of work that makes a feature actually useful for people.

That mattered to me. I did not want this to be a change that only worked on my machine, with one setup, on one good day. The goal was to make the integration fit naturally into both projects without making life harder for everyone else.

## I thought it would never merge

There were definitely moments when I thought this would never make it over the finish line.

Open source can be intimidating when you are coming back after a break. You see a large codebase, experienced maintainers, review comments, CI jobs, and all these things that seem like they belong to people who have been doing this forever. There were plenty of times I looked at the gVisor PR and wondered whether I had missed something obvious or whether it would simply stall.

But the reviews made the change better. The questions forced me to understand the boundaries more clearly, and the process showed me that a contribution does not have to be huge to be meaningful. It just has to solve a real problem carefully.

Seeing it merge felt unreal.

## Why this one stays with me

gVisor is one of those projects that makes you stop and appreciate how much thoughtful engineering is hidden below the surface. It gives containers stronger isolation without asking people to give up the speed and convenience that makes containers useful in the first place. Being able to help make that story work better with CRI-O and Kubernetes is genuinely exciting.

This was also a personal milestone. After time away from open source, it reminded me how much I enjoy working on systems problems—following a thread through the code, understanding why something behaves the way it does, and collaborating with people who care deeply about getting the details right.

## What is next?

The work is merged, but there is still a nice next step for anyone interested in picking it up: integration tests. It would be great to have real end-to-end coverage in both gVisor and CRI-O, so we can keep proving that the two projects work together as they evolve. If that sounds interesting to you, I would love to see it happen.

If you would like the technical details, the [gVisor PR](https://github.com/google/gvisor/pull/13279) and the earlier [CRI-O PR](https://github.com/cri-o/cri-o/pull/9974) have them all. For tonight, I mostly just wanted to pause and enjoy this moment.

I am also starting to think seriously about finding a job where I can work on projects like this more often. If you are hiring for systems, containers, Kubernetes, Linux, or open-source infrastructure work, please let me know.

Time to sleep. 😴
