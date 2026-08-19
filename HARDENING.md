<!-- markdownlint-disable -->

# Hardening Report: hetznercloud--setup-hcloud/v2.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **hetznercloud--setup-hcloud/v2.0.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable tags instead of pinned full 40-character SHA digests, making them vulnerable to supply-chain attacks if the tag is moved.

In .github/workflows/ci.yml:
- `uses: actions/checkout@v6` (lines 17, 37, 62)
- `uses: actions/setup-node@v6` (lines 19, 39)
- `uses: codecov/codecov-action@v6` (line 46)

In .github/workflows/release.yml:
- `uses: actions/checkout@v6` (line 15)

In .github/workflows/releaser-pleaser.yml:
- `uses: apricote/releaser-pleaser@v0.8.0` (line 21)

All of these should be pinned to a full commit SHA, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/ci.yml:17`
- `.github/workflows/ci.yml:19`
- `.github/workflows/ci.yml:37`
- `.github/workflows/ci.yml:39`
- `.github/workflows/ci.yml:46`
- `.github/workflows/ci.yml:62`
- `.github/workflows/release.yml:15`
- `.github/workflows/releaser-pleaser.yml:21`

### script-injection (severity: high)

release.yml contains three `run:` steps that directly interpolate `${{ }}` expressions into shell commands (rule a). This allows an attacker who can influence the GitHub event data to inject arbitrary shell commands.

Line 22: `run: awk -F '.' '{print "major=" $1}' <<< '${{ github.event.release.tag_name }}' >> "$GITHUB_OUTPUT"` — `github.event.release.tag_name` is interpolated directly into the shell command string.

Line 25: `run: git tag --force "${{ steps.major.outputs.major }}" "${{ github.event.release.tag_name }}"` — both `steps.major.outputs.major` and `github.event.release.tag_name` are interpolated directly.

Line 28: `run: git push --force origin "${{ steps.major.outputs.major }}"` — `steps.major.outputs.major` is interpolated directly.

All values should be passed via `env:` variables and then referenced as quoted shell variables (e.g. `"$TAG_NAME"`) instead of being interpolated directly.

Locations:

- `.github/workflows/release.yml:22`
- `.github/workflows/release.yml:25`
- `.github/workflows/release.yml:28`

### github-env-injection (severity: high)

In release.yml line 22, the untrusted value `${{ github.event.release.tag_name }}` is interpolated directly into a `run:` shell command and its output is written to `$GITHUB_OUTPUT` without sanitization:

`run: awk -F '.' '{print "major=" $1}' <<< '${{ github.event.release.tag_name }}' >> "$GITHUB_OUTPUT"`

A crafted tag name containing newline characters could inject arbitrary key-value pairs into `$GITHUB_OUTPUT`, potentially overwriting other outputs and influencing downstream steps. The value must be sanitized with `printf '%s' "$TAG_NAME" | tr -d '\n\r'` before being written to the special environment file.

Locations:

- `.github/workflows/release.yml:22`

### missing-permissions (severity: medium)

releaser-pleaser.yml has no top-level `permissions:` key and the single job `releaser-pleaser` also has no `permissions:` key. This means the workflow runs with the default (broad) token permissions. This is especially risky because the workflow is triggered by `pull_request_target`, which runs with write access to the base repository and access to secrets. A minimal explicit `permissions:` block (e.g. `permissions: contents: write` or `permissions: {}`) should be added.

Locations:

- `.github/workflows/releaser-pleaser.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all four findings:

1. **unpinned-uses**: Pinned all action references to full SHAs:
   - `actions/checkout@v6` → `@d23441a48e516b6c34aea4fa41551a30e30af803 # v6` (ci.yml lines 17, 37, 62; release.yml line 15)
   - `actions/setup-node@v6` → `@249970729cb0ef3589644e2896645e5dc5ba9c38 # v6` (ci.yml lines 19, 39)
   - `codecov/codecov-action@v6` → `@fb8b3582c8e4def4969c97caa2f19720cb33a72f # v6` (ci.yml line 46)
   - `apricote/releaser-pleaser@v0.8.0` → `@a1ce9493fd3f3abe60f22c37249d257bc10081dc # v0.8.0` (releaser-pleaser.yml line 21)

2. **script-injection**: In release.yml, moved all `${{ }}` expressions out of `run:` shell strings into `env:` blocks and referenced them as quoted shell variables (`"$TAG_NAME"`, `"$MAJOR"`).

3. **github-env-injection**: In release.yml, sanitized the tag_name value with `printf '%s' "$TAG_NAME" | tr -d '\n\r'` before processing and writing to `$GITHUB_OUTPUT`.

4. **missing-permissions**: Added explicit `permissions: contents: write` and `pull-requests: write` top-level block to releaser-pleaser.yml (minimum permissions needed for the releaser-pleaser action to function).

