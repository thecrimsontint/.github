# AGENTS.md — thecrimsontint/.github

Canonical agent instructions for this repo (`CLAUDE.md` only imports this file).

**This repository is PUBLIC.** Everything committed here is world-readable. Never add internal
hostnames, IP addresses, internal URLs, secret-store paths, tailnet names, usernames, or any other
infrastructure detail that is not already present. Refer to private systems generically ("the in-cluster
Gitea", "the private homelab-docs vault", "the Homebox inventory").

## What this repo is

The org-level `.github` repository for the `thecrimsontint` GitHub org. It holds **org-wide reusable
GitHub Actions workflows** that the other (private) org repos call. Today that is one workflow:

| Path | Purpose |
|---|---|
| `.github/workflows/gitea-mirror-sync.yml` | Reusable (`on: workflow_call`). Asks the in-cluster Gitea to pull its mirror of the calling repo immediately instead of waiting for the periodic pull. Runs only on pushes to the caller's default branch, on the org self-hosted runner set `thecrimsontint-arc`, using the `GITEA_MIRROR_TOKEN` secret inherited from the caller (`secrets: inherit`). Tries the org owner first, then a legacy owner, and fails if neither mirror exists. |
| `README.md` | One-line description. |

Self-hosted Renovate (config + scheduled workflow) used to live here; it moved to the private
`thecrimsontint/renovate` repo because the org's self-hosted runner group does not serve public repos.
Do not move it back.

Because this is the org's `.github` repo, community-health files placed here (e.g.
`PULL_REQUEST_TEMPLATE.md`, `CONTRIBUTING.md`, `SECURITY.md`, issue templates, at the root or under
`.github/`) become **defaults for every org repo** that lacks its own. Adding one is an org-wide change;
none exist today.

## How callers use it

Each mirrored repo carries a small caller, pinned to a full commit SHA with a `# main` comment:

```yaml
name: Sync Gitea mirror
on:
  push:
    branches: [main] # the caller's default branch; no runs (and no check-runs) on PR branches
  workflow_dispatch:
jobs:
  mirror-sync:
    uses: thecrimsontint/.github/.github/workflows/gitea-mirror-sync.yml@<40-hex sha> # main
    secrets: inherit
```

Renovate (`helpers:pinGitHubActionDigests`) bumps that SHA in **every** org repo after a change merges
here (`chore(deps): update thecrimsontint/.github digest to <sha>`). So:

- A merge here fans out into one Renovate PR per repo; those automerge when green. A broken workflow
  therefore breaks mirror-sync org-wide within a Renovate cycle — test before merging.
- Keep the `workflow_call` interface backward compatible (no new required inputs/secrets) or every caller
  breaks at once.
- A reusable workflow called from a private repo runs in the caller's context (caller's runners and
  secrets); this repo's own jobs could NOT use the self-hosted runner group.

## Workflow design rules (learned the hard way)

- **Default-branch pushes only, never cancelled.** Feature and Renovate branches used to trigger runs
  that cancelled each other under `cancel-in-progress`; a CANCELLED or failed check blocks automerge on
  every PR, whereas a skipped job counts as passing. Keep the job-level `if:` on the default branch and
  `cancel-in-progress: false`, AND filter the caller's trigger to its default branch (`on.push.branches`):
  two pushes of the same SHA seconds apart (a Renovate rebase) queue two runs in one concurrency group
  and GitHub cancels the second before the job `if:` is evaluated, leaving a `cancelled` check-run on the
  PR head. The group name carries a `-v2` suffix so an older revision of this workflow can never share
  (and cancel in) the group.
- Pin third-party actions by full commit SHA with a version comment (Renovate maintains them).
- Use `set -euo pipefail` in `run:` steps and keep secrets in `env:`, never inline in commands or logs.
- Validate workflow syntax before pushing, e.g. `actionlint .github/workflows/*.yml` (if installed) or
  `python3 -c 'import yaml,sys; [yaml.safe_load(open(f)) for f in sys.argv[1:]]' .github/workflows/*.yml`.

## Conventions

- Conventional Commits, short imperative subject: `fix: ...`, `feat: ...`, `chore: ...`, `ci: ...`,
  `docs: ...`. One concern per PR; squash-merge with branch deletion.
- Renovate policy for the org: take every update to latest, majors included, except documented holds
  (those live in the private renovate repo).
- Never hand-build or push container images; images ship only via the org's IaC + CI pipelines.
- There is no CI on PRs here beyond what a change itself adds; review the diff for leaked internal
  details before merging (see the PUBLIC warning above).

## Documentation and inventory

Standing rule (part of the definition of done): any change made via this repo — or any live change it
relates to — that alters anything documented in the private homelab-docs vault or the Homebox inventory
must include the commensurate update to those in the same piece of work (for example, changing how
mirror-sync works or which runners it uses). Make those updates in the private vault/inventory, never by
adding details to this public repo.
