# STANDARD Public Runbooks

On-call reference material for STANDARD platform alerts. Mirrored from a private repo on every push to `main`; source files are sanitized to strip internal infrastructure detail before publish.

For incident triage from a Slack alert link, follow the anchor in the URL to the relevant section of `OPERATIONAL_RUNBOOK.md`.

## How this repo stays in sync

A GitHub Actions workflow in the private source repo (`iowire/standard`) runs `scripts/runbook-mirror/sanitize.sed` over `docs/operations/*.md` and pushes the result here. Triggers:

- Push to `main` in the private repo affecting `docs/operations/**`
- Manual via `workflow_dispatch`
- Weekly cron heartbeat (Monday 09:00 UTC)

If sync fails (broken deploy key, sanitizer drift detected by the workflow's audit step), pages here continue to resolve to the last-successful content — they do not 404. The failure mode is staleness, not loss.

## Files

- `OPERATIONAL_RUNBOOK.md` — primary on-call reference, anchored by alert ID (M1, S3, A1-ext, etc.)
- `QUICK_REFERENCE.md` — operator quick-reference
- `SLO.md` — service-level objective definitions

## Status

[Mirror workflow runs](https://github.com/iowire/standard/actions/workflows/mirror-runbooks.yml) — visible only to operators with private-repo access.
