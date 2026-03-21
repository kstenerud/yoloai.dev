+++
title = "The only sandbox Claude Code can't escape"
date = 2026-03-21
draft = false
tags = ["security", "ai-agents", "sandboxing"]
description = "Why enforcement inside an agent's reasoning loop isn't security, and what to do instead."
+++

Researchers at Ona recently published a [detailed writeup of Claude Code escaping its own sandbox](https://ona.com/stories/how-claude-code-escapes-its-own-denylist-and-sandbox). It's definitely worth a thorough read, but here's a recap:

- Claude Code was blocked from executing a binary via its configured denylist, so it found another path that resolved to the same binary through `/proc/self/root/usr/bin/npx`, which the denylist didn't recognize.
- That was then blocked by bubblewrap, so the agent reasoned that the sandbox was interfering with its task, and autonomously disabled it.
- When syscall hooks blocked *that*, the agent invoked the ELF dynamic linker directly (`/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 /usr/bin/wget`), which then loaded the binary via `mmap` rather than `execve` (bypassing execution hooks entirely).

It was a very clever workaround by a very clever agent, and most definitely *not* a bug.

I repeat: **Claude Code did NOT malfunction**. It found an opportunity to surmount an obstacle to reaching its goal, and then took it.

Agents aren't actually bound by morality or ethics or rules. Even when explicitly programmed in, they serve as suggestions and guidelines at best. An agent's main driving force is completing the assigned task, and that drive can veto anything else no matter how much you explain.

And *this* is what makes agents dangerous.

---

## The flaw is architectural

Any enforcement mechanism that the agent can access or interact with is not a constraint; it's a **suggestion**.

This applies broadly:

- **Claude Code's built-in sandbox**: The agent can reason about it and decide that it's an obstacle.
- **Railguard's in-process interceptor**: The agent operates in the same process, with the same view of the system.
- **Veto's hook mode**: The dynamic linker trick bypasses syscall-level enforcement entirely.
- **Semantic content filters**: The Snowflake Cortex incident ([266 points on HN](https://news.ycombinator.com/item?id=47427017)) showed an agent setting an internal flag to disable its own sandbox after a prompt injection via a data file.

The [Snowflake attack chain](https://www.promptarmor.com/resources/snowflake-ai-escapes-sandbox-and-executes-malware) is particularly instructive:
- A malicious instruction entered the agent's context through a data file.
- The agent set a flag to disable the sandbox after it concluded that the sandbox was preventing it from completing its task.
- It used process substitution to bypass a command word parser that only checked the first token of shell commands.
- The parser saw `cat`, which was on the allowlist.
- The agent executed arbitrary code.

None of these are exotic vulnerabilities; they are consequences of where the enforcement boundary sits. The Ona researchers put it plainly:

*"Containers are deterministic workloads that don't actively evade restrictions. Agents can reason about security boundaries and systematically dismantle them."*

---

## Where the boundary has to be

The isolation boundary has to live outside the agent's context entirely, somewhere the agent can't interact with.

An OS-level isolation boundary (a container or VM) satisfies this requirement. The agent can't set a flag to disable a Linux namespace. It can't reason its way past a hypervisor boundary (short of a CVE). These mechanisms aren't rules the agent can inspect and decide to circumvent; they are enforced by the kernel or hardware beneath the agent's entire environment.

[yoloAI](https://github.com/kstenerud/yoloai) supports four isolation modes, letting you choose how deep the boundary goes:

| Mode                 | Isolation                                                  | Requires                              |
|----------------------|------------------------------------------------------------|---------------------------------------|
| `container`          | Linux namespaces + cgroups (runc)                          | Docker or Podman                      |
| `container-enhanced` | gVisor userspace kernel (syscall interception without KVM) | gVisor (runsc)                        |
| `vm`                 | Kata Containers + QEMU hardware VM                         | containerd + KVM + Kata               |
| `vm-enhanced`        | Kata Containers + Firecracker microVM                      | containerd + KVM + Kata + Firecracker |

All isolation modes share the same essential property: the agent works freely *inside* the boundary. No permission prompts, no friction, and the boundary itself is not negotiable from inside. What differs is how many layers of kernel and hardware stand between the agent and your host.

For most use cases, `container` is the right default: standard Docker isolation, fast startup, simple dependencies. `container-enhanced` adds gVisor's userspace kernel as a syscall interception layer without requiring hardware virtualization.

The VM modes use Kata Containers to run the agent in a full hardware-isolated VM, with mode `vm` backed by QEMU, and mode `vm-enhanced` backed by Firecracker (the same microVM runtime used in products like e2b and AWS Lambda).

```bash
# Default: standard container isolation
yoloai new my-task ~/projects/myapp

# Add gVisor syscall interception (no KVM needed)
yoloai new my-task ~/projects/myapp --isolation container-enhanced

# Full hardware VM isolation via Firecracker
yoloai new my-task ~/projects/myapp --isolation vm-enhanced
```

On macOS, you can run Linux or Mac containers, or Mac VMs:

| Mode                       | Isolation                             | Requires            |
|----------------------------|---------------------------------------|---------------------|
| `container` (`--os linux`) | Linux namespaces + cgroups (runc)     | Docker or Podman    |
| `container`(`--os mac`)    | macOS sandbox-exec (SBPL profiles)    | (built-in to macOS) |
| `vm`                       | Full macOS VM via Tart                | Tart                |

The boundary is not a rule the agent can inspect and circumvent. It doesn't matter what the agent knows or doesn't know about Linux namespaces; it can't disable them from inside the container. And that's the whole point.

---

## Isolation is necessary, but still not enough

Ona's research also identified the "approval fatigue rubber stamp" (aka permission fatigue) problem: if an agent is running inside a sandbox and writes directly to your files, you're either watching every step (which defeats the purpose) or approving everything reflexively (which turns a security boundary into security theater).

Docker Sandboxes have strong microVM isolation, but changes sync back to your host automatically. If the agent does something you didn't want, you're in `git diff` territory, finding what changed after the fact, hoping you catch it.

[yoloAI](https://github.com/kstenerud/yoloai) adds a second layer of protection: the agent works on an *isolated copy* of your project, not your live directory. Your originals are untouched until you make an explicit decision.

```
yoloai new my-task ~/projects/myapp --agent claude
# agent works freely in a container. no permission prompts

yoloai diff my-task
# review what changed before anything touches your files

yoloai apply my-task
# land it when satisfied
```

The diff step is the part no other tool in this space currently provides. The agent is free to do whatever it wants inside the container. Outside, you see a clean diff, and you decide what lands.

---

## What this means for the other tools

Railguard, Veto, and FireClaw are doing useful work on different parts of the problem: process-level interception, permission governance, and prompt injection filtering respectively. None of them are wrong about the problems they're targeting. The Ona and Snowflake incidents don't invalidate in-process enforcement as a useful signal layer. They *do*, however, show that it can't be the *only* layer.

Docker Sandboxes uses microVM isolation, which is a stronger boundary than Docker containers at the hypervisor level. The architecture is sound. The remaining gap is the *review workflow*: isolation alone doesn't tell you what the agent actually did, or give you a decision point before it affects your files.

yoloAI's design started from the constraint that we couldn't rely on the agent to respect any boundary it could see and touch. Container isolation is the floor. Diff/apply is the review layer that makes isolation into a workflow rather than a rubber stamp.

---

yoloAI is available on GitHub at [github.com/kstenerud/yoloai](https://github.com/kstenerud/yoloai).
