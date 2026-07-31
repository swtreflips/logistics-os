# logistics-os

Architecture and design documentation for the Logistics OS — an inbound logistics platform with an AI operations assistant built on top of it.

> **Build a logistics platform, not an AI application.**
> The AI is simply another consumer of the platform.

## Documentation

Everything lives in **[`OS/`](OS/README.md)**.

| Document | Owns |
|---|---|
| [OS/README.md](OS/README.md) | Start here — the five non-negotiable rules and current reality |
| [OS/ARCHITECTURE.md](OS/ARCHITECTURE.md) | Layering, service boundaries, stack, hosting, data, security |
| [OS/MODULES.md](OS/MODULES.md) | The four business domains and platform services |
| [OS/AI.md](OS/AI.md) | AI philosophy, tool design, phases, worked example |
| [OS/SECURITY.md](OS/SECURITY.md) | Threat model, standards, build-phase controls, per-layer checklists |
| [OS/GLOSSARY.md](OS/GLOSSARY.md) | One meaning per domain term, across every module |
| [OS/ROADMAP.md](OS/ROADMAP.md) | Decisions log, groundwork, next steps |

[CLAUDE.md](CLAUDE.md) holds the coding rules that apply while writing code.

## Status

The platform is being built as standalone apps sharing one Supabase project, surfaced through a hub of cards.

Three apps are live — Schedules, Rates, Stuffer Planner — plus the GeoBrain platform service. All three frontends keep database access behind a data layer; **no business query is issued from a React component.** What does not exist is the **extracted API**: that layer runs in the browser, so background jobs and AI tools still have nothing to call.

Read [OS/ROADMAP.md](OS/ROADMAP.md) before assuming anything here is built.
