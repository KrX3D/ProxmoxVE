# Repo conventions for driving PRs here

This repo is a small, single-maintainer collection of Proxmox VE helper
scripts (currently `tools/pve/host-backup.sh`). There is no build step and
no test suite — validation is static analysis plus manual exercise of
changed logic.

## Before pushing any shell script change

- `bash -n <file>.sh` must pass (syntax).
- `shellcheck -x --severity=error <file>.sh` must pass — this is what CI
  gates on (`.github/workflows/lint.yml`). Warnings and info-level findings
  are not build-breaking; fix the ones related to your change, but don't
  feel obligated to clear out unrelated pre-existing ones in an unrelated
  PR.
- When practical, exercise the changed code path locally (e.g. mock
  `whiptail`/`crontab` with stub scripts on `PATH` and run the script
  end-to-end) rather than relying on static analysis alone. Note what you
  verified in the PR description.

## CI and approvals

- CI here is `Lint Shell Scripts` only (bash syntax + shellcheck errors).
  There is no separate Claude Approvals check configured, so a PR is done
  once CI is green and there's no merge conflict — it does not wait on an
  approvals gate that doesn't exist.
- If CI is red, root-cause it against the diff before assuming flake; these
  jobs are deterministic (no network calls, no external services), so a
  failure here is almost always real.

## Merge conflicts

- Single-maintainer repo, so conflicts should be rare. A normal merge of
  the base branch into the PR branch is fine; there's no house convention
  requiring rebase.
