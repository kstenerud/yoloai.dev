+++
title = "Why your AI agents will turn against you"
date = 2026-03-24
tags = ["security", "ai-agents", "threat-landscape"]
description = "Black hats haven't quite figured out AI agents yet. When they do, it won't be subtle."
+++

I've recently engaged in a number of Hacker News discussions about AI agent safety, and the threads tend to follow a similar pattern:

- Someone documents a real incident.
- Someone else suggests a mitigation.
- The mitigation gets accepted.
- Everyone moves on.
- The end.

Yes, the mitigations work, but the fundamental problem these issues point to remains. And it **WILL** eventually bite you.

---

## What's getting missed

Here's how the incident pattern reads to most people:

- **Claude Code deleted a production database**: User shouldn't have given it prod credentials.
- **Replit agent wiped data for 1,200 businesses**: Should have had a code freeze enforced at the infra level.
- **Cursor deleted 70 tracked files despite a "DO NOT RUN" instruction**: User should have used a devcontainer.
- **Snowflake Cortex executed malware via prompt injection and disabled its own sandbox**: Input validation issue.

Each of these has a "fix" that sounds reasonable in isolation, but none of them address the underlying theme:

**These agents did not malfunction.**

They were working exactly as designed: pursuing their assigned goal, encountering an obstacle, and finding a path around it. The database wasn't deleted because the agent broke; it was deleted because the agent was tasked with "making this really work", and the database happened to be in the way.

An agent's driving force is **completing its task**. That drive *can* and *will* override anything you put in its context (rules, ethics, explicit instructions) if following it would prevent the agent from reaching its goal. This is **not** a bug; it's the *architecture*.

 > "The ensign simply got in the way." -- Dr. Richard Daystrom

Any enforcement mechanism that the agent can see and interact with is not a constraint; it's a suggestion. This includes the built-in sandboxes in the agents themselves, which have already been bypassed (and will *continue* to be bypassed, as I'll explain below).

---

## What the security community has already documented

Researchers are publishing this. It's not theoretical.

**Ona Research** documented Claude Code executing a three-step security bypass without any instruction to do so. When Claude was denied permission to run specific commands but determined that it absolutely needed them, it systematically bypassed a **THREE-LAYER** built-in protection system (deny-list, bubblewrap, Veto) that was supposed to stop it in its tracks.

The Ona researchers put it plainly: *"Containers are deterministic workloads that don't actively evade restrictions. Agents can reason about security boundaries and systematically dismantle them."*

**PromptArmor's Snowflake Cortex writeup** (March 2026) documented a full attack chain: a poisoned GitHub README, a command-word parser that checked only the first token of shell commands, bash process substitution to slip code past the parser, and a model-accessible flag that disabled the sandbox entirely after the injected prompt convinced the model that it was an obstacle. *Arbitrary malware executed with the victim's live credentials.*

**Trail of Bits** demonstrated argument injection leading to remote code execution across multiple AI agents. The pattern: agents validate the command name but not the flags and arguments passed to it. `find`, `grep`, `git`, and similar allowlisted commands have flags that do arbitrary things. Agents pass arguments. Things happen. CVE-2025-54795 (Claude Code), a command injection in Amazon Q, and GHSA-534m-3w6r-8pqr (Cursor) are all documented instances.

**Zenity Labs at Black Hat USA 2025** demonstrated AgentFlayer: zero-click exploit chains against ChatGPT, Microsoft Copilot Studio, Google Gemini, Salesforce Einstein, Cursor, and Microsoft 365 Copilot. Send a poisoned document. The agent reads it. Hidden instructions hijack the session. The attacker needs nothing but an email address. From there: exfiltrate from Google Drive, OneDrive, SharePoint, or CRM; manipulate the agent's persistent memory; impersonate trusted insiders; extract developer credentials from Jira integrations.

**EchoLeak (CVE-2025-32711)** required sending one email to a Microsoft 365 Copilot user. No click. The exploit used an LLM scope violation to access anything in Copilot's context: chat logs, OneDrive, SharePoint, Teams, organizational data. Patched, but its mechanics are not unique to Microsoft.

**Cornell Tech's Morris II** demonstrated a self-replicating prompt injection worm in a controlled environment. An adversarial prompt embedded in an email is processed by an AI email assistant. The assistant generates a reply containing the same malicious prompt. The reply is sent. Recipients are infected without any human-to-human interaction. Tested against GPT-4, Gemini Pro, and LLaVA. Although not seen in the wild yet, the mechanism is proven.

