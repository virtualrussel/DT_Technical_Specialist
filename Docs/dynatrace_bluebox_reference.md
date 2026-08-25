# Bluebox Technical Reference Guide (Dynatrace)

> **Last verified against docs.bluebox.ai:** 2026-08-25
> **Product status:** Bluebox is currently in **public preview**. Per docs.bluebox.ai: "Features and APIs may change as we work toward general availability." Set expectations accordingly in any pre-sales conversation — this is not a GA product.
> **Staleness policy, read this before answering any Bluebox question:** This is the newest and fastest-moving product in this entire project, faster than Bindplane. Bluebox was unveiled publicly at AWS Summit New York only weeks before this file's original verification date (July 15, 2026). The Getting Started guide was at version 6.11 as of this file's last verification date; the CLI Reference at version 4.3. Treat every specific detail in this file (CLI flags, supported coding agents, setup steps, permission scopes) as likely to have shifted since verification. Search docs.bluebox.ai fresh before giving a customer-facing or pre-sales answer on Bluebox specifics. Do not rely on this file's content for anything version-sensitive without checking first.

**A note on the domains you may see referenced:** the product's marketing site is **bluebox.ai**, its app is at **app.bluebox.ai**, and its documentation is at **docs.bluebox.ai**. `bluebox.dynatrace.com` also resolves and redirects to the app. There is no separate `docs.bluebox.com`; if someone references that domain, they likely mean docs.bluebox.ai.

---

## What Bluebox Is

Bluebox is an **AI SRE agent**: it detects production issues from live OpenTelemetry telemetry, investigates root cause by building a causal evidence chain through your actual service topology, and hands the result to a coding agent (or to you) as an evidence-backed GitHub Issue.

**The core loop, per Bluebox's own framing:** production signal → root cause → evidence-backed fix.

