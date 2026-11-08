---
type: process-note
title: Review Order — Specialists Converge Before the Owner
description: Correction to the PR review order - the agent runs the Omar (security)
  and Yusuf (QA) rounds to convergence on the PR first, and only then asks the owner.
  Owner approval is the final gate, never a substitute for the specialist rounds.
tags:
- tahirs-list
- process
- review
- security
- qa
- pr
status: active
generated: 2026-11-08
generated_by: hamza
verified: true
sources:
- resource: AGENTS.md
  title: PR-based review flow (source of truth)
- resource: docs/process/feature-workflow.md
  title: Phases 11-12
- resource: .opencode/skills/tahirs-feature-workflow/SKILL.md
  title: Steps 11-12
- resource: https://github.com/Losers-Lab/tahirs-list/pull/77
  title: Process PR (merged as 394b6e5)
permalink: tahirs-list-kb/process/review-order
---

# Review Order — Specialists Converge Before the Owner

**Decided 2026-11-08.** The review order was implicit and was got wrong once: the
SC-209 MCP PR went to the owner before the specialist rounds had run. The rule is
now explicit and encoded in three places.

## The rule

1. The **agent runs the specialist rounds itself**: Omar (security, static) and
   Yusuf (QA, dynamic) review the PR diff and post `[omar]` / `[yusuf]` findings as
   PR comments.
2. Fix, reply per thread, re-request, until **both** post `Verdict: APPROVE` on the
   PR head.
3. Only then ask the owner to review. The owner sees the diff plus the recorded
   verdicts and approves or requests changes.
4. Owner-requested changes invalidate the head: they go back through the specialist
   rounds before the owner is asked again.
5. Three rounds without convergence → escalate to the owner **for a decision**
   (not a code review). Merge still requires both `Verdict: APPROVE` verdicts.

Owner approval is the final gate; it is **not** a substitute for the specialist
rounds. The verdicts are the reviewers' own assessments — the implementing agent
writes the fixes, not the verdicts.

## Where it lives

- `AGENTS.md` → PR-based review flow: the fixed order and the `Verdict: APPROVE`
  requirement for both specialists.
- `docs/process/feature-workflow.md` → Phase 11 (review to convergence) and
  Phase 12 (owner approval, merge, deploy), with the owner gate in the flowchart
  and the change-request loop returning to Phase 11.
- `.opencode/skills/tahirs-feature-workflow/SKILL.md` → Steps 11–12 for agents.
- `docs/handoff/03-development-process.md` → the single-account approval
  workaround now requires **both** attributed APPROVE comments, a green head, and
  owner approval (it previously accepted either reviewer's approval).

## Why it matters

On a single shared GitHub account, `reviewDecision` is always empty, so the
attributed review comments are the only approval evidence. "Either reviewer" would
have let a PR merge with the security review skipped. Requiring both, on the head,
after convergence, is what makes the review trail auditable.
