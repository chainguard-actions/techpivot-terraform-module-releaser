<!-- markdownlint-disable -->

# Hardening Report: techpivot--terraform-module-releaser/v2.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **techpivot--terraform-module-releaser/v2.0.0** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The 'Test Action Outputs' run: block in ci.yml directly interpolates ${{ steps.test-action.outputs.* }} expressions (steps.*.outputs.* context) into shell commands without routing through env: variables. For example: `if [[ -n "${{ steps.test-action.outputs.changed-module-names }}" ]]` and `echo '${{ steps.test-action.outputs.changed-modules-map }}' | jq ...`. Any attacker-controlled value in these outputs could inject shell metacharacters.

Locations:

- `.github/workflows/ci.yml:57`

### script-injection (severity: high)

Sub-rule (a): The 'Validate version input' and 'Update package.json version' run: blocks in release-start.yml directly interpolate ${{ env.VERSION }} (which is set from the user-controlled github.event.inputs.release_version workflow_dispatch input) into shell commands. Examples: `if ! [[ '${{ env.VERSION }}' =~ ^[0-9]+(\.[0-9]+){2}$ ]]` and `run: npm version --no-git-tag-version '${{ env.VERSION }}'`. Although single-quoted, the expression is expanded by the YAML template engine before the shell sees it, allowing injection of single-quote characters or other metacharacters.

Locations:

- `.github/workflows/release-start.yml:38`
- `.github/workflows/release-start.yml:47`

### script-injection (severity: high)

Sub-rule (a): The 'Get GitHub App User Details' run: block in release-start.yml directly interpolates ${{ steps.app-token.outputs.app-slug }} into shell commands: `user_name="${{ steps.app-token.outputs.app-slug }}[bot]"` and `user_id=$(gh api "/users/${{ steps.app-token.outputs.app-slug }}[bot]" --jq .id)`. The steps.*.outputs.* context is workflow-controllable and must not be interpolated directly into run: scripts.

Locations:

- `.github/workflows/release-start.yml:88`

### script-injection (severity: high)

Sub-rule (a): The 'Create and push tag' run: block in release.yml directly interpolates ${{ steps.extract-version.outputs.result }} into shell commands: `git tag "v${{ steps.extract-version.outputs.result }}"` and `git push origin v${{ steps.extract-version.outputs.result }}`. The steps.*.outputs.* context is workflow-controllable and must not be interpolated directly into run: scripts.

Locations:

- `.github/workflows/release.yml:55`

### github-env-injection (severity: high)

The 'Get GitHub App User Details' step in release-start.yml writes values derived from ${{ steps.app-token.outputs.app-slug }} to $GITHUB_OUTPUT without sanitization. The shell variable `user_name` is set directly from the ${{ steps.app-token.outputs.app-slug }} expression and then written via `echo "user-name=${user_name}" >> "$GITHUB_OUTPUT"` (and similarly for user-id and email). No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write. A newline character in the app-slug value could inject additional GITHUB_OUTPUT entries.

Locations:

- `.github/workflows/release-start.yml:88`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all 5 findings across 3 workflow files:

1. ci.yml: Moved all steps.test-action.outputs.* expressions from inline shell interpolation into an env: block on the 'Test Action Outputs' step. Shell script now uses plain $VAR references.

2. release-start.yml (VERSION injection): Added env: INPUT_VERSION: ${{ env.VERSION }} to 'Validate version input' and 'Update package.json version' steps. Replaced single-quoted ${{ env.VERSION }} interpolations with double-quoted $INPUT_VERSION env var references.

3. release-start.yml (app-slug injection + github-env-injection): Added APP_SLUG: ${{ steps.app-token.outputs.app-slug }} to env: block of 'Get GitHub App User Details'. Shell now uses ${APP_SLUG} instead of inline ${{ }} expressions. Added printf '%s' ... | tr -d '\n\r' sanitization before all GITHUB_OUTPUT writes to prevent newline injection.

4. release.yml: Added env: RELEASE_VERSION: ${{ steps.extract-version.outputs.result }} to 'Create and push tag' step. Shell now uses ${RELEASE_VERSION} instead of inline ${{ }} expressions.

