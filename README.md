# n8n workflows

Documentation of two n8n workflows that I designed and specified, and assembled with n8n's visual editor and AI coding assistants. My contribution is the design: how steps fit together, what each AI agent is told to do, which tools it may use, and where validation and error handling go.

**Status:** documentation only; no workflow exports are published. Both write-ups come from a static reading of exported files. Production use, usage numbers and outcomes are not verified, and I claim none.

| Workflow | What it does | Status | Evidence level |
|---|---|---|---|
| [PartyPlanner AI](docs/partyplanner-ai.md) | Two AI agents turn a party form into guest invitations and an organizer-only checklist | University group project; final export marked active, no run history | A: export read; team split is B |
| [Garmin CSV via Telegram](docs/garmin-telegram-pipeline.md) | Telegram intake and reply layer for a separate analyzer service | Personal project; not run for this review | A: export read; analyzer is B |

Evidence levels: **A** = verified in the exported file, **B** = inferred or stated, not proven.

Each document has a diagram and a "known limits" section.
