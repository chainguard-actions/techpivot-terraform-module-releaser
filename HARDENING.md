<!-- markdownlint-disable -->

# Hardening Report: techpivot--terraform-module-releaser/v1.7.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **techpivot--terraform-module-releaser/v1.7.1** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable version tags (e.g. @v4, @v5, @v3, @v7, @v2) instead of immutable full 40-character commit SHAs. This exposes the workflow to supply-chain attacks if the tag is moved to a malicious commit.

Failing references:
- check-dist.yml: actions/checkout@v5, actions/setup-node@v4, actions/upload-artifact@v4
- ci.yml: actions/checkout@v5, actions/setup-node@v4
- codeql-analysis.yml: actions/checkout@v5, github/codeql-action/init@v3, github/codeql-action/autobuild@v3, github/codeql-action/analyze@v3
- lint.yml: actions/checkout@v5, actions/setup-node@v4
- release-start.yml: actions/checkout@v5, actions/setup-node@v4, actions/github-script@v7 (×2), actions/create-github-app-token@v2
- release.yml: actions/checkout@v5, actions/github-script@v7 (×3)
- test.yml: actions/checkout@v5, actions/setup-node@v4

Locations:

- `.github/workflows/check-dist.yml:19`
- `.github/workflows/check-dist.yml:25`
- `.github/workflows/check-dist.yml:56`
- `.github/workflows/ci.yml:21`
- `.github/workflows/ci.yml:27`
- `.github/workflows/codeql-analysis.yml:24`
- `.github/workflows/codeql-analysis.yml:30`
- `.github/workflows/codeql-analysis.yml:35`
- `.github/workflows/codeql-analysis.yml:40`
- `.github/workflows/lint.yml:19`
- `.github/workflows/lint.yml:25`
- `.github/workflows/release-start.yml:24`
- `.github/workflows/release-start.yml:30`
- `.github/workflows/release-start.yml:63`
- `.github/workflows/release-start.yml:79`
- `.github/workflows/release-start.yml:86`
- `.github/workflows/release.yml:20`
- `.github/workflows/release.yml:25`
- `.github/workflows/release.yml:36`
- `.github/workflows/release.yml:50`
- `.github/workflows/test.yml:18`
- `.github/workflows/test.yml:23`

### script-injection (severity: high)

Multiple run: blocks directly interpolate ${{ }} expressions into shell commands, violating rule (a). This allows an attacker to inject arbitrary shell commands by controlling the expression value.

1. ci.yml — 'Test Action Outputs' step: `${{ steps.test-action.outputs.changed-module-names }}`, `${{ steps.test-action.outputs.changed-module-paths }}`, `${{ steps.test-action.outputs.changed-modules-map }}`, `${{ steps.test-action.outputs.all-module-names }}`, `${{ steps.test-action.outputs.all-module-paths }}`, `${{ steps.test-action.outputs.all-modules-map }}` are all interpolated directly into shell echo/if/jq commands. These outputs come from the action itself and could contain shell metacharacters.

2. release-start.yml — 'Validate version input' step: `if ! [[ '${{ env.VERSION }}' =~ ... ]]` — ${{ env.VERSION }} (sourced from workflow_dispatch input) is interpolated directly into the shell command. Even though it is single-quoted, the expression is substituted before the shell sees it, so a value containing a single quote can break out.

3. release-start.yml — 'Update package.json version' step: `run: npm version --no-git-tag-version '${{ env.VERSION }}'` — same issue as above.

4. release-start.yml — 'Get GitHub App User Details' step: `user_name="${{ steps.app-token.outputs.app-slug }}[bot]"` and `gh api "/users/${{ steps.app-token.outputs.app-slug }}[bot]"` — steps output interpolated directly into shell.

5. release.yml — 'Create and push tag' step: `git tag "v${{ steps.extract-version.outputs.result }}"` and `git push origin v${{ steps.extract-version.outputs.result }}` — step output (derived from PR body) interpolated directly into shell commands.

Locations:

- `.github/workflows/ci.yml:62`
- `.github/workflows/release-start.yml:41`
- `.github/workflows/release-start.yml:47`
- `.github/workflows/release-start.yml:93`
- `.github/workflows/release.yml:55`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Fixed all unpinned action references across 7 workflow files by pinning to full 40-character commit SHAs (preserving version tags as comments). Fixed script injection in ci.yml by moving all 6 steps.test-action.outputs.* expressions into an env: block and referencing them as $VAR_NAME in the shell. Fixed script injection in release-start.yml by moving ${{ env.VERSION }} into INPUT_VERSION env var for the validate/update steps, and ${{ steps.app-token.outputs.app-slug }} into APP_SLUG env var for the app-user step. Fixed script injection in release.yml by moving ${{ steps.extract-version.outputs.result }} into RELEASE_VERSION env var for the tag creation step. Note: ${{ }} expressions remaining in with:/env:/if: blocks and in actions/github-script script: blocks (JavaScript context, not shell) are not shell injection vectors and were left as-is.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed the 'Get GitHub App User Details' step in .github/workflows/release-start.yml. Added sanitization for all three values written to $GITHUB_OUTPUT: safe_user_name, safe_user_id, and safe_email are each computed using `printf '%s' ... | tr -d '\n\r'` to strip embedded newlines/carriage returns before being written to $GITHUB_OUTPUT, preventing newline injection attacks.

