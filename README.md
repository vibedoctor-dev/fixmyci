# `ci-fix`

Reusable GitHub Action for turning a failed GitHub Actions run into a VibeDoctor task at `https://app.vibedoctor.dev/api/tasks`.

The action:

- reads failure context from `workflow_run` or the current workflow environment
- fetches failed job metadata and workflow logs from GitHub Actions
- builds a prompt with repository details, failing steps, and log excerpts
- calls the VibeDoctor task API with the repo context and optional GitHub PAT

## Required secrets

- `VIBEDOCTOR_PAT`: VibeDoctor PAT used as `Authorization: Bearer ...` for `app.vibedoctor.dev`
- `CI_FIX_GITHUB_PAT`: GitHub PAT with repository access. This is forwarded to the task API as `gitPatToken`

For GitHub log collection, the action can usually use `${{ github.token }}`. Keep `CI_FIX_GITHUB_PAT` for repository access inside the created VibeDoctor task.

## Recommended install

Use a separate `workflow_run` workflow so the fixer runs after the failed workflow completes and full logs are available.

```yaml
name: CI Fix On Failure

on:
  workflow_run:
    workflows:
      - Build and Publish Next.js Docker Image
    types:
      - completed

jobs:
  submit-fix-task:
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read

    steps:
      - name: Submit VibeDoctor task
        uses: vibedoctor/vibedoctor-action@v1
        with:
          vibedoctor-pat: ${{ secrets.VIBEDOCTOR_PAT }}
          github-pat: ${{ secrets.CI_FIX_GITHUB_PAT }}
          github-token: ${{ github.token }}
```

## Inline install

If you want the call inside the failing workflow itself, add a final step with `if: failure()`. This is lighter, but log collection can be less complete than the `workflow_run` pattern.

```yaml
- name: Submit VibeDoctor task
  if: ${{ failure() }}
  uses: vibedoctor/vibedoctor-action@v1
  with:
    vibedoctor-pat: ${{ secrets.VIBEDOCTOR_PAT }}
    github-pat: ${{ secrets.CI_FIX_GITHUB_PAT }}
    github-token: ${{ github.token }}
```

## Inputs

- `vibedoctor-pat`: **Required.** Your VibeDoctor Personal Access Token.
- `github-pat`: Optional GitHub PAT. It is forwarded to the VibeDoctor API to ensure repository write access.
- `github-token`: Optional GitHub Actions scoped token (e.g. `${{ github.token }}`). Used strictly to download logs and metadata from the failed action.
- `selected-agent`: Optional AI agent (defaults to `gemini`).
- `selected-model`: Optional AI model (defaults to `gemini-3.1-pro-preview`).
- `prompt-prefix`: String prepended to the generated failure prompt.
- `extra-instructions`: String appended to the generated prompt.
- `runner-script-url`: Optional override for the runner bash script.

## Outputs

- `task-id`
- `task-url`
- `request-path`
- `response-path`

## Remote runner override

If you want the published action to bootstrap a runner script from another location, set `runner-script-url`. The downloaded script must support the same CLI arguments as `scripts/run-ci-fix.sh`.

This keeps the installer contract stable while letting you replace the execution logic independently of the action wrapper.