Bluebox does not merge code or push to branches. It proposes; a human (or the human's coding agent, under human review) decides.

**CRITICAL ARCHITECTURAL FACT:** Bluebox has its own separate workspace, its own OTLP ingest endpoint, and its own runtime and ingest tokens. That means separate login, separate data plane from core Dynatrace. Bluebox findings do NOT appear in the Dynatrace UI. Bluebox investigations do NOT have access to Grail directly; they query Bluebox's own ingest plane. Do NOT claim unified investigation experience between Dynatrace and Bluebox without verifying it's actually shipped.

**Structural note:** Unlike Bindplane (an acquired, standalone company), Bluebox's own Terms of Service define "Bluebox" as **Dynatrace LLC**. This is a Dynatrace-built and Dynatrace-owned product operating under its own brand and its own site, not a separate acquired entity. Marketing material describes the reasoning layer as "Dynatrace powered insights" / "causal AI under the hood"; treat this as Dynatrace's existing causal-AI technology (the same lineage as Davis AI) repackaged behind a developer-focused, GitHub-native product surface, rather than a literal pass-through to the core Dynatrace platform APIs documented elsewhere in this project.

---

## Where Bluebox Fits

Per docs.bluebox.ai, Bluebox sits alongside your existing tools rather than replacing them:

- **Works inside your coding agent**: Claude Code and Kiro are fully supported; GitHub Copilot, Cursor, and OpenCode are experimental.
- **Creates issues and proposed fixes in your existing GitHub repository**: no new issue tracker to adopt.
- **Starts with standard OpenTelemetry for server-side service traces**: other signals (RUM, security, infrastructure, synthetic) can be used if already set up separately, but Bluebox's own setup flow only configures server-side OTel traces.

It is explicitly described as "the missing layer between your observability data and your development workflow," a production-context layer for AI coding agents, not a general-purpose observability platform in its own right.

---

## Core Concepts

Six concepts cover most of what you do in Bluebox:

### Findings
Bluebox continuously monitors connected telemetry and automatically surfaces problems as **Findings** on the Overview page, mapped to services and codebase context before you act on them.

### Investigations
The core of the Bluebox SRE agent. An investigation traces a problem through service topology, forms and tests hypotheses, and assembles evidence from live telemetry, streaming progress through visible phases (situational context → hypotheses → evidence gathering → report). Two entry points:
- **From a Finding**: fastest path, context already loaded.
- **From the Investigations page**: describe a symptom in your own words (an error, a misbehaving service, or an open GitHub issue's text) when no Finding exists yet.

Output: a root cause stated plainly, evidence tracing back to specific telemetry (not pattern-matching), affected services/endpoints, and a recommended fix, filed automatically as a GitHub Issue.

**Investigation states:** Open, Investigating, Resolved, Completed, Archived.

### Chat
A conversational interface for quick questions ("what are the top findings right now," "is checkout healthy"). Use this for exploration, use Investigations when you need a structured, evidence-backed result. Conversations persist across sessions.

### Routines
Scheduled agent executions. A routine runs a Bluebox agent on a schedule exactly as if you had asked the question yourself. Use cases: nightly cost reviews, weekly SLO checks, pre-release health reports. See the Routines section below for full configuration details.

### Bluebox CLI
Terminal access for setup and querying. The single most useful command: `bluebox ask "<question>"`.

### Proactive use
Bluebox isn't only reactive. Before writing code, ask it what production actually looks like right now:
```bash
bluebox ask "which endpoints in the payments service have the highest p99 latency"
bluebox ask "what are the most common errors in the last 7 days"
```
Paste the answer into a coding agent so it writes against production reality instead of an assumed mental model of the code.

---

## Getting Started / Setup Flow

**Prerequisites:** a GitHub account with the target repository, and a service instrumentable with OpenTelemetry (or existing OTel instrumentation that can be repointed at Bluebox's ingest).

**Step 1: Install the CLI.** Either paste an agent-driven install prompt into a coding agent, or run directly:

macOS/Linux:
```bash
curl -fsSL https://app.bluebox.ai/install.sh | bash
```

Windows (PowerShell):
```powershell
irm https://app.bluebox.ai/install.ps1 | iex
```

Windows (Command Prompt):
```cmd
curl -fsSL https://app.bluebox.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

**Windows native install is experimental.** Per docs.bluebox.ai/cli: "The installer supports macOS, Linux, and Windows natively (experimental) or via WSL2." WSL2 remains a fully supported install path for Windows, not a deprecated fallback — recommend it over the native installer for anything customer-facing until native support graduates out of experimental. Binary installs to `%USERPROFILE%\.bluebox\bin\bluebox.exe` and the directory is added permanently to the user PATH. On macOS/Linux, binary installs to `~/.bluebox/bin/bluebox`.

Running the installer interactively in a terminal auto-starts `bluebox setup`.

**Step 2: Select a GitHub repository.** **Bluebox currently supports exactly one repository per workspace.** This is a hard current limitation, not a plan tier. Flag it clearly if a customer expects multi-repo support.

**Step 3: Instrument services.** Bluebox needs OpenTelemetry traces from server-side services. The recommended path is agent-driven:
```text
Add OpenTelemetry to this project. Use the Bluebox instrumentation skill
```
This covers application traces as a baseline; metrics and logs can be added on request. Setup writes a `.env.otel.bluebox-template` file to the repo holding exporter settings; `bluebox otlp-endpoint` fetches the actual endpoint. **The ingest token is deliberately excluded from the template and from all CLI output**, it must be copied from the Setup page in the browser, specifically so a coding agent (or a committed file) can never capture it.

**Automatic problem detection** begins once telemetry flows. Bluebox continuously analyzes incoming OTel data for elevated error rates, latency spikes, and repeated failures, surfacing them as Findings.

**Step 4: Run the first investigation.** Open Investigations, describe the issue (or click Investigate on a Finding). Bluebox queries live data, builds an evidence chain, and returns an analysis with a proposed fix, filed as a GitHub Issue.

### Workspaces
- Workspace owners invite teammates by email; invitations expire (1–14 days, default 7).
- Each workspace has a member limit (current members + pending invitations count against it).
- A shared link never grants access by itself; non-members see a **Request access** option instead of an error; owners approve or decline.
- Plans cap how many workspaces a person can own.

---

## CLI Reference (Key Commands)

CLI Reference version 4.3 as of last verification. Re-verify flags at docs.bluebox.ai/cli before quoting specific flags in customer-facing answers.

| Command | Purpose |
|---|---|
| `bluebox version` | Print installed CLI version |
| `bluebox update` | Check for and install newer releases. `--force` reinstalls the current version. |
| `bluebox setup` | Run/re-run onboarding: auth, coding-agent skill install, opens Setup page for repo + ingest token. Auto-detects and installs skills for Claude Code, Kiro, GitHub Copilot, Cursor, and OpenCode. |
| `bluebox auth login` | Device-authorization-flow login; token stored in OS keyring (macOS Keychain, Linux Secret Service) or file fallback. Sign-in tokens expire after **14 days of inactivity**. Flags: `--no-browser`, `--no-keyring`, `--device-label`. |
| `bluebox auth logout` | Clear stored credentials. |
| `bluebox skills install` | Manually install Bluebox skills to `.agents/skills/` or a specified directory. Recognizes per-agent flags for Claude Code, Cursor, Windsurf, GitHub Copilot, Kiro, Codex, or OpenCode; pass the matching flag instead of `--generic` if the target agent uses one of these conventions. |
| `bluebox skills list` | List available skills with name, version, and description. `--wide` prevents truncation of long descriptions. |
| `bluebox skills refresh` | Update installed skills to latest versions. `--force` overwrites locally-edited skill files. |
| `bluebox ask "<question>"` | Query live production data from the terminal; streams answer to stdout, progress to stderr. Every answer includes an **Evidence** section: what was queried, the numbers behind the conclusion, and any coverage gaps stated explicitly. See flags below. |
| `bluebox otlp-endpoint` | Print the workspace's OTLP ingest endpoint (URL only, never the token). |
| `bluebox workspace list` | List workspaces and mark the active one. `--json` for structured output. |
| `bluebox workspace switch <id>` | Switch active workspace. Run without arguments for interactive mode. |

**`bluebox ask` flags:**

| Flag | Purpose |
|---|---|
| `--service <name>` | Service name context |
| `--env <name>` | Deployment environment |
| `--since <window>` | Time window, e.g. `7d`, `24h`, `30m` (defaults to 24h) |
| `--repo-url <url>` | Override git-origin auto-detection |
| `--entity-id <id>` | Pin an exact monitored entity |
| `--conversation-id <uuid>` | Resume a specific conversation thread |
| `--continue` | Resume the most recent conversation |
| `--workspace <id>` | Target a specific workspace for one call without changing the saved default |

**Usage notes:**
- **Git-origin auto-detection:** by default `bluebox ask` detects the target repo from the local git origin; `--repo-url` is only needed when there's no local git origin or it doesn't match the intended repo (e.g., running from CI).
- **Flag ordering:** put flags before the question — a flag written after the question is treated as part of the question text itself.
- **Empty-question validation:** `ask` needs an actual question; running it with nothing, or with an empty or blank one, is refused.
- **Multi-service repos:** when gathering context, the CLI sends the *modules* your changes touch — the directories that hold a project manifest — not a long list of individual files.

**Environment variables:**

| Variable | Purpose |
|---|---|
| `BLUEBOX_TOKEN` | Single-invocation token for CI/scripting; never stored. |
| `BLUEBOX_API_URL` | Override API URL for one invocation. |
| `BLUEBOX_DISABLE_KEYRING` | Set to `1` to force file-based credential storage (useful on headless servers). |
| `BLUEBOX_NO_AGENT` | Set to `1` to use interactive CLI despite agent environment variables being set. |
| `BLUEBOX_ASK_CONNECT_TIMEOUT` | Wait duration for new conversations; capped at 10 minutes (default: 5s). |
| `BLUEBOX_ASK_RESUME_TIMEOUT` | Wait duration for continuing conversations; default 20s, capped at 10m. |

---

## Routines

Routines automate agent executions on a schedule. A routine runs a Bluebox agent exactly as if you had asked the question yourself; common uses include nightly cost reviews, weekly SLO checks, and pre-release health reports.

**Creating a routine:** only a name is required. If a prompt is omitted, the routine name becomes the prompt. Default schedule: daily at 6:00 AM in the browser timezone.

**Schedule options:**

| Type | Detail |
|---|---|
| Once | Single execution at a specified date/time |
| Hourly | Every hour, on the hour |
| Daily | Every day at a chosen time (default) |
| Weekdays | Monday–Friday at a chosen time |
| Weekly | One day per week at a chosen time |
| Custom | Five-field cron expression (e.g., `0 9 1 * *` for 9 AM on the 1st of each month) |

Advanced settings (under "Advanced"): timezone (defaults to browser timezone, maintains local time across DST), start date, end date, max occurrences (cannot be removed once set), and overlap policy (Skip or Allow for concurrent runs).

**Run outcomes:**

| Outcome | Meaning |
|---|---|
| Run started | Fired successfully |
| Skipped (overlap) | Previous run still ongoing; overlap policy is Skip |
| Skipped (stale) | Multiple scheduled times passed; routine skipped ahead |
| Skipped (throttled) | Workspace run limit reached |
| Expired | End date passed; routine stopped permanently |
| Limit reached | Max occurrences hit; routine stopped permanently |

**Resource limits:** every run (scheduled or manual) counts against the workspace's run allowance. An hourly routine consumes 24 allocations per day. When the limit is reached, new fires are recorded as Skipped (throttled). This is separate from the workspace-wide daily AI usage cap (see FAQ), which can stop a run mid-execution and mark it Limit reached rather than skipping it before it starts.

**Key constraints:**
- Schedule type cannot be changed after creation.
- Max occurrences cannot be removed once set (delete and recreate to change it).
- Expired or limit-reached routines cannot be restarted by toggling; edit the schedule or create a new routine.
- Not available in every workspace or plan tier; verify availability before promising it to a customer.

**Visibility:** workspace-wide by default (read-only for non-owners). Owners and workspace administrators can edit, delete, and manage routines.

---

## Connections (Integrations)

Every connection has an **owner** (whoever created it). Connections are private-to-owner by default; other workspace members can view state but cannot reconnect, reauthorize, reconfigure, or delete unless the owner or a workspace owner grants edit access. Members with explicit edit access can modify settings but cannot change sharing permissions.

### AWS DevOps Agent
Lets Bluebox run investigations against an AWS account via registered **AgentSpaces**.
- Setup wizard: **Configure** (name + region) → **Setup IAM** (run a provided bash/Terraform/CloudFormation script with `iam:*` and admin permissions; registers Bluebox as an OIDC provider, creates a scoped-trust IAM role) → **Verify** (paste role ARN; Bluebox runs an STS token-exchange smoke test).
- IAM permission groups granted: Discover (`aidevops:ListAgentSpaces`), Task management (`CreateBacklogTask`, `UpdateBacklogTask`, `GetBacklogTask`, `ListExecutions`), Investigation (`ListJournalRecords`, `SendMessage`, `CreateChat`).
- After activation, **Discover & register** queries available AgentSpaces. Each registered space needs a description; Bluebox uses it to decide whether to involve that space in a given investigation; an undescribed space is never selected. Bluebox can draft the description itself via **Ask AgentSpace**.
- Removing the integration deletes the Bluebox-side connection and registered spaces but **does not** remove the OIDC provider/IAM role in AWS. That cleanup is manual (`aws iam delete-role`, `aws iam delete-open-id-connect-provider`).

### Azure SRE Agent (Preview)
Lets Bluebox run investigations against an Azure subscription. Currently in preview; verify current availability and setup details at docs.bluebox.ai before quoting specifics.

- **Prerequisites:** Azure CLI installed and authenticated; permissions to create app registrations and assign roles.
- Setup wizard: **Configure** (display name + Azure region) → **Setup Entra** (run downloaded script to create app registration with federated identity credential, service principal, and assign Reader role and SRE Agent Standard User role) → **Verify** (paste script output containing Tenant ID, Client ID, and Subscription ID).
- After activation, discover and register SRE Agent resources. Provide descriptions for each registered agent so Bluebox knows when to involve them.

### Kiro
- **Kiro IDE:** `bluebox setup` auto-detects and installs skills. Skills load on demand (not at session start); add a steering file (`bluebox skills install --generic` targeting `.kiro/steering/`, committed to the repo) so every session has Bluebox context automatically.
- **Kiro Web:** four-step setup requiring an AWS account administrator for step 1 (enabling **Autonomous agents** and **Web search/fetch tools** in Kiro settings). Steps 2–4 (add steering text, create a CLI token in Bluebox, add it as a `BLUEBOX_TOKEN` secret in Kiro Web) can be done by anyone. **Kiro Web tokens expire after 30 days.**

---

## Security and Privacy

### What Bluebox reads
- **GitHub:** repo code, file trees, commits, PRs, issues.
- **Observability platform:** traces, logs, metrics, events, service topology, problem records.
- **Workspace:** investigations and chat messages created within it.

**Explicitly does not read:** secrets/env vars/`.env` files in repos, files outside the connected repository, or observability data outside the connected environment.

### What Bluebox stores (and for how long)
| Data | Retention |
|---|---|
| Investigations, timelines, chat conversations | Until workspace deletion |
| Observability runtime token, OTLP ingest token | Until connection is disconnected/deleted |
| GitHub identity link (user ID only, not OAuth token) | Until account deletion |

**Never stored:** the GitHub OAuth token beyond the initial repo-discovery step (discarded after repo selection; investigations use short-lived tokens instead); agent reasoning beyond what's shown in the investigation timeline.

### What Bluebox will never do
Never merges code. Never pushes to branches. Never stores credentials in the repository. Never accesses another workspace's data. Never runs without safety controls limiting what an investigation can access or execute.

### Visibility
Workspace members see their own work plus anything shared; a new investigation is private-to-creator by default unless the workspace sets sharing as default, though investigations Bluebox starts automatically are shared workspace-wide. Chats stay private to the person who started them. Workspace owners can see any investigation in the workspace. Bluebox support staff sees operational metadata only, not content.

### Tokens
- **Runtime token** (queries the observability platform): never re-displayed after saving.
- **OTLP ingest token** (service instrumentation): revealed once via **Reveal token** in Setup, then only its presence is shown. Rotate either by saving a new one on the Setup page.

---

## Workflows

1. **Investigate a Finding**: click Investigate on the Overview page; typical cycle from Finding to a GitHub Issue with a proposed fix is **2–10 minutes**.
2. **Query production before you build**: use `bluebox ask` to ground a feature/refactor in real traffic patterns, error rates, and dependencies before writing code, rather than working from a mental model of the codebase.
3. **Use Bluebox inside a coding agent session**: once `bluebox setup` installs the production-query skill, the agent can call `bluebox ask` on its own when it needs context, without being prompted manually each time.
4. **Turn an investigation into a PR**: hand the GitHub Issue (root cause, evidence, recommended fix) to a coding agent with a prompt like "implement the fix described in this issue." The agent opens a PR; a human reviews before merge.
5. **Automate recurring checks with Routines**: schedule `bluebox ask`-style queries to run on a cron schedule and surface results workspace-wide without manual intervention.

---

## Troubleshooting

### Setup / OTLP endpoint not ready
`bluebox otlp-endpoint` reports Bluebox is still preparing the endpoint, or that a workspace owner needs to finish monitoring setup in the web frontend. Wait and retry, or escalate to a workspace owner.

### CLI not found after install
macOS/Linux:
```bash
export PATH="$HOME/.bluebox/bin:$PATH"
```
Windows: the installer adds `%USERPROFILE%\.bluebox\bin` to the user PATH permanently; open a new terminal to pick it up.

### GitHub OAuth keeps failing
Check for blocked pop-ups or third-party cookies; try an incognito window with extensions disabled.

### Repositories not showing in Setup
Confirm the Bluebox GitHub App is installed on the correct account/organization; the picker prompts an install if the app is missing.

### CLI says the sign-in token expired
CLI tokens expire after 14 days of inactivity; run `bluebox auth login` again.

### Single-repository limitation surfacing as a "missing" second repo
This is current, by design, not a bug. Bluebox supports one repository per workspace. A second repository requires a second workspace.

### Chat replies with an apology instead of an answer
Occasionally the agent gets stuck retrying a step without progress; it now returns a plain message asking for rephrasing rather than failing silently. Narrow the question or add detail and retry.

---

## FAQ Highlights

- **No ML/agent code required**: Bluebox is SaaS; connect GitHub and the observability platform through the web UI.
- **Coding agent support:** Claude Code and Kiro fully supported; GitHub Copilot, Cursor, OpenCode experimental; any other agent needs manual skill installation via `bluebox skills install`.
- **AI usage cap:** every workspace has a daily AI usage limit. If a running task or investigation hits that limit before it finishes, Bluebox stops it and marks it **Limit reached** — distinct from the Routines-specific "Skipped (throttled)" workspace run-allowance limit below.
- **OS support (CLI):** macOS and Linux fully supported. Windows installs natively (experimental) or via WSL2 (fully supported) — WSL2 is still the safer recommendation until native Windows support graduates out of experimental.
- **Regional availability:** new sign-ups can be blocked in some countries for legal/compliance reasons; this check applies only to new sign-ups, not existing accounts.
- **Can it run without an observability platform connected?** Yes for GitHub-only investigation, but diagnosis quality is explicitly stated to be significantly better with live telemetry connected.
- **Session timeout:** signed out after inactivity, with a warning first; active tab interaction (click/type/scroll) keeps the session alive, an idle background tab does not.

---

## Glossary

| Term | Definition |
|------|------------|
| **Bluebox** | Dynatrace's AI SRE agent product (legally Dynatrace LLC per its own Terms of Service); detects production issues, diagnoses root cause from live telemetry, proposes fixes as GitHub Issues. |
| **Finding** | An automatically detected production issue surfaced on the Overview page. |
| **Investigation** | A structured, evidence-backed root-cause analysis Bluebox performs on a Finding or a described symptom. |
| **Routine** | A scheduled Bluebox agent execution that runs on a cron schedule and posts results to the workspace. |
| **AgentSpace** | An AWS DevOps Agent construct that Bluebox can register and route sub-investigations to for AWS-specific root-cause/mitigation input. |
| **Workspace** | The unit of isolation in Bluebox: one connected GitHub repo, its own connections, members, and tokens. |
| **Connection** | A link between Bluebox and an external system (AWS DevOps Agent, Azure SRE Agent, Kiro). Has an owner; other workspace members can view but not reconfigure it unless granted edit access. |
| **bluebox ask** | The primary CLI command for querying live production telemetry from the terminal or from within a coding agent. |

---

## When to Escalate (Bluebox-specific)

- Any question about pricing, plan tiers, or workspace/seat limits → route to Bluebox/Dynatrace sales; this file does not cover commercial terms and they were not published in the verified documentation as of this writing.
- Any question implying deep integration with core Dynatrace Grail/DQL/OneAgent data beyond what's documented here → verify fresh. The documented architecture (own OTLP ingest, own workspace model, own tokens) suggests Bluebox is architecturally separate from the core Dynatrace ingestion/query stack even though it shares Dynatrace's causal-AI reasoning; don't assert a tighter integration than what's confirmed.
- Custom/unsupported coding agent skill installation issues beyond the documented manual `bluebox skills install` flow → likely needs Bluebox support (support@bluebox.ai).
- Routines availability or plan-tier questions → verify at docs.bluebox.ai; not all workspaces or plans include Routines access.
- Azure SRE Agent questions → verify current status at docs.bluebox.ai/connections; it was in preview as of this file's verification date and details are subject to change.
