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
| Push to main, manual run on main | `pattform validate`, then `pattform up` (Client specs first) |
| Manual run on any other branch | `pattform validate`, then `pattform preview` (the portal's dry runs, on `portal/dry-run/*`) |
| Schedule | `pattform validate`, then `pattform drift` (fails on any drift) |

AWS and GCP access is keyless through GitHub OIDC (`pattform-github-deploy` role, `github-actions` workload identity pool). Callers pin the workflow by ref; switch `@main` to a release tag once one exists.

## Renovate preset

`default.json` is the shared Renovate config for every Pattform and PartPlanner repo (the Mend Renovate GitHub App runs it). Each repo carries only:

```json
{ "extends": ["github>Pattform/.github"] }
```

- **Automerge**: non-major updates merge (squash) once every check on the PR passes, which for app repos is the deploy workflow's validate + preview. Repos with no PR checks never automerge; their PRs wait for the portal. Majors open a PR but never automerge; merge them from the portal. Every pending update is a PR, so the portal (which reads PRs, not issues) sees how far behind each repo is.
- **Capabilities**: a regex manager reads `platform.pattform.io/capability-version` in `deploy/*.yaml` and maps the spec's `kind` to its npm package (`LandingSite` is `@pattform/capability-landing-site`). All capability bumps in a repo land in one `renovate/pattform-capabilities` PR labelled `capability`.
- **Engine**: a second regex manager bumps the `engine-version` default in `pattform-deploy.yml`.
- `@pattform/*` packages skip the 3-day release-age wait that applies to everything else.
- Platform infrastructure repos (`pattform-infrastructure`, `partplanner-infrastructure`, `pattform-bootstrap`) set `"automerge": false`.
