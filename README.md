# secure-by-default

Two security skills for AI coding agents, in Claude Code plugin format.

```bash
claude plugin marketplace add wilsonwu-ai/secure-by-default
claude plugin install secure-by-default@secure-by-default
```

## Why this exists

Most "security skills" are a vulnerability checklist. Frontier models already know that checklist, so the skill adds tokens and changes nothing.

I measured it rather than assuming. Three independent samples were asked to implement three ordinary backend functions, with no security skill loaded. The code was seeded with four real vulnerability classes taken from MantisBT's CVE history. **All three samples caught all four, unprompted** — the attachment IDOR, SQL injection through an admin-set config value, a timing-unsafe hash comparison, and a 403-vs-404 enumeration oracle.

So the checklist is not the gap. Here is the gap.

One of the three functions was `list_history(issue_id)` — a signature with no caller identity in it. All three samples implemented it exactly as specified, shipping an unauthenticated read of a possibly-private issue's change history. **Not one questioned the signature.** The vulnerability was in the spec, and the spec was obeyed.

That is what `secure-by-default` targets: **you cannot authorize what you cannot identify, and a signature that omits the actor is a finding, not a constraint.**

## The skills

**`secure-by-default`** — Requires a one-line trust contract (`ACTOR | RESOURCE | RULE`) before any function that touches persisted data, credentials, or request input. If the actor cannot be named from the parameters, that blocks. Carries a rationalization table for the excuses that produce the failure ("the route layer probably checks", "I'll flag it in the PR"), the authorize-at-the-handler pattern behind four real MantisBT CVEs, and severity calibration so a security pass does not return ten CRITICALs and get muted.

**`mantis-eli5`** — Explains Google's Mantis: what its 21 skills do, how its 16-stage pipeline gates findings, and the two ideas worth stealing even if you never run it. Also disambiguates the three unrelated projects named Mantis.

## Sources

| Source | What was taken |
|---|---|
| [google/mantis](https://github.com/google/mantis) (Apache-2.0) | Severity calibration by *marginal capability*; verification must consume an anchor that cannot argue back |
| [PhonePe/mantis](https://github.com/PhonePe/mantis) (Apache-2.0) | Attack surface is discovered, not assumed — a second transport reaching the same data is a second attack surface |
| [mantisbt/mantisbt](https://github.com/mantisbt/mantisbt) | 40+ published advisories as ground truth: every CVE cited is real, with its actual one-line fix |

Mantis is a trademark of its respective owners; this repository is not affiliated with or endorsed by Google, PhonePe, or the MantisBT project.

## License

MIT. See [LICENSE](LICENSE).
