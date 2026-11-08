---
type: knowledge-bundle-index
title: "Tahir's List KB"
description: "Entry point for the Tahir's List knowledge base: product, process, operations, design, QA, security, and research knowledge maintained by AI agents and validated as an OKF v0.2 bundle."
tags: [tahirs-list, halal-food-finder, knowledge-base, okf]
status: active
sources:
  - resource: https://github.com/Losers-Lab/tahirs-list
    title: "Tahir's List implementation repository"
  - resource: https://github.com/GoogleCloudPlatform/open-knowledge-format
    title: "OKF specification (v0.2)"
---

# Tahir's List KB

The knowledge base for [Tahir's List](https://github.com/Losers-Lab/tahirs-list),
separated from the implementation repository so that feature docs stay with the
code they describe and everything else lives here as an agent-maintained,
OKF v0.2-conformant knowledge bundle.

## Layout

| Directory | Contents |
|---|---|
| `product/` | Product vision, personas, brand decisions, open product questions |
| `process/` | Development process, engineering workflow, team and roles |
| `operations/` | Operations, infrastructure, deployment, cron automation |
| `work/` | Work ledger: what shipped, what remains |
| `risks/` | Risk register, current-state assessments and their evidence |
| `roadmap/` | Roadmap and decisions pending; prioritized fixes |
| `references/` | Glossary and external references |
| `setup/` | Machine setup and troubleshooting |
| `design/` | UI/UX design decisions, screen specs, tokens |
| `qa/` | QA conventions, test-gap plans, assessments |
| `reviews/` | Review records |
| `security/` | Security reviews and threat context |
| `research/` | Research briefs and external-services notes |
| `architecture/` | Architecture goals, scalability and reliability notes |

## Maintenance rules

1. **OKF v0.2 is the structure contract.** Every note carries YAML frontmatter;
   `type` is required. Validate with `basic-memory okf check` before merging.
2. **Agents write through the Basic Memory MCP tools** (structured notes with
   observations and typed relations), or plain markdown PRs for large imports.
3. **Feature-specific docs stay in tahirs-list** under `docs/features/SC-*`;
   only cross-cutting knowledge lives here.
4. **Work in small PRs**; every change to this repo goes through a GitHub PR
   like the implementation repo does.
