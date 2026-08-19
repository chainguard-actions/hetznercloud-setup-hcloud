<!-- markdownlint-disable -->

# Hardening Report: hetznercloud--setup-hcloud/v2.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **hetznercloud--setup-hcloud/v2.0.1** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Three run: steps in release.yml directly interpolate GitHub Actions expressions into shell commands. Line 22: `awk ... <<< '${{ github.event.release.tag_name }}'` — the release tag name is attacker-influenced and injected directly into the shell command string. Line 25: `git tag --force "${{ steps.major.outputs.major }}" "${{ github.event.release.tag_name }}"` — both expressions are interpolated directly. Line 28: `git push --force origin "${{ steps.major.outputs.major }}"` — the step output is interpolated directly. All three violate the rule that no ${{ ... }} expression should appear inside a run: shell command string.

Locations:

- `.github/workflows/release.yml:22`
- `.github/workflows/release.yml:25`
- `.github/workflows/release.yml:28`

### github-env-injection (severity: high)

In release.yml line 22, the value of `${{ github.event.release.tag_name }}` (an attacker-controllable release tag) is written directly to $GITHUB_OUTPUT without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A crafted tag name containing newlines could inject arbitrary key=value pairs into the GitHub output environment. The offending line: `run: awk -F '.' '{print "major=" $1}' <<< '${{ github.event.release.tag_name }}' >> "$GITHUB_OUTPUT"`

Locations:

- `.github/workflows/release.yml:22`

### missing-permissions (severity: medium)

The workflow file releaser-pleaser.yml has no top-level `permissions:` key and its only job (`releaser-pleaser`) also has no job-level `permissions:` key. This means the workflow runs with the default GitHub token permissions (read/write on most scopes). This is especially concerning because the workflow is triggered by `pull_request_target`, a high-privilege event that runs in the context of the base repository even for PRs from forks.

Locations:

- `.github/workflows/releaser-pleaser.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed three findings in two workflow files:

1. release.yml (script-injection + github-env-injection): Rewrote all three run: steps that contained ${{ }} expressions. The 'Extract major from version' step now uses an env: block with TAG_NAME=${{ github.event.release.tag_name }}, sanitizes it with `tr -d '\n\r'` before processing, and writes the sanitized major version to GITHUB_OUTPUT using printf. The 'Update major tag' and 'Push updated major tag' steps now use env: blocks (MAJOR and TAG_NAME) and reference plain shell variables instead of ${{ }} expressions.

2. releaser-pleaser.yml (missing-permissions): Added `permissions: {}` at the top level to restrict the default GITHUB_TOKEN permissions. The workflow uses secrets.HCLOUD_BOT_TOKEN (a PAT) for all actual operations via the releaser-pleaser action, so the GITHUB_TOKEN does not need any permissions.

