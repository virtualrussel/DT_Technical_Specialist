# DT Technical Specialist

A Claude Desktop project providing a Dynatrace Technical Specialist AI agent. Configure it once in Claude Desktop and the agent answers Dynatrace, Bindplane, Bluebox, and OpenPipeline questions with the precision and guardrails of a senior field engineer.

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

## How it works

This repo contains no executable code — only curated Markdown documents used as context in a Claude Desktop project. `Instructions.md` defines the agent's persona, question routing logic, and guardrails. The seven reference files are its knowledge base.

Claude Desktop projects let you attach both a system prompt (Project Instructions) and a set of knowledge files to a persistent conversation context. `Instructions.md` goes in as the Project Instructions; the seven reference files are uploaded as project knowledge. Every conversation in the project starts with all of this context already loaded.

## Audience

Dynatrace Technical Specialists, field engineers, pre-sales engineers, and support practitioners who need fast, accurate answers during customer engagements, proof-of-concepts, or troubleshooting sessions.

## Setup

You need [Claude Desktop](https://claude.ai/download) with a Pro or Team plan (project support is not available on the free tier).

1. Clone this repository:
   ```
   git clone https://github.com/virtualrussel/DT_Technical_Specialist.git
   ```
2. In Claude Desktop, create a new project.
3. Open the project's **Instructions** field and paste the full contents of `Docs/Instructions.md`.
4. Upload all seven reference files as project knowledge:
   - `Docs/dynatrace_reference_guide.md`
   - `Docs/dynatrace_api_quick_reference.md`
   - `Docs/dynatrace_dql_reference.md`
   - `Docs/dynatrace_common_questions.md`
   - `Docs/dynatrace_troubleshooting_trees.md`
   - `Docs/dynatrace_bindplane_reference.md`
   - `Docs/dynatrace_bluebox_reference.md`
5. Start a conversation in the project and ask a Dynatrace question.

## Reference files

| File | Covers | Staleness window |
|------|--------|-----------------|
| `Docs/Instructions.md` | Agent persona, routing rules, guardrails | Author-controlled |
| `Docs/dynatrace_reference_guide.md` | Core platform architecture: OneAgent, ActiveGate, Grail, deployment modes, OpenPipeline | ~90 days |
| `Docs/dynatrace_api_quick_reference.md` | Classic vs. Platform API generations, auth models, token scopes, common endpoints | ~90 days |
| `Docs/dynatrace_dql_reference.md` | DQL syntax, pipeline model, Smartscape topology queries, performance tuning | ~90 days |
| `Docs/dynatrace_common_questions.md` | Pre-structured FAQ for pre-sales and support scenarios | ~90 days |
| `Docs/dynatrace_troubleshooting_trees.md` | Decision trees for OneAgent, Kubernetes, API auth, ingest, and OpenPipeline issues | ~90 days |
| `Docs/dynatrace_bindplane_reference.md` | Bindplane architecture, pipeline configuration, Dynatrace integration status | ~30 days |
| `Docs/dynatrace_bluebox_reference.md` | Bluebox AI SRE agent: workspace setup, investigations, GitHub integration | ~14 days |

## Staleness policy

Every reference file carries a `Last verified` date and a staleness window. The agent treats these as freshness signals:

- **~90 days** (core Dynatrace files): re-verify before answering questions involving specific version numbers, UI menu paths, API field names, or token scope names.
- **~30 days** (Bindplane): acquired April 2026; integration depth with core Dynatrace changes frequently. Re-verify integration claims every time.
- **~14 days** (Bluebox): launched weeks before initial verification; CLI flags and setup steps change at high frequency. Treat every specific detail as a hypothesis to confirm against [docs.bluebox.ai](https://docs.bluebox.ai).

Live documentation always wins over the reference files. When a discrepancy is found, the agent flags it explicitly.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to update reference files, report stale content, or propose new coverage.

## License

This work is licensed under [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/). See [LICENSE](LICENSE) for details.
