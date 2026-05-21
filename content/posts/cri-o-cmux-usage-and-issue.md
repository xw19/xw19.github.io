---
title: "Investigating CRI-O's cmux usage: a code-reading journey"
date: 2025-05-22
draft: false
tags: ["kubernetes", "containers", "cri-o", "linux", "internals"]
categories: ["infrastructure"]
description: "Reading through CRI-O source, I stumbled onto an abandoned dependency with CVE exposure. Here's what I found, the issue I filed, and my take on the path forward."
---

I've been reading through the CRI-O codebase lately — partly out of curiosity, partly to solidify my mental model of how container runtimes wire themselves together. While doing that I tripped over a piece of plumbing that caught my attention: `cmux`.

## What I found in main.go

While stepping through `cmd/crio/main.go`, I noticed CRI-O uses `soheilhy/cmux` — a connection multiplexer — to share a single Unix socket between gRPC and HTTP traffic.

```go
// cmd/crio/main.go
m := cmux.New(lis)
grpcL := m.MatchWithWriters(cmux.HTTP2MatchHeaderFieldSendSettings("content-type", "application/grpc"))
httpL := m.Match(cmux.HTTP1Fast())

go grpcServer.Serve(grpcL)   // CRI v1 Runtime + Image
go httpServer.Serve(httpL)   // /info, /config, /containers/:id, /pause/:id, /unpause/:id, /debug/*
go m.Serve()
```

The idea is clean enough — one socket, two protocols, cmux sniffs the first bytes and routes traffic accordingly. Worth noting: the metrics server and pprof endpoints already use *independent* listeners. So cmux in `main.go` is the only remaining usage of this dependency.

At first glance it looks like a neat trick. But then I checked the upstream project.

## The dependency is abandoned

`soheilhy/cmux` hasn't seen a commit since **February 5, 2021**. 26 open issues, 10 open PRs, some dating back to 2019 — zero maintainer response. It's effectively a ghost town.

The library pins a stale `golang.org/x/net` snapshot from November 2020, which predates several CVE fixes in that package:

| CVE | Impact | Fixed in |
|---|---|---|
| CVE-2022-41723 (CVSS 7.5) | HPACK quadratic-time DoS | x/net v0.7.0 |
| CVE-2023-39325 (Rapid Reset) | HTTP/2 stream-reset DoS | x/net v0.17.0 |
| CVE-2023-45288 (CONTINUATION flood) | Header parsing DoS | x/net v0.23.0 |
| CVE-2024-45338 | Non-linear HTML parsing | x/net v0.33.0 |

I want to be careful not to overstate this. Go's minimum-version-selection means CRI-O's top-level `go.mod` likely pulls in a fixed version of `golang.org/x/net` already, so the actual exploitability in a compiled CRI-O binary is unclear to me — I didn't trace the full dependency graph to confirm. What I *can* say is that `go.mod` / `go.sum` still *record* cmux's stale pin, so SBOM and CVE scanners (Trivy, Grype, Snyk, AWS Inspector) will flag CRI-O with "High" and "Critical" findings regardless of the true runtime exposure. Whether those scanner findings translate to real risk in a CRI-O deployment, I genuinely don't know — and with the library unmaintained, there's nobody upstream to even investigate it.

There are also some code-quality signals that illustrate just how stale the library is:
- References to `tls.VersionSSL30` (deprecated since Go 1.13; an unmerged PR to remove it has sat open since Nov 2019)
- Uses `ioutil.Discard` (deprecated since Go 1.16)
- Calls `net.Error.Temporary()` (deprecated since Go 1.18)
- The `HTTP1Fast` matcher — the one CRI-O uses — has a known bug where it silently drops `PATCH` requests

## Looking at alternatives

So I surveyed the alternatives to understand what a replacement would actually look like:

| Option | Maintained | Fit | Notes |
|---|---|---|---|
| Two separate listeners (stdlib) | n/a | ★★★★★ | Cleanest technically, but non-trivial for OpenShift |
| `elastic/gmux` | Active (Apr 2026) | ★★★★ | Near-drop-in, modern x/net, ~400 lines |
| `cockroachdb/cmux` | Semi-internal | ★★★ | Stopgap at best |
| `grpc.Server.ServeHTTP` | Part of grpc-go (EXPERIMENTAL) | ★★ | Avoid |
| `connect-go` | Active, larger change | ★★ | Out of scope |

**`grpc.Server.ServeHTTP`** sounds tempting but is still marked EXPERIMENTAL in grpc-go, silently ignores keep-alives and max-connection-age options, and benchmarks put it at roughly 2× slower than native gRPC (73 µs/op vs 37 µs/op). Not viable on a kubelet-critical path.

**`elastic/gmux`** is an actively maintained drop-in from Elastic — modern Go, modern `golang.org/x/net`, built specifically for the gRPC+HTTP multiplexing case. The codebase is small and auditable. Main concern is bus factor (one org effectively maintains it), but that's a manageable risk.

**Two separate listeners** is the architecturally purest answer and matches how containerd handles it. But as I dug into the OpenShift context below, I realized it's not a trivial swap.

## Why the single-socket design exists

Before jumping to conclusions, I wanted to understand *why* CRI-O chose this design in the first place. Two reasons stood out:

**1. OpenShift + SELinux.** OpenShift runs on RHCOS with strict SELinux MCS policies. Every socket on the host needs to be labeled and tracked. A single socket keeps the policy surface small and auditable.

**2. `oc debug node`.** OpenShift worker nodes are effectively immutable — you can't just SSH in and `curl` a local endpoint. Instead, admins use `oc debug node`, which launches a privileged diagnostic container on the target node. If the HTTP inspect API were on a separate socket path, you'd need to mount that socket into the debug container as well — a small but real operational change across OpenShift tooling and documentation. The current single-socket design lets you volume-mount one `.sock` file and get both gRPC and HTTP debug access from the same file descriptor.

So this isn't a correctness bug — it's a design tradeoff with real OpenShift implications. Splitting the sockets cleanly would mean touching OpenShift's node debugging workflow, SELinux policy, and probably some operator configuration. That's a cross-team decision, not just a CRI-O one.

## What I did with this

I filed [Issue #9969](https://github.com/cri-o/cri-o/issues/9969) on the CRI-O repo with the full research: CVE table, alternatives evaluation, code-quality audit, and a breakdown of the OpenShift constraints. The goal was to surface the problem clearly so the maintainers and Red Hat stakeholders can decide the right path.

**My own take:** I'd go with `elastic/gmux` as the replacement. It's a near-drop-in swap — the API is close enough that the diff in `main.go` is minimal, it removes the abandoned dependency and its scanner noise, and it doesn't require any architectural changes. Splitting to two separate sockets is the purer solution, but it's an OpenShift-wide architectural decision that involves more stakeholders than just the CRI-O team. That's not my call to make.

## Why I did this at all

Honestly, this investigation was primarily about solidifying my own understanding of how CRI-O wires up its runtime interfaces — the relationship between the gRPC listener, the HTTP inspect API, and how cmux sits between them. The issue submission was a natural byproduct: once I had a clear picture of the problem and the options, it seemed worth documenting for the people who actually own the codebase.

The full details are in the issue if you want to dig in or weigh in.

Happy hacking.
