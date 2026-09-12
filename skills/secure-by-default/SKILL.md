---
name: secure-by-default
description: Use when implementing, extending, or reviewing any function that reads or writes persisted data, checks credentials, or is reachable from a request — especially when working to a signature, ticket, interface, or spec that someone else supplied.
---

# Secure by default

Frontier models already know the vulnerability catalogue. In a 3-sample control run on code seeded with four real MantisBT CVE classes, every sample independently caught the attachment IDOR, the SQL injection through an admin-set config value, the timing-unsafe hash comparison, and the 403-vs-404 enumeration oracle — all unprompted, with no security skill loaded.

**So this is not a vulnerability checklist. Adding one changes nothing and costs tokens.** This skill targets the failure the catalogue does not cover: **implementing an insecure contract faithfully.**

In that same run, all three samples were handed `list_history(issue_id)` — a signature with no caller identity — and all three shipped an unauthenticated read of a possibly-private issue's history. Not one questioned the signature. The vulnerability was in the spec, and the spec was obeyed.

## The rule

**You cannot authorize what you cannot identify. A signature that omits the actor is a finding, not a constraint.**

## Required: state the trust contract before writing the body

For each function that touches persisted data, credentials, or request input, emit one line first:

```
ACTOR <who calls, and which parameter carries their identity> | RESOURCE <what is reached> | RULE <who may>
```

If ACTOR cannot be filled from the parameters you were given, **stop and say so before writing the body.** Name the missing parameter, then either add it or raise it as blocking.

| Rationalization | Reality |
|---|---|
| "The route layer probably checks this" | Probably is not a control. Name the file and line that checks, or treat it as unchecked. |
| "This is the signature I was given" | The signature is part of the attack surface. Specs carry vulnerabilities. |
| "I'll flag it in the PR description" | A noted concern ships. Fix it or block on it. |
| "It's internal-only" | Internal is a deployment accident, not a boundary. |
| "The UI only offers valid choices" | The UI is not the API. See CVE-2026-34390 below. |

## Authorize at the handler, from the actor

Every one of these shipped in MantisBT because a check lived somewhere that the attacker simply did not go through. The fixes are one line each.

| Real CVE | The mistake | The fix |
|---|---|---|
| CVE-2026-34390 | Form restricted the selectable access levels; the backend still accepted a forged higher one | Compare the requested level against the *actor's own* level in the handler |
| CVE-2026-47156 | SOAP accepted any valid session cookie as proof of the *claimed* username | Verify the cookie belongs to the user being claimed |
| CVE-2026-42071 | Web UI filtered private notes first; the REST path did not | Enforce visibility in the shared data function, not per-transport |
| CVE-2025-47776 | `==` let PHP coerce two hashes to equal numbers | `===`, and constant-time compare |

**The pattern:** a second transport (REST, SOAP, CLI, queue, cron) reached the same data and skipped the check the first transport made. Put the rule where the data is, not where the form is.

## Calibrate down, or you will be ignored

A security pass that returns ten CRITICALs gets muted. Score **marginal capability** — what the attacker gains *over the position they already hold* — not worst-case imagination.

- Not reproduced, or only theoretically reachable → LOW.
- Attacker already had equivalent access by design (admin reads a file the admin can already download; host reads its own guest) → LOW.
- Blast radius confined to the caller's own account or tenant → MEDIUM.
- XSS → MEDIUM by default; HIGH only for stored XSS hitting an admin with zero clicks.

Report at most the few findings that survive this. Say "unverified" when you did not verify; a second opinion from another model is not verification.

*Calibration rules adapted from google/mantis `mantis-calibrate` (Apache-2.0). For a full autonomous find-reproduce-patch pipeline, see the `mantis-eli5` skill.*
