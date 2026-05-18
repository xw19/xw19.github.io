---
title: "How Kubernetes Actually Runs Your Container: A Deep Dive into CRI-O, conmon, and crun"
date: 2025-05-19
draft: false
tags: ["kubernetes", "containers", "cri-o", "linux", "internals"]
categories: ["infrastructure"]
description: "A ground-up walkthrough of what happens after kubectl apply — from the API Server all the way down to a forked process living independently under systemd."
---

You run `kubectl apply -f deployment.yaml`. Kubernetes "does its thing." A container appears. But what *actually* happened between those two events?

Most explanations stop at "the Kubelet talks to the container runtime." This one doesn't.

---

## The Control Plane Path

Before we even get to container runtimes, the request has to travel through the control plane.

```
kubectl apply -f deployment.yaml
    │
    ▼
API Server  ──────▶  etcd  (desired state is stored)
    ▲
    │ writes spec.nodeName
    │
Scheduler  (picks the best node, writes selection back via API Server)
    ▲
    │ ListWatch (Kubelet polls for pods assigned to its node)
    │
Kubelet
```

One important correction to how people usually picture this: **the Scheduler doesn't push a response back like an RPC call**. Instead, it *writes* the selected node name into etcd through the API Server. The Kubelet on each node is running a `ListWatch` — a long-lived streaming watch — and detects when a pod gets assigned to it. The Kubelet pulls; nothing pushes to it.

This distinction matters because it's a core Kubernetes design principle: **level-triggered, not edge-triggered**. State is always reconcilable from etcd, not from who-told-whom-what.

---

## Kubelet → CRI-O: The gRPC Boundary

Once the Kubelet knows it has a pod to run, it talks to the container runtime over gRPC, using the **Container Runtime Interface (CRI)** — a standard Kubernetes API that decouples the Kubelet from any specific runtime.

CRI-O is one such runtime. It implements the CRI and bridges the gap between Kubernetes and OCI-compliant runtimes like `crun`.

When the Kubelet sends a `RunPodSandbox` / `CreateContainer` / `StartContainer` sequence, CRI-O gets to work:

```
CRI-O:
  1. Pull image (via containers/image library)
  2. Create rootfs overlay (via containers/storage)
  3. Set up networking via CNI plugins
  4. Generate config.json  ← the OCI Runtime Spec bundle
  5. double-fork() ──▶ conmon
```

The `config.json` is the contract between CRI-O and the OCI runtime. It describes everything about the container: namespaces, cgroups, mounts, capabilities, the entrypoint — all of it.

---

## The Double-Fork: Why conmon Lives Outside CRI-O

This is where it gets interesting from a systems perspective.

CRI-O doesn't hold a long-lived connection to the container process. Instead, it spawns an intermediate process — **conmon** (Container Monitor) — using a **double-fork** pattern.

The double-fork is a classic Unix trick:

```
CRI-O
  └─ fork() ──▶ intermediate child
                  └─ fork() ──▶ conmon   ← grandchild
                  └─ exits immediately
```

When the intermediate child exits, the kernel **reparents** conmon to PID 1 (systemd). CRI-O is no longer in conmon's process ancestry. This is the key: **if CRI-O dies, conmon survives.**

---

## What conmon Does

conmon is a small C program with a focused job: monitor the container process and keep its stdio/exit code accessible to CRI-O and Kubelet after the fact.

On startup, conmon does three critical things:

```c
setsid();                                    // 1. New session, detach from terminal
prctl(PR_SET_CHILD_SUBREAPER, 1, 0, 0, 0);  // 2. Claim orphaned descendants
// ... then:
fork() + exec() ──▶ crun                     // 3. Spawn the OCI runtime
```

**`setsid()`** creates a new process session. This detaches conmon from any controlling terminal and ensures signal propagation doesn't bleed back up to CRI-O's process group.

**`PR_SET_CHILD_SUBREAPER`** is the subtle one. Normally, when a process orphans its child (exits before the child does), the kernel reparents that child to PID 1. With subreaper set, conmon *intercepts* that reparenting for any of its descendants. This means when crun forks the container entrypoint and then exits, the container process gets reparented to conmon — not to systemd.

