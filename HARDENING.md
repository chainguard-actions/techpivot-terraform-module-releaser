<!-- markdownlint-disable -->

# Hardening Report: techpivot--terraform-module-releaser/v1.8.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **techpivot--terraform-module-releaser/v1.8.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple run: blocks directly interpolate ${{ ... }} expressions inside shell commands (sub-rule a), allowing script injection.

1. ci.yml "Test Action Outputs" step: ${{ steps.test-action.outputs.changed-module-names }} (and other step outputs) are interpolated directly into shell if/echo/jq commands, e.g.: `if [[ -n "${{ steps.test-action.outputs.changed-module-names }}" ]]; then`

2. release-start.yml "Validate version input" step: ${{ env.VERSION }} (derived from github.event.inputs.release_version, a user-controlled workflow_dispatch input) is interpolated directly into the shell: `if ! [[ '${{ env.VERSION }}' =~ ^[0-9]+(\.[0-9]+){2}$ ]]; then`

3. release-start.yml "Update package.json version" step: ${{ env.VERSION }} is interpolated directly into the shell command: `run: npm version --no-git-tag-version '${{ env.VERSION }}'`

4. release-start.yml "Get GitHub App User Details" step: ${{ steps.app-token.outputs.app-slug }} is interpolated directly into shell variable assignments and a gh api URL: `user_name="${{ steps.app-token.outputs.app-slug }}[bot]"`

5. release.yml "Create and push tag" step: ${{ steps.extract-version.outputs.result }} is interpolated directly into git tag and git push commands: `git tag "v${{ steps.extract-version.outputs.result }}"`

All of these should be moved to env: variables and the env vars should be double-quoted in the shell.

Locations:

- `.github/workflows/ci.yml:63`
- `.github/workflows/release-start.yml:46`
- `.github/workflows/release-start.yml:54`
- `.github/workflows/release-start.yml:98`
- `.github/workflows/release.yml:59`

### github-env-injection (severity: high)

The "Get GitHub App User Details" step in release-start.yml writes values derived from ${{ steps.app-token.outputs.app-slug }} to $GITHUB_OUTPUT without sanitization. The shell variables user_name (set from "${{ steps.app-token.outputs.app-slug }}[bot]") and user_id (from gh api output) are written directly via `echo "user-name=${user_name}" >> "$GITHUB_OUTPUT"` etc. without the required `printf '%s' "$VAR" | tr -d '\n\r'` sanitization step. A newline in the app-slug output could allow injection of arbitrary key=value pairs into GITHUB_OUTPUT.

Locations:

- `.github/workflows/release-start.yml:100`

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable version tags instead of pinned 40-character commit SHAs, making them vulnerable to supply-chain attacks if the tag is moved.

check-dist.yml: actions/checkout@v6, actions/setup-node@v6, actions/upload-artifact@v5
ci.yml: actions/checkout@v6, actions/setup-node@v6
codeql-analysis.yml: actions/checkout@v6, github/codeql-action/init@v4, github/codeql-action/autobuild@v4, github/codeql-action/analyze@v4
lint.yml: actions/checkout@v6, actions/setup-node@v6
release-start.yml: actions/checkout@v6, actions/setup-node@v6, actions/github-script@v8, actions/create-github-app-token@v2, actions/github-script@v8 (x2)
release.yml: actions/checkout@v6, actions/github-script@v8 (x3)
test.yml: actions/checkout@v6, actions/setup-node@v6

Note: super-linter/super-linter, peter-evans/create-pull-request, and SonarSource/sonarqube-scan-action are correctly pinned to full SHA digests.

Locations:

- `.github/workflows/check-dist.yml:18`
- `.github/workflows/ci.yml:23`
- `.github/workflows/codeql-analysis.yml:27`
- `.github/workflows/lint.yml:22`
- `.github/workflows/release-start.yml:27`
- `.github/workflows/release.yml:21`
- `.github/workflows/test.yml:15`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings across 7 workflow files:

1. **unpinned-uses**: Pinned all mutable action tags to full 40-char commit SHAs in check-dist.yml, ci.yml, codeql-analysis.yml, lint.yml, release-start.yml, release.yml, and test.yml. Actions pinned: actions/checkout@v6, actions/setup-node@v6, actions/upload-artifact@v5, github/codeql-action/{init,autobuild,analyze}@v4, actions/github-script@v8, actions/create-github-app-token@v2.

2. **script-injection**: Moved all ${{ }} expressions out of run: shell blocks into env: variables. In ci.yml, all 6 step output references are now in env: block. In release-start.yml, VERSION and app-slug references are moved to env: vars (INPUT_VERSION, APP_SLUG, APP_USER_NAME). In release.yml, the version output is moved to RELEASE_VERSION env var. For github-script steps, values are accessed via process.env instead of direct interpolation.

3. **github-env-injection**: The 'Get GitHub App User Details' step in release-start.yml now sanitizes all values written to GITHUB_OUTPUT using `printf '%s' "$VAR" | tr -d '\n\r'` before writing user-name, user-id, and email. The gh api URL also uses URL-encoded brackets (%5Bbot%5D) to avoid shell injection from the APP_SLUG variable.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in .github/workflows/release.yml at the 'Create GitHub Release via API' step. Moved `${{ toJSON(steps.extract-release-notes.outputs.result) }}` out of the script: block and into the env: block as `RELEASE_NOTES: ${{ steps.extract-release-notes.outputs.result }}`. Updated the script to read the value safely via `process.env.RELEASE_NOTES` instead of direct expression interpolation.

