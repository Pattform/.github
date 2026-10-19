# .github
Pattform Readme

## Reusable deploy workflow

`.github/workflows/pattform-deploy.yml` deploys an app repo's `deploy/*.yaml` specs with `@pattform/engine` (ADR-001). Each app repo carries only this caller:

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  schedule: [{ cron: "0 17 * * *" }]   # nightly drift check
  workflow_dispatch:
jobs:
  deploy:
    uses: Pattform/.github/.github/workflows/pattform-deploy.yml@main
    permissions: { contents: read, id-token: write }
```

| Event | Runs |
| --- | --- |
| Pull request | `pattform validate`, then `pattform preview` |
| Push to main, manual run | `pattform validate`, then `pattform up` (Client specs first) |
| Schedule | `pattform validate`, then `pattform drift` (fails on any drift) |

AWS and GCP access is keyless through GitHub OIDC (`pattform-github-deploy` role, `github-actions` workload identity pool). Callers pin the workflow by ref; switch `@main` to a release tag once one exists.
