# Contributing

This repo contains curated Markdown reference files used as context for a Dynatrace Technical Specialist AI agent. Contributions that keep those files accurate and current are the most valuable thing you can do.

## What you can contribute

**Fix an error.** If a file states something that is factually wrong (wrong scope name, wrong API field, wrong behavior), open a pull request with the correction and a link to the live doc that confirms it.

**Update stale content.** If a file's "Last verified" date is older than its staleness window and you've verified the content against current docs, update the affected sections and the `_Last verified_` date at the top of the file.

**Add new coverage.** If a topic is missing entirely, open a New Topic Request issue first to discuss scope before writing. New files follow the same structure and verification standards as existing ones.

## Before you submit a pull request

All content changes must be verified against official documentation:

- **Core Dynatrace:** [docs.dynatrace.com](https://docs.dynatrace.com)
- **Bindplane:** [docs.bindplane.com](https://docs.bindplane.com)
- **Bluebox:** [docs.bluebox.ai](https://docs.bluebox.ai)

Community forum posts are not authoritative. Do not use them as the sole source for a change.

## Staleness windows

When updating a file, respect these freshness expectations:

| File(s) | Window | Why |
|---------|--------|-----|
| Core Dynatrace files | ~90 days | Platform ships on a ~2-week cycle; architectural facts change slowly |
| `dynatrace_bindplane_reference.md` | ~30 days | Acquired April 2026; integration depth with Dynatrace is actively changing |
| `dynatrace_bluebox_reference.md` | ~14 days | Launched weeks before initial verification; CLI and setup steps change frequently |

If you update any file, set `_Last verified_` to today's date in `YYYY-MM-DD` format at the top of the file.

## High-risk content

The following categories have the highest chance of becoming stale or causing harm if wrong:

- **API token scope names** — these change between releases; always verify the exact string against live docs
- **Auth header format** — Classic API (`Api-Token`) and Platform API (`Bearer`) use different formats; confirm which API generation an endpoint belongs to before describing auth
- **Integration depth claims for Bindplane and Bluebox** — roadmap statements are not shipped features; only document confirmed, shipped integrations
- **Pricing and licensing numbers** — do not add these; route licensing questions to Dynatrace sales

## Reporting stale content without contributing a fix

Open a [Stale Content Report](../../issues/new?template=stale_content.yml) issue. Include the file, what it currently says, what the live docs say now, and the source URL where you verified.

## Pull request checklist

Before submitting, confirm:

- [ ] All changes verified against live docs (include the URL in the PR description)
- [ ] `_Last verified_` date updated at the top of every modified file
- [ ] Scope names, API field names, and CLI flags confirmed against live docs
- [ ] No pricing or licensing numbers added
- [ ] No competitor comparisons added
- [ ] Any Bindplane/Bluebox integration depth claims confirmed as shipped features, not roadmap statements
