<!-- markdownlint-disable -->

# Hardening Report: techpivot--terraform-module-releaser/v2.1.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **techpivot--terraform-module-releaser/v2.1.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The 'Test Action Outputs' run: block directly interpolates ${{ steps.test-action.outputs.* }} expressions (steps.*.outputs.* context) into shell commands. These values flow through YAML template substitution before the shell sees them, enabling script injection. Offending lines include: `if [[ -n "${{ steps.test-action.outputs.changed-module-names }}" ]]`, `echo '${{ steps.test-action.outputs.changed-modules-map }}' | jq -r ...`, and similar patterns throughout the step.

Locations:

- `.github/workflows/ci.yml:66`

### script-injection (severity: high)

Sub-rule (a): Multiple run: blocks directly interpolate ${{ env.VERSION }} (sourced from github.event.inputs.release_version, a user-controlled workflow_dispatch input) and ${{ steps.app-token.outputs.app-slug }} into shell commands. Offending lines: (1) 'Validate version input' step: `if ! [[ '${{ env.VERSION }}' =~ ^[0-9]+(\.[0-9]+){2}$` and `echo "Error: ... Got: ${{ env.VERSION }}"`; (2) 'Update package.json version' step: `run: npm version --no-git-tag-version '${{ env.VERSION }}'`; (3) 'Get GitHub App User Details' step: `user_name="${{ steps.app-token.outputs.app-slug }}[bot]"` and `user_id=$(gh api "/users/${{ steps.app-token.outputs.app-slug }}[bot]" --jq .id)`.

Locations:

- `.github/workflows/release-start.yml:43`
- `.github/workflows/release-start.yml:44`
- `.github/workflows/release-start.yml:50`
- `.github/workflows/release-start.yml:100`
- `.github/workflows/release-start.yml:101`

### script-injection (severity: high)

Sub-rule (a): The 'Create and push tag' run: block directly interpolates ${{ steps.extract-version.outputs.result }} (a steps.*.outputs.* context value) into git tag and git push shell commands without routing through an env: variable. Offending lines: `git tag "v${{ steps.extract-version.outputs.result }}"` and `git push origin v${{ steps.extract-version.outputs.result }}`.

Locations:

- `.github/workflows/release.yml:59`
- `.github/workflows/release.yml:60`

### github-env-injection (severity: high)

The 'Get GitHub App User Details' step writes user_name and user_id (both derived from ${{ steps.app-token.outputs.app-slug }}, a steps.*.outputs.* context value) to $GITHUB_OUTPUT without the required sanitization step (printf '%s' ... | tr -d '\n\r'). The values are interpolated directly into shell variables via ${{ }} expressions and then written verbatim to the special environment file, allowing newline injection to poison subsequent GITHUB_OUTPUT entries.

Locations:

- `.github/workflows/release-start.yml:100`
- `.github/workflows/release-start.yml:101`
- `.github/workflows/release-start.yml:105`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all four findings across three workflow files:

1. **ci.yml** (script-injection, line 66): Moved all six `${{ steps.test-action.outputs.* }}` expressions into an `env:` block on the 'Test Action Outputs' step (CHANGED_MODULE_NAMES, CHANGED_MODULE_PATHS, CHANGED_MODULES_MAP, ALL_MODULE_NAMES, ALL_MODULE_PATHS, ALL_MODULES_MAP). All shell references now use plain `$VAR_NAME` syntax.

2. **release-start.yml** (script-injection, lines 43-44, 50): Moved `${{ env.VERSION }}` into a `VERSION_INPUT` env var for both the 'Validate version input' step (replacing single-quoted interpolation in regex test and error message) and the 'Update package.json version' step (replacing single-quoted interpolation in npm version command).

3. **release-start.yml** (script-injection + github-env-injection, lines 100-105): In 'Get GitHub App User Details', moved `${{ steps.app-token.outputs.app-slug }}` into `APP_SLUG` env var. Added sanitization using `printf '%s' "$VAR" | tr -d '\n\r'` for all values (safe_slug, safe_user_name, safe_user_id) before writing to $GITHUB_OUTPUT to prevent newline injection.

4. **release.yml** (script-injection, lines 59-60): Moved `${{ steps.extract-version.outputs.result }}` into a `RELEASE_VERSION` env var in the 'Create and push tag' step. Both `git tag` and `git push` commands now reference `${RELEASE_VERSION}` as a plain shell variable.