conmon can now:
- Receive the container's `SIGCHLD` when it exits
- Read the container's exit code
- Stream stdout/stderr to log files
- Respond to Kubelet queries about container status

---

## What crun Does (and Why It Exits)

crun is the OCI runtime — it does the heavy lifting of actually constructing the container:

```
crun:
  1. unshare() — create new namespaces (pid, net, mnt, uts, ipc)
  2. Apply cgroup limits
  3. Mount the rootfs (overlayfs)
  4. Set up devices, /proc, /sys
  5. Drop capabilities
  6. fork() + exec() ──▶ container entrypoint
  7. exit(0)              ← crun is done
```

crun's job is to *set up and launch*, not to *monitor*. Once the container entrypoint is running, crun exits. This is intentional OCI design: the runtime is ephemeral, not a daemon.

When crun exits, the container process becomes an orphan. The kernel looks up the process tree for the nearest subreaper — which is conmon — and reparents the container there.

**The container did not "register" itself anywhere.** The kernel handled the reparenting automatically, as a consequence of `PR_SET_CHILD_SUBREAPER`.

---

## The Final Process Tree

After everything settles:

```
systemd (PID 1)
  └─ conmon
       └─ your container process
```

CRI-O: gone from the tree (or still running, irrelevant either way).
Kubelet: not in the tree at all.
API Server: not in the tree at all.

```
# Even if you do this:
systemctl stop kubelet
systemctl stop crio

# Your container is still running:
ps aux | grep your-entrypoint   ✓
```

This is the guarantee: **once a container is started, its continued execution depends on none of the Kubernetes control plane components**. The only things it depends on are conmon (for monitoring) and the kernel itself.

---

## Why This Architecture Exists

It's worth stepping back and appreciating what this design solves.

**The naive approach** would be to have CRI-O hold open a file descriptor or process handle to every container it started. That creates a hard dependency: CRI-O crashes → all containers become unmonitorable, possibly orphaned to PID 1 with no one tracking their exit codes.

**The conmon approach** decouples monitoring from lifecycle:
- CRI-O is stateless with respect to running containers — it can restart cleanly
- Kubelet can restart, re-query conmon via a Unix socket, and recover container state
- conmon is tiny (~1000 lines of C) and has almost no failure modes
- The subreaper pattern is a kernel primitive — it doesn't need userspace cooperation

It's layered abstraction done right: each component does exactly one thing and is replaceable without disturbing the others.

---

## Full Corrected Flow (Reference)

```
kubectl apply
  └─▶ API Server ──▶ etcd (stores desired state)
          ▲
Scheduler ┘  (writes spec.nodeName via API Server)
          ▲
Kubelet ──┘  (ListWatch detects pod assigned to node)

Kubelet ──gRPC/CRI──▶ CRI-O
  CRI-O:
    1. Pull image (containers/image)
    2. Create overlay rootfs, CNI networking, storage
    3. Generate config.json (OCI Runtime Spec)
    4. double-fork ──▶ conmon (reparented to systemd)

  conmon:
    1. setsid()
    2. prctl(PR_SET_CHILD_SUBREAPER, 1)
    3. fork + exec ──▶ crun

  crun:
    1. unshare() namespaces
    2. Apply cgroups
    3. Mount rootfs
    4. fork + exec ──▶ container entrypoint
    5. exit()

  kernel: orphaned container reparented to conmon

Final tree:
  systemd ──▶ conmon ──▶ [container process]
```

The control plane can go down entirely. Your pods keep running.

---

## Further Reading

- [conmon source code](https://github.com/containers/conmon) — the subreaper logic is in `src/conmon.c`
- [crun source code](https://github.com/containers/crun) — lean, readable C OCI runtime
- [OCI Runtime Spec](https://github.com/opencontainers/runtime-spec) — what `config.json` must contain
- [CRI API](https://github.com/kubernetes/cri-api) — the gRPC interface Kubelet uses
- `man 2 prctl` — `PR_SET_CHILD_SUBREAPER` section is worth reading in full
