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
| [OS/ROADMAP.md](OS/ROADMAP.md) | Decisions log, groundwork, next steps |

[CLAUDE.md](CLAUDE.md) holds the coding rules that apply while writing code.

## Status

The platform is being built as standalone apps sharing one Supabase project, surfaced through a hub of cards. The service layer described in these documents does not exist yet.

Read [OS/ROADMAP.md](OS/ROADMAP.md) before assuming anything here is built.
