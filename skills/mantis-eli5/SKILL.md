---
name: mantis-eli5
description: Use when someone asks what Google's Mantis is, what the mantis-* skills do, which Mantis stage to run, how the Mantis pipeline fits together, or how Mantis differs from PhonePe's Mantis recon framework or the MantisBT bug tracker. Also use when explaining Mantis to a non-security audience.
---

# Mantis, explained simply

Google's Mantis (github.com/google/mantis, Apache-2.0) is **a security review team made of AI agents**, shipped as 21 skills plus an optional Python harness. It is not a scanner. It is a pipeline that finds a bug, argues with itself about whether the bug is real, proves it, fixes it, and only then tells a human.

**Core principle: every stage exists to make the previous stage prove itself.** A scanner reports. Mantis prosecutes, then defends, then sentences.

## Disambiguation — three unrelated projects named Mantis

Say which one is meant before answering. Getting this wrong wastes the whole conversation.

| Project | What it is | Use when |
|---|---|---|
| **google/mantis** | Skills that review **your source code** with AI agents | "find vulns in my repo", "secure code review" |
| **PhonePe/mantis** | CLI that scans **your internet-facing domains** (subdomains, ports, certs, secrets, phishing lookalikes) | "what's exposed on our domains", attack-surface management |
| **mantisbt/mantisbt** | A PHP **bug tracker** from 2000, unrelated to security tooling | "we file bugs in Mantis" |

## The pipeline

The 16-stage default run (`reference/workflow.json`). Findings that fail a gate skip straight to calibrate — they are scored and kept, not deleted.

```
history → structural-index → architecture → threat-model → plan → researcher
                                                                     ↓
                                                                  dedupe
                                                                     ↓
   ┌──── confirmed? ──── review ←────────────────────────────────────┘
   │  no ↓        yes ↓
   │      ...      critic ──── viable? ──no──┐
   │                  yes ↓                  │
   │              reproduce ─── worked? ──no─┤
   │                  yes ↓                  │
   │           chain → patch                 │
   │                  ↓                      │
   └──────────→ calibrate ←──────────────────┘
                      ↓
                 reflect → report
```

## What each stage is for

Think of it as a newsroom: reporters find stories, editors kill the weak ones, fact-checkers demand proof, and only then does anything print.

**Build the map (nobody hunts yet)**
- `mantis-history` — reads git history for bugs you already fixed once, so you don't reintroduce them
- `mantis-structural-index` — builds a semantic index so agents navigate code instead of grepping
- `mantis-architecture` — turns raw analysis into a knowledge base
- `mantis-threat-model` — writes down *who* attacks you and *where* they touch the system
- `mantis-plan` — turns the threat model into a concrete list of files worth auditing

**Hunt**
- `mantis-researcher` — actually audits the code and files findings
- `mantis-dedupe` — collapses the same bug reported five ways

**Disprove (the part that makes Mantis different)**
- `mantis-review` — checks each finding against real source; kills false positives
- `mantis-critic` — "would this still fire in a production build with assertions off?" Kills debug-only bugs. Explicitly instructed to distrust every earlier stage.
- `mantis-reproduce` — writes and runs an actual exploit. Sandboxed.
- `mantis-chain` — combines several low-severity bugs into one high-severity path

**Fix and grade**
- `mantis-patch` — writes a minimal fix, then *re-attacks it* to confirm the fix holds. A patch is only `VERIFIED_SECURE` when a separate adversarial pass fails to bypass it.
- `mantis-calibrate` — scores severity against 27 anti-inflation rules
- `mantis-reflect` — records what was learned this pass
- `mantis-report` — writes the human-facing packet

**Use day to day (the one to actually adopt first)**
- `mantis-advise` — *before* you write code, asks: what has broken in this file before, what fix was verified, what was already triaged as a false positive
- `mantis-meta-agent`, `mantis-pipeline-adapter`, `mantis-configure`, `mantis-launch` — orchestration and setup

## The two ideas worth stealing even if you never run Mantis

**1. A verifier must consume something that cannot argue back.** Mantis will not call a bug real because a model said so. It demands a reproducer that reaches the sink, and will not call a patch good until a *different* agent has tried to break it and failed. A second model agreeing with the first is not verification.

**2. Calibrate severity downward by default.** The 27 rules in `mantis-calibrate/references/calibration_rules.md` exist because LLMs inflate severity and drown humans in "CRITICAL". The governing idea is **marginal capability**: score what the attacker *gains over the position they already hold*. Host attacking its own guest gains nothing → LOW. Bug not reproduced → capped LOW. All XSS → capped HIGH at most, usually MEDIUM. An admin exploiting a bug to read a file the admin could already download → capped MEDIUM.

## Running it

```bash
cd reference && ./install.sh
python3 scripts/configure.py --auto
./run.sh path/to/code
./run.sh path/to/code --objective "Audit for SSRF in webhook handlers"
```

**Two warnings from the maintainers, both load-bearing:**
- It generates and executes code autonomously. Run it in an isolated environment, never against production or internal networks.
- Findings are AI-generated and may be hallucinated. A human expert verifies before anything is reported. Do not mass-file AI-generated reports to open-source maintainers.

## Producing the explainer

When asked to *explain* Mantis (rather than run it), publish an Artifact.

**REQUIRED: load `artifact-design` before writing the page, and `artifact-diagramming` before drawing.** Then:

- **Draw the pipeline, not a logo.** The diagram must show the mechanism the explanation turns on: findings flowing through gates, and the kill-paths where a finding drops out to calibrate. A row of 16 labeled boxes with no arrows is a restated list, not a diagram.
- **Label every arrow** (`confirmed`, `viable`, `repro failed`, `re-attack`). Unlabeled arrows say only "related somehow".
- **Show the disprove loop as the centerpiece** — it is what distinguishes Mantis from a scanner, so it should be what the eye lands on.
- Use `currentColor` so it reads in light and dark; reserve one literal hue for the kill-path.
- Lead with the newsroom analogy, then the diagram, then the stage table. Keep the three-Mantis disambiguation visible near the top.
