# DT Technical Specialist

A Claude Code project context repository for a Dynatrace Technical Specialist AI agent. Load this repo as your Claude Code project directory and the agent answers Dynatrace, Bindplane, Bluebox, and OpenPipeline questions with the precision and guardrails of a senior field engineer.

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

## How it works

This repo contains no executable code — only curated Markdown documents that Claude Code loads as project context. `Instructions.md` defines the agent's persona, question routing logic, and guardrails. The seven reference files are its knowledge base.

When you open this directory in Claude Code, the agent has immediate access to all reference material and applies the routing rules in `Instructions.md` automatically. No additional setup is required beyond having Claude Code installed.

## Audience

Dynatrace Technical Specialists, field engineers, pre-sales engineers, and support practitioners who need fast, accurate answers during customer engagements, proof-of-concepts, or troubleshooting sessions.

## Setup

1. [Install Claude Code](https://docs.anthropic.com/en/docs/claude-code)
2. Clone this repository:
   ```
   git clone https://github.com/virtualrussel/DT_Technical_Specialist.git
   cd DT_Technical_Specialist
   ```
3. Open the directory with Claude Code:
   ```
   claude
   ```
4. Ask a Dynatrace question. The agent uses the reference files automatically.

## Reference files

| File | Covers | Staleness window |
|------|--------|-----------------|
| `Instructions.md` | Agent persona, routing rules, guardrails | Author-controlled |
| `dynatrace_reference_guide.md` | Core platform architecture: OneAgent, ActiveGate, Grail, deployment modes, OpenPipeline | ~90 days |
| `dynatrace_api_quick_reference.md` | Classic vs. Platform API generations, auth models, token scopes, common endpoints | ~90 days |
| `dynatrace_dql_reference.md` | DQL syntax, pipeline model, Smartscape topology queries, performance tuning | ~90 days |
| `dynatrace_common_questions.md` | Pre-structured FAQ for pre-sales and support scenarios | ~90 days |
| `dynatrace_troubleshooting_trees.md` | Decision trees for OneAgent, Kubernetes, API auth, ingest, and OpenPipeline issues | ~90 days |
| `dynatrace_bindplane_reference.md` | Bindplane architecture, pipeline configuration, Dynatrace integration status | ~30 days |
| `dynatrace_bluebox_reference.md` | Bluebox AI SRE agent: workspace setup, investigations, GitHub integration | ~14 days |

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
