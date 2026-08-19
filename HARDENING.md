<!-- markdownlint-disable -->

# Hardening Report: techpivot--terraform-module-releaser/v1.8.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **techpivot--terraform-module-releaser/v1.8.1** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files use action references pinned to mutable version tags instead of immutable 40-character commit SHAs, making them vulnerable to supply-chain attacks if the tag is moved.

check-dist.yml: actions/checkout@v6 (line 24), actions/setup-node@v6 (line 30), actions/upload-artifact@v5 (line 57)

ci.yml: actions/checkout@v6 (line 23), actions/setup-node@v6 (line 28)

codeql-analysis.yml: actions/checkout@v6 (line 31), github/codeql-action/init@v4 (line 36), github/codeql-action/autobuild@v4 (line 41), github/codeql-action/analyze@v4 (line 45)

lint.yml: actions/checkout@v6 (line 23), actions/setup-node@v6 (line 29), actions/checkout@v6 (line 44)

release-start.yml: actions/checkout@v6 (line 26), actions/setup-node@v6 (line 32), actions/github-script@v8 (line 67), actions/create-github-app-token@v2 (line 86), actions/github-script@v8 (line 103)

release.yml: actions/checkout@v6 (line 20), actions/github-script@v8 (line 27), actions/github-script@v8 (line 43), actions/github-script@v8 (line 57)

test.yml: actions/checkout@v6 (line 19), actions/setup-node@v6 (line 25)

Locations:

- `.github/workflows/check-dist.yml:24`
- `.github/workflows/check-dist.yml:30`
- `.github/workflows/check-dist.yml:57`
- `.github/workflows/ci.yml:23`
- `.github/workflows/ci.yml:28`
- `.github/workflows/codeql-analysis.yml:31`
- `.github/workflows/codeql-analysis.yml:36`
- `.github/workflows/codeql-analysis.yml:41`
- `.github/workflows/codeql-analysis.yml:45`
- `.github/workflows/lint.yml:23`
- `.github/workflows/lint.yml:29`
- `.github/workflows/lint.yml:44`
- `.github/workflows/release-start.yml:26`
- `.github/workflows/release-start.yml:32`
- `.github/workflows/release-start.yml:67`
- `.github/workflows/release-start.yml:86`
- `.github/workflows/release-start.yml:103`
- `.github/workflows/release.yml:20`
- `.github/workflows/release.yml:27`
- `.github/workflows/release.yml:43`
- `.github/workflows/release.yml:57`
- `.github/workflows/test.yml:19`
- `.github/workflows/test.yml:25`

### script-injection (severity: high)

Multiple run: blocks interpolate ${{ }} expressions directly into shell commands, allowing expression values to be interpreted as shell code before the shell ever sees them.

(a) ci.yml — 'Test Action Outputs' step (line 57): ${{ steps.test-action.outputs.changed-module-names }}, ${{ steps.test-action.outputs.changed-module-paths }}, ${{ steps.test-action.outputs.changed-modules-map }}, ${{ steps.test-action.outputs.all-module-names }}, ${{ steps.test-action.outputs.all-module-paths }}, ${{ steps.test-action.outputs.all-modules-map }} are all interpolated directly inside the run: shell script. These are steps.*.outputs.* context values that flow through YAML template substitution before the shell quotes them.

(a) release-start.yml — 'Validate version input' step (line 41): `if ! [[ '${{ env.VERSION }}' =~ ... ]]` and `echo "... Got: ${{ env.VERSION }}"` interpolate env.VERSION (sourced from workflow_dispatch input) directly into the shell. Single-quoting does not prevent injection when the value itself contains a single quote.

(a) release-start.yml — 'Update package.json version' step (line 49): `run: npm version --no-git-tag-version '${{ env.VERSION }}'` interpolates env.VERSION (a workflow_dispatch input) directly into the shell command.

(a) release-start.yml — 'Get GitHub App User Details' step (line 92): `user_name="${{ steps.app-token.outputs.app-slug }}[bot]"` and `gh api "/users/${{ steps.app-token.outputs.app-slug }}[bot]"` interpolate steps.*.outputs.* values directly into the shell.

(a) release.yml — 'Create and push tag' step (line 55): `git tag "v${{ steps.extract-version.outputs.result }}"` and `git push origin v${{ steps.extract-version.outputs.result }}` interpolate steps.*.outputs.* values directly into shell commands.

Locations:

- `.github/workflows/ci.yml:57`
- `.github/workflows/release-start.yml:41`
- `.github/workflows/release-start.yml:49`
- `.github/workflows/release-start.yml:92`
- `.github/workflows/release.yml:55`

### github-env-injection (severity: high)

In release-start.yml, the 'Get GitHub App User Details' step (line 91) writes values derived from steps.app-token.outputs.app-slug (a steps.*.outputs.* context value) to $GITHUB_OUTPUT without the required sanitization step (printf '%s' ... | tr -d '\n\r'). Specifically:

  user_name="${{ steps.app-token.outputs.app-slug }}[bot]"
  user_id=$(gh api "/users/${{ steps.app-token.outputs.app-slug }}[bot]" --jq .id)
  {
    echo "user-name=${user_name}"
    echo "user-id=${user_id}"
    echo "email=${user_id}+${user_name}@users.noreply.github.com"
  } >> "$GITHUB_OUTPUT"

If the app-slug value contains newline characters, an attacker could inject arbitrary key=value pairs into $GITHUB_OUTPUT, potentially overwriting subsequent step outputs and influencing downstream workflow behavior.

Locations:

- `.github/workflows/release-start.yml:91`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all 23 unpinned action references across check-dist.yml, ci.yml, codeql-analysis.yml, lint.yml, release-start.yml, release.yml, and test.yml by pinning to full commit SHAs. Fixed script injection in ci.yml (Test Action Outputs step), release-start.yml (Validate version input, Update package.json version, Get GitHub App User Details steps), and release.yml (Create and push tag step) by moving all ${{ }} expressions into env: blocks. Fixed github-env-injection in release-start.yml by sanitizing the app-slug value with printf/tr before writing to GITHUB_OUTPUT.

