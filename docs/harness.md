# The harness behind these projects

The projects in this portfolio were built with AI coding agents. This page describes the working system around the agent, not the agent: the rules, files, hooks and review steps that make its output checkable.

This setup came from my own professional experience, and I built it before reading anything formal on the subject. It is organized by five concerns that [Learn Harness Engineering](https://walkinglabs.github.io/learn-harness-engineering/en/) also lays out, which supports the choices after the fact: constrain behavior, maintain context, stop premature victory, verify work, make the runtime observable. Each section says what exists, whether it is **mechanical** (enforced by a tool whoever is typing) or **advisory** (a rule an agent is asked to follow), and where it has failed.

The honest summary: the advisory layer catches most things, and every time a rule was broken twice it was promoted to a mechanical one. A review of the commit history in October 2026 checked this claim and found it only partly true, which is why the second half of this page exists.

## At a glance

| Concern | Mechanical | Advisory |
|---|---|---|
| Constrain behavior | `commit-msg` hook (no AI trailer, proof line on Critical/High fixes), subagent guard hook, structural safety in JSCC (sanitizer, mode isolation) | Coding-preferences file, per-project `AGENTS.md` (including a stay-inside-the-active-feature rule), Explore-only reviewers |
| Maintain context | Plan, decision, changelog and gate docs in the repo, claims and links pinned by tests | Daily running log |
| Stop premature victory | Harness-controlled `passing` state, an initialization gate before any feature work, eval pass thresholds in code, mutation-verified guards | "Never claim a pass you did not run", retrospective step |
| Verify work | CI, `pre-push` hook (same gates as CI, tested for equality), scanner, type checks, record/replay evals, feature-list check | `/pre-commit-check`, two-lens gate review |
| Observable runtime | Call ledger and cost report in JSCC, CI status script | Gate-doc audit trail |

## 1. Constrain agent behavior

**Instruction layers.** A personal instruction file loaded into every session. A separate coding-preferences file states how to work: scope, plan, execute, verify; product decisions are mine and arrive as numbered options; two failed attempts at the same thing and the agent stops; a check-in every three to five actions so I can redirect early; smallest change that works; real implementations only, no mocks that fake a pass. Each repo has a short `AGENTS.md` with run and verify commands, its hard rules and links to topic docs, so a fresh session can start from the repo alone.

One giant instruction file fails, so the always-on file stays short and points elsewhere, and recurring workflows are skills, not paragraphs in it.

**Skills.** Sixteen named workflows, each a single file: `/gate-review`, `/plan-review`, `/ux-gate-review`, `/pre-commit-check`, `/backlog-prune`, `/test-prune`, `/docs-consolidate` and `/harness-audit` for engineering, plus daily-planning and job-search workflows that live in a private workspace and are not published.

**Mechanical constraints.**
- A `commit-msg` hook rejects AI co-author trailers. This started as a written rule, recurred twice from a stale default in the assistant's own commit instructions, and was fixed by history rewrite both times before it became a hook. The same hook requires a `Proof:` line (reproduction, mutation or deletion) on a fix that names a Critical or High finding. It checks presence only, so leaving the proof out is deliberate, not accidental.
- JSCC makes the unsafe thing impossible instead of discouraged: one sanitizer is the only path to a model, payloads carry an HMAC so a forged or mutated one fails verification, and real and synthetic data are separated by a database mode marker. The design rule is "safety is structural, not habitual."
- Review subagents are spawned as read-only Explore agents with an explicit no-write instruction. A general-purpose reviewer once committed an unrequested edit mid-gate; a PreToolUse hook now also blocks a subagent from changing git state in a main checkout (builder subagents in worktrees are allowed).

## 2. Maintain context

**The repo is the system of record.** Plans, decisions (ADRs with rejected alternatives), changelogs and gate documents are files in the project, so a new session can reconstruct state from the tree instead of a chat. Changelogs are compacted one phase behind, so the record stays readable.

**A verification ledger, not a roadmap.** `FEATURES.json` holds check gates and controls, each with its observable behavior, a verification command and a state; `PROGRESS.md` is generated from it and never hand-edited. I tested whether this carries continuity by withholding both files from a fresh agent: it still answered all five onboarding questions, from the Status tables in the docs, and gave a fuller account of what was open. So the ledger is where "is it verified" lives. Product progress and open work live in the README and the project's Status table.

**A running log.** Every session appends timestamped one-line entries to a daily note. It is the continuity layer across interruptions.

**Claims pinned by tests.** Numbers and status statements that docs quote are asserted by tests against their source, so a stale claim fails a build instead of waiting for a reviewer. In JSCC this is one registry: each claim (test, ADR, threat and phase counts, eval figures) names its file, its phrase and its source of truth, and a sweep fails any count-shaped phrase with no row. Review-finding ids are banned outside the gate record by a test that shares a denylist with the private-path check, and the changelog has a line and byte budget plus a header that must match the phase table, so compaction is forced by a failing build rather than a standing step.

**Where it fell short.** Docs drift. A README test count went stale four times before a test pinned it. Anything fixed by a one-time scrub or a written close-out step recurred; only pinned claims stayed fixed.

## 3. Stop premature victory

The failure to guard against is an agent saying "done" because the diff looks right.

- **The harness decides what is done, not the agent.** A feature moves to `passing` only through `features.py verify`, which runs the feature's own command and records sealed evidence (commit, time, exit code). Only one feature is active at a time. A hand edit to `passing` fails the integrity check in the pre-push hook. The seal is tamper-evidence, not security: the point is that marking something done becomes a deliberate, visible act.
- **Never say a test passes without running it this turn.** Pasted output, including failures.
- **Differential proof for Critical and High findings.** A fix is closed only with a reproduction (fails before, passes after), a mutation-verified guard (revert the fix, the test must fail), or a deletion proof. This came from a guard test that kept passing with the guard removed.
- **Do not trust a "FIXED" in a prior gate doc.** Re-probe the behavior. Carried-forward residuals get their severity re-derived, because a disposition can itself be wrong. One documented-and-accepted DNS gap in JSCC's fetcher was later shown exploitable and fixed in code.
- **Grep for the claim, not the line.** A stale claim fixed at the cited line usually survives elsewhere in different words.
- **A retrospective is a closing step.** A gate pass is not closed until the owner's lessons are recorded next to the draft ones, each with a disposition (applied, deferred with a trigger, or noted with a reason).

## 4. Verify work

**Layers, cheapest first.**
1. **Tests and evals written first.** For model behavior the eval is the test: cases and expected properties before the prompt. Pass thresholds live in code, recorded model responses are replayed keyed on a hash of the post-sanitizer prompt, and results are reported as bands when capture-to-capture variance is large (76% and 82% on the same prompt).
2. **`/pre-commit-check`.** Format, lint, build, tests, scanner, trailer and feature-list checks on touched files, run before any push is proposed, reported in one status line.
3. **`pre-push` hook.** Runs the feature-list check, then the same steps CI runs, in CI's order, so a red CI run is a local failure first. About 5 seconds for the browser project's full check (plus its headless UX checks), about 25 in JSCC. A local gate that is a subset of CI's goes red after the push: JSCC's CI failed at the type check because the hook did not run it, and platform-gated code type-checked on Windows and not on CI's Linux runner. The hook now runs every CI gate, a test asserts that, and the type checker is pinned to Linux.
4. **CI.** Actions are SHA-pinned and installs use a frozen lockfile. It runs the feature-list check and replays every recorded eval through the shipped CLI, and a test pins which case ids pass per suite, not only how many.
5. **Two-lens gate review at phase boundaries.** An adversarial code review plus an outside-reviewer walkthrough, both read cold (the reviewers are barred from the prior findings until they have their own), then reconciled and labeled NEW, REPEAT-OF, or CONTRADICTS. CONTRADICTS, a finding that overturns a prior "closed," is the highest-signal type. A backlog sweep runs first so deferred items cannot ride through another boundary.
6. **A UX pass on the running product** in every theme at narrow and wide widths, which caught what both code lenses missed (a silent overwrite on a content-derived id).

**What reviews have found.** Every gate run has found something real, including a regression of a seeded-RNG fix that its own test did not detect, and a trust-boundary bug structurally identical to one already solved elsewhere in the same codebase. A hand sweep for "the same shape elsewhere" was then wrong within a day, so the class was moved into a table-driven registry test: every client-writable field on the web forms and the ingest commands gets a row, discovered from the routes and the command tree, and a new field fails until it has one. Building it found a real gap (pasted text from stdin or a file had no size ceiling, unlike a fetched page) and, as a side effect, a quadratic email pattern that took 46 seconds on a 160 KB token; both are fixed and tested. The scanner got the same treatment: a self-test of its true positives and its known false-positive shapes, and of its shell wrapper's exit code, which replaced four call-site patches with one carve-out.

## 5. Make the runtime observable

- **JSCC's call ledger.** Every model call is recorded at the call site with cost, latency, and a failure marker, so spend and failed calls are visible instead of inferred. A billed call whose response failed to parse used to write nothing; the rule now is that nothing that can fail after money is spent belongs inside the instrumented function.
- **Honest reporting of cost and time.** An active-time estimate from commit history prints its own bias.
- **A CI status script** that reads the latest run per repo and says whether it matches the local HEAD, because the GitHub CLI is not installed.
- **Gate documents** as an append-only audit trail of what was found, what closed it and with what proof.
- **An end-of-session clean-state check.** `features.py clean` verifies the feature list, runs the full check, looks for debug leftovers in the diff, and warns about uncommitted changes and extra worktrees. A session is not finished until it passes or says why not.

## Starting a project: the initialization phase

A new project does setup only in its first session, with no feature code until an acceptance checklist is ticked: one setup command, pinned runtime, frozen lockfile, a passing example test, lint and format clean, CI running the same commands as the local check, the hooks installed, a feature list whose verify commands exist, a short `AGENTS.md`, and a fresh-session test (a new session with only the repo can say what the project is, how to run and verify it, and where it stands). A scaffold script writes the template, the hooks and the runner in one command. This exists because both projects reached CI late (about commit 90 of 160 in JSCC, commit 160 of 180 in the browser project), and the linter found real bugs the day it landed.

**The phase is a gate, not only a checklist.** A scaffolded `FEATURES.json` starts with init pending. While it is pending, `activate` refuses and the integrity check in the pre-push hook fails if any feature has moved past not started, so a hand edit cannot skip the phase. `features.py init-check` verifies the items a tool can check: placeholders filled, hooks present and wired, line endings and runtime pinned and the pin matched in the CI workflow, a lockfile, a CI workflow that runs the feature check, at least three features, and the full check green. `init-check --complete` needs a clean committed tree and records a sealed checkpoint. The fresh-session test stays manual, and a flag records that it was done. A project with no init record is treated as legacy and not gated. Eleven tests cover the gate.

**Bringing the two existing projects under it found real gaps.** JSCC had no line-ending pin and the browser project had no runtime pin. JSCC's recorded full-check command omitted the type check its own `AGENTS.md` listed. The runner script itself failed JSCC's formatter on its first copy, which would have turned CI red. The fresh-session test, run as read-only subagents with only the repo, answered all five questions in both projects, and also surfaced drift a reviewer would have caught later: a stale progress file, a README that pointed at a section that did not exist, and a superseded decision still listed as current. All were fixed.

## Auditing the harness

`/harness-audit` runs periodically. It scores the five subsystems with evidence, runs the fresh-session test, lints the instruction file for size and for rules with no source or expiry, checks that each control has actually fired, and proposes one disable-and-measure experiment. A control with no evidence of ever catching anything gets a deliberate test that it still can.

## Diagnosing a failure by layer

| Failure layer | First place to look here |
|---|---|
| Task specification | The feature's behavior and verify command in `FEATURES.json`. No verify command means the step is not specified yet. |
| Context provision | `AGENTS.md`, plan doc, ADRs, the changelog's current phase |
| Execution environment | CI config, lockfile, the pinned runtime |
| Verification feedback | The hooks, CI, and the gate's "verified clean" list |
| State management | README and project Status table, daily log, gate doc dispositions (`PROGRESS.md` for verification state only) |

## Known gaps

Stated so the page does not oversell the setup.

- **Hooks are per clone.** `core.hooksPath` is local git config. A fresh clone runs no hooks until `git config core.hooksPath .githooks` is run once, which `AGENTS.md` and the scaffold both say. CI runs the same checks as a backstop.
- **`--no-verify` exists.** The hooks make skipping a checks step visible, not impossible. The rule is that a bypass gets said out loud.
- **The feature-list seal is friction, not security.** An agent that reads the runner could recompute it. The control is review and the hook, not cryptography.
- **The subagent guard is a backstop with a known hole.** It blocks git-state changes from a subagent in a main checkout. It does not catch a subagent editing files there; the first control is spawning reviewers as read-only Explore agents.
- **CI status is readable only for public repos without a token.** Both projects are public now, so the status script reports real results for them; a private repo needs a read-only token, and without one the script reports "unknown," not "green." Before the second project was public, its CI could not be confirmed from the agent side and was checked by hand.
- **The advisory layer is still a large part of the harness.** Coding preferences and the gate method rely on the agent and owner following them. The commit-history review found several prose steps (a test-name check, a changelog compaction step, a sweep for the same defect shape) that recurred after being written down; only rules turned into tests stayed fixed. The registry and budget tests above are that conversion, applied to the recurring ones; the remaining prose steps are still prose.
- **Evals are small.** Case counts are sized against measured noise but remain tens of cases, and a prompt tuned against its own fixtures is a known risk the gate method calls out.
- **Scope is a written rule, not a check.** Every `AGENTS.md` says to stay inside the active feature, but nothing compares a diff to the paths a parallel lane was given. Leaks across lanes have happened (a change to a shared renderer reached screens the plan said were unchanged), and plan review now checks for them in advance. A diff-scope check is deferred until a leak recurs despite that.
- **Type checking is enforced, strictness and coverage are not.** JSCC runs mypy at default settings and the browser project type-checks its JavaScript with `tsc`, both in the hook and in CI with no suppressions in the browser project. JSCC's strict mode is a per-module ratchet still to come. Coverage is reported in CI but not gated, and the threshold is undecided.