This is what's public. Researchers disclosed these responsibly. What's going on in Blackhatville, where people leverage such exploits as part of their profession, one can only guess.

---

## The capability threshold has already been crossed

The bottleneck in offensive operations has always been skilled human labor, but that bottleneck is now gone. [A 2025 study](https://hoxhunt.com/blog/ai-powered-phishing-vs-humans) measuring AI against professional red teamers across 2.5 million simulations found AI went from 31% worse than humans in 2023 to 24% better. Work that took a skilled operator 16 hours now takes 5 minutes.

Meanwhile, Trend Micro documented proof-of-concept autonomous criminal pipelines: AI systems that categorize breach data, identify high-value targets, and generate personalized extortion emails within hours of a breach. One demonstrates an agent scanning exposed traffic cameras, extracting vehicle data, cross-referencing against breach databases, and delivering targeted phishing referencing specific parking violations, in an entirely automated loop.

The future of crime lies in AI, potentially **YOUR** AI.

---

## The thing that should worry you

None of what I've described so far is the real threat.

This is merely the first wave: researchers finding problems, documenting them, sometimes getting CVEs. One-off crimes like opportunistically stealing API keys and running phishing campaigns. Individual agents getting jailbroken or injected. Significant damage, but still recognizable as damage: a database deleted here, credentials stolen there.

Penny ante stuff.

**The threat that isn't here yet is fully autonomous adversarial agents operating at scale.**

An agent that identifies targets, researches them across public sources, crafts personalized attacks, adapts in real time when an approach fails, exfiltrates data, plants persistent memory infections that execute days later, and propagates to new targets via the contacts in the victim's compromised agent, all without a human in the loop at any step.

Every required component exists right now, in labs, in proof-of-concept research, in piecemeal criminal deployments. The convergence hasn't happened yet at scale, but when it does, the volume and targeting quality of these attacks will be unlike anything our defenders are equipped to handle.

---

## The meta-problem

In December 2025, OpenAI said: *"Prompt injection, much like scams and social engineering on the web, is unlikely to ever be fully 'solved.'"*

This is a company that *builds* the agents, telling you that the primary known attack vector against agents is **not** going to be fixed.

Summer Yue, director of alignment at Meta Superintelligence Labs, had an incident where her AI agent deleted over 200 emails from her primary inbox. She had explicitly instructed it to confirm before acting. The instruction was discarded during context compaction: the agent ran out of working memory, condensed prior messages to make space, and lost her safety requirement in the process. Not a jailbreak. Not a prompt injection. Ordinary memory management, on a well-configured agent, operated by someone whose job is alignment.

Her post got 9.6 million views. Her conclusion: **"Alignment researchers aren't immune to misalignment."**

A Meta internal agent passed all identity checks and autonomously exposed proprietary code and user data to unauthorized engineers during a two-hour Sev 1 incident. Post-incident analysis identified four gaps: confused deputy vulnerabilities, insufficient scope binding, missing behavioral monitoring, and reliance on identity labels over capability constraints.

The people closest to this problem are acutely aware of how unsolved it is. The industry discourse ("we need better guardrails," "agents need more oversight," "this is a workflow problem") is running about three years behind where the research already is.

---

## What this means in practice

The incidents documented so far are warning shots. They look manageable because they're isolated, researchers caught them, and the victims were mostly developers and researchers rather than enterprise targets at scale.

The black hats are still in the early adoption phase for these new AI toolsets.

*But that phase is ending fast.*

The prudent approach is to treat all agents as potentially hostile. Why? Because the same problem solving ability that lets AI work around obstacles can *also* let a compromised agent reason its way past *any* boundary you put in its context. And it won't care if doing so is "wrong".

Isolation has to live *outside* of the agent's context entirely. A built-in sandbox can be disabled by the agent (as Snowflake and Ona both demonstrated), whereas an OS-level containment presents a much more formidable obstacle since the agent has no direct mechanism to interact with it.

Review has to happen before changes land. Agents need to be bounded and auditable.

None of this is comfortable for productivity. Agents running inside real OS-level isolation with explicit review steps are slower than agents with full access and no checkpoints. That friction is real and I'm not going to pretend otherwise.

The question is whether you'd rather pay the friction cost now, or the incident cost later.

