<!-- markdownlint-disable -->

# Hardening Report: techpivot--terraform-module-releaser/v2.2.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **techpivot--terraform-module-releaser/v2.2.0** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): Direct ${{ }} expression interpolation inside run: shell commands. The 'Test Action Outputs' step interpolates ${{ steps.test-action.outputs.changed-module-names }}, ${{ steps.test-action.outputs.changed-module-paths }}, ${{ steps.test-action.outputs.changed-modules-map }}, ${{ steps.test-action.outputs.all-module-names }}, ${{ steps.test-action.outputs.all-module-paths }}, and ${{ steps.test-action.outputs.all-modules-map }} directly into shell commands. These step outputs may contain attacker-controlled data (e.g. from PR content processed by the action) and are interpolated before the shell parses the command, enabling command injection.

Locations:

- `.github/workflows/ci.yml:68`

### script-injection (severity: high)

Rule (a): Direct ${{ }} expression interpolation inside run: shell commands. The 'Validate version input' step interpolates ${{ env.VERSION }} (sourced from the workflow_dispatch input github.event.inputs.release_version) directly into shell commands: `if ! [[ '${{ env.VERSION }}' =~ ... ]]` and `echo "Error: ... ${{ env.VERSION }}"`. The 'Update package.json version' step also interpolates it: `npm version --no-git-tag-version '${{ env.VERSION }}'`. Even though single-quoting is used in some places, any ${{ }} expression in a run: block is a script-injection finding because YAML template substitution occurs before the shell ever sees the string.

Locations:

- `.github/workflows/release-start.yml:43`
- `.github/workflows/release-start.yml:44`
- `.github/workflows/release-start.yml:50`

### script-injection (severity: high)

Rule (a): Direct ${{ }} expression interpolation inside run: shell commands. The 'Get GitHub App User Details' step interpolates ${{ steps.app-token.outputs.app-slug }} directly into shell variable assignments and a gh API call: `user_name="${{ steps.app-token.outputs.app-slug }}[bot]"` and `user_id=$(gh api "/users/${{ steps.app-token.outputs.app-slug }}[bot]" --jq .id)`. The app-slug value flows through YAML template substitution before the shell parses the command, enabling command injection if the value contains shell metacharacters.

Locations:

- `.github/workflows/release-start.yml:100`
- `.github/workflows/release-start.yml:101`

### script-injection (severity: high)

Rule (a): Direct ${{ }} expression interpolation inside run: shell commands. The 'Create and push tag' step interpolates ${{ steps.extract-version.outputs.result }} directly into git commands: `git tag "v${{ steps.extract-version.outputs.result }}"` and `git push origin v${{ steps.extract-version.outputs.result }}`. Although the version is extracted from the PR body via a regex in a prior github-script step, the value is still a ${{ steps.* }} expression interpolated directly into a run: shell block, which is a script-injection finding.

Locations:

- `.github/workflows/release.yml:56`
- `.github/workflows/release.yml:57`

### github-env-injection (severity: high)

The 'Get GitHub App User Details' step in release-start.yml writes values derived from ${{ steps.app-token.outputs.app-slug }} to $GITHUB_OUTPUT without sanitization. The shell variables user_name and user_id are set from the interpolated app-slug value and then written via `echo "user-name=${user_name}"`, `echo "user-id=${user_id}"`, and `echo "email=${user_id}+${user_name}@..."` redirected to $GITHUB_OUTPUT. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the writes, allowing newline injection into the output file which could poison subsequent step outputs.

Locations:

- `.github/workflows/release-start.yml:102`
- `.github/workflows/release-start.yml:103`
- `.github/workflows/release-start.yml:104`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all findings across three workflow files:

1. **ci.yml** (line 68): Moved all six `${{ steps.test-action.outputs.* }}` expressions into the step's `env:` block (CHANGED_MODULE_NAMES, CHANGED_MODULE_PATHS, CHANGED_MODULES_MAP, ALL_MODULE_NAMES, ALL_MODULE_PATHS, ALL_MODULES_MAP). Replaced `echo '${{ ... }}'` with `printf '%s' "$VAR"` for safe piping to jq.

2. **release-start.yml** (lines 43-44, 50): Fixed 'Validate version input' and 'Update package.json version' steps by moving `${{ env.VERSION }}` into each step's `env:` block as `INPUT_VERSION` and referencing it as `"$INPUT_VERSION"` in the shell.

3. **release-start.yml** (lines 100-104): Fixed 'Get GitHub App User Details' step by moving `${{ steps.app-token.outputs.app-slug }}` into the `env:` block as `APP_SLUG`. Also fixed the github-env-injection by sanitizing all three output values (user-name, user-id, email) with `printf '%s' "$VAR" | tr -d '\n\r'` before writing to `$GITHUB_OUTPUT`.

4. **release.yml** (lines 56-57): Fixed 'Create and push tag' step by moving `${{ steps.extract-version.outputs.result }}` into the step's `env:` block as `RELEASE_VERSION` and referencing it as `"v${RELEASE_VERSION}"` in git commands.

