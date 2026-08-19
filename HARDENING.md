<!-- markdownlint-disable -->

# Hardening Report: hetznercloud--setup-hcloud/v1.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **hetznercloud--setup-hcloud/v1.0.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable tag refs instead of full 40-character SHA commit hashes. This exposes the workflow to supply-chain attacks if the referenced tag is moved or the action is compromised.

ci.yml: actions/checkout@v4 (lines 17, 36, 56), actions/setup-node@v4 (lines 19, 38)
release-please.yml: google-github-actions/release-please-action@v4 (line 14)
release.yml: actions/checkout@v4 (line 15)

Locations:

- `.github/workflows/ci.yml:17`
- `.github/workflows/ci.yml:19`
- `.github/workflows/ci.yml:36`
- `.github/workflows/ci.yml:38`
- `.github/workflows/ci.yml:56`
- `.github/workflows/release-please.yml:14`
- `.github/workflows/release.yml:15`

### script-injection (severity: high)

release.yml has ${{ }} expressions directly interpolated inside run: shell command strings, violating rule (a). An attacker who can influence the release tag name or step outputs can inject arbitrary shell commands.

Line 22: `run: awk -F '.' '{print "major=" $1}' <<< '${{ github.event.release.tag_name }}' >> "$GITHUB_OUTPUT"` — github.event.release.tag_name is interpolated directly into the shell command.
Line 25: `run: git tag --force "${{ steps.major.outputs.major }}" "${{ github.event.release.tag_name }}"` — both expressions are interpolated directly.
Line 28: `run: git push --force origin "${{ steps.major.outputs.major }}"` — steps output interpolated directly.

Locations:

- `.github/workflows/release.yml:22`
- `.github/workflows/release.yml:25`
- `.github/workflows/release.yml:28`

### github-env-injection (severity: high)

release.yml line 22 writes the value of ${{ github.event.release.tag_name }} directly to $GITHUB_OUTPUT without sanitization (no `printf '%s' ... | tr -d '\n\r'` step). A tag name containing newlines could inject arbitrary key=value pairs into the output context.

Offending line: `run: awk -F '.' '{print "major=" $1}' <<< '${{ github.event.release.tag_name }}' >> "$GITHUB_OUTPUT"`

Locations:

- `.github/workflows/release.yml:22`

### missing-permissions (severity: medium)

release-please.yml has no top-level `permissions:` key and no job-level `permissions:` key on the `release-please` job. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be broader than necessary (e.g., write access to contents, pull-requests, etc.).

Locations:

- `.github/workflows/release-please.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all four findings across three workflow files:

1. ci.yml: Pinned actions/checkout@v4 (3 occurrences) and actions/setup-node@v4 (2 occurrences) to full SHAs.

2. release-please.yml: Pinned google-github-actions/release-please-action@v4 to full SHA; added top-level `permissions: {}` and job-level `permissions: { contents: write, pull-requests: write }` (minimum needed for release-please to create PRs and releases).

3. release.yml: Pinned actions/checkout@v4 to full SHA; fixed all three script-injection instances by moving ${{ github.event.release.tag_name }} and ${{ steps.major.outputs.major }} into env: blocks; fixed github-env-injection by sanitizing the tag name with `printf '%s' "$TAG_NAME" | tr -d '\n\r'` before writing to GITHUB_OUTPUT.

