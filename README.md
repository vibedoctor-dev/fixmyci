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
        uses: your-org/your-repo/ci-fix@v1
        with:
          vibedoctor-pat: ${{ secrets.VIBEDOCTOR_PAT }}
          github-pat: ${{ secrets.CI_FIX_GITHUB_PAT }}
          github-token: ${{ github.token }}
          workflow-run-id: ${{ github.event.workflow_run.id }}
          repo: ${{ github.event.workflow_run.head_repository.full_name }}
          repo-url: ${{ github.event.workflow_run.head_repository.html_url }}
          base-branch: ${{ github.event.workflow_run.head_branch }}
          workflow-name: ${{ github.event.workflow_run.name }}
          workflow-url: ${{ github.event.workflow_run.html_url }}
          event-name: ${{ github.event.workflow_run.event }}
          actor: ${{ github.event.workflow_run.actor.login }}
          head-sha: ${{ github.event.workflow_run.head_sha }}
          open-pr: 'true'
```

## Inline install

If you want the call inside the failing workflow itself, add a final step with `if: failure()`. This is lighter, but log collection can be less complete than the `workflow_run` pattern.

```yaml
- name: Submit VibeDoctor task
  if: ${{ failure() }}
  uses: your-org/your-repo/ci-fix@v1
  with:
    vibedoctor-pat: ${{ secrets.VIBEDOCTOR_PAT }}
    github-pat: ${{ secrets.CI_FIX_GITHUB_PAT }}
    github-token: ${{ github.token }}
    workflow-run-id: ${{ github.run_id }}
    repo: ${{ github.repository }}
    repo-url: ${{ github.server_url }}/${{ github.repository }}
    base-branch: ${{ github.ref_name }}
```

## Inputs

- `app-base-url`: defaults to `https://app.vibedoctor.dev`
- `task-api-path`: defaults to `/api/tasks`
- `prompt-prefix`: prepended to the generated failure prompt
- `extra-instructions`: appended to the generated prompt
- `selected-agent`: defaults to `claude`
- `selected-model`: optional model override
- `sandbox-provider`: defaults to `penify`
- `install-dependencies`: defaults to `true`
- `max-duration`: defaults to `900`
- `sandbox-lifecycle`: defaults to `keep-alive`
- `open-pr`: defaults to `false`
- `merge-pr`: defaults to `false`
- `max-log-chars`: defaults to `120000`
- `runner-script-url`: optional override for the runner script
- `dry-run`: generate the request payload without calling the API

## Outputs

- `task-id`
- `task-url`
- `request-path`
- `response-path`

## Remote runner override

If you want the published action to bootstrap a runner script from another location, set `runner-script-url`. The downloaded script must support the same CLI arguments as `scripts/run-ci-fix.sh`.

This keeps the installer contract stable while letting you replace the execution logic independently of the action wrapper.
