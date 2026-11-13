# goxboard-builder

Public GitHub Actions orchestration for the private `yuyuzheng-dev/goxboard` project.

This repository contains CI workflow definitions only. It must never contain a checkout, copy, archive, generated bundle, Docker image, or other artifact containing the private source.

## How it works

- `poll-private.yml` runs on a five-minute schedule and on manual dispatch.
- It checks all open same-repository PRs targeting `main` and scans `main`, `fix/**`, `feat/**`, `refactor/**`, and `ci/**` branch tips whose commit timestamp is within the last 15 minutes. This detects recent pushes without building every stale historical branch on first setup.
- It deduplicates work by commit SHA using the `goxboard-builder/ci` commit status.
- New commits are dispatched to `build-private.yml`.
- The build workflow runs Go, Web, and Docker jobs on standard GitHub-hosted Ubuntu runners and writes one aggregate status back to the private commit.
- No build artifacts or caches are uploaded. Checkout credentials are not persisted.

## Required secret

Create a fine-grained personal access token and add it to this repository as:

- Secret name: `GOXBOARD_ACCESS_TOKEN`
- Repository access: only `yuyuzheng-dev/goxboard`
- Repository permissions:
  - Contents: Read-only
  - Pull requests: Read-only
  - Commit statuses: Read and write

Add the secret at:
https://github.com/yuyuzheng-dev/goxboard-builder/settings/secrets/actions/new

After adding the secret, open the Actions tab and run **Poll private goxboard** with `ref` set to the PR branch or exact commit SHA and `force=true` to test that revision once. For PR #25, the current head is `6f638008be0ec4161b0adbb96721e00da8439877`. Leave `ref` empty to scan all eligible branches and open same-repository PRs.

## Security and visibility

This repository is public. Workflow definitions, run metadata, and logs are public. Although the private checkout is not committed or uploaded, build/test failure output can reveal paths, identifiers, snippets, or other information printed by the source project. Do not use this design if build logs must be confidential. Do not print environment variables, and do not upload artifacts or caches containing project files.

The build jobs execute code from the selected private commit. The source token is supplied only to the checkout action with `persist-credentials: false`; it is not exposed as an environment variable to Go, pnpm, Docker, or project test commands. The aggregate status job runs on a separate runner after the build jobs finish.

## GitHub Actions policy

Standard GitHub-hosted runners are free for public repositories under GitHub's published billing rules. This builder is intended solely to develop and test the associated goxboard software project. Public-runner billing treatment does not override GitHub's Terms, Actions limits, or abuse policies. If GitHub considers this cross-repository arrangement outside the permitted use, stop using it and move CI to private-repository hosted runners or a compliant self-hosted setup.

## Scope and limitations

- Same-repository PRs are discovered automatically. PRs whose head branch is in a fork are skipped.
- Polling is best-effort and may be delayed by GitHub's scheduled-workflow queue.
- A completed success/failure status is not rerun for the same SHA unless `force=true` is used.
- A pending status older than 45 minutes is eligible for retry.
