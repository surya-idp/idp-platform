# idp-platform

Shared CI for services on the IDP. A service repo does not copy a pipeline; it calls the
workflow here, pinned to a commit. One place decides what "CI" means, and every service
gets a fix or a new check through a version bump.

Related repos: `idp-portal` (Backstage and the service template), `idp-gitops` (what runs in
the cluster), `idp-infra` (AWS, Terraform).

## Workflows

| Workflow | For | Does |
|---|---|---|
| `.github/workflows/python-service.yml` | Python services from the golden path | lint, format check, type check, tests with coverage gate, arm64 image build, Trivy scan, push to ECR `svc/<repo>` on `main` |

It does not deploy. Argo CD Image Updater (in `idp-gitops`) notices the new image and commits
its tag; Argo CD rolls it out. Service repos therefore hold no credential for the deploy repo.

## Use it

`.github/workflows/ci.yml` in the service repo (the template writes this file):

```yaml
name: CI
on:
  push:
    branches: ["**"]
  pull_request:
concurrency:
  group: ci-${{ github.sha }}
  cancel-in-progress: true
permissions: {}
jobs:
  ci:
    uses: surya-idp/idp-platform/.github/workflows/python-service.yml@<commit SHA> # v1.0.0
    permissions:
      contents: read
      id-token: write
```

Pin a full commit SHA, not a tag or branch: a tag can be moved, a SHA cannot. The comment
after it tells Dependabot which release the SHA is, so it can open the bump PR.

| Input | Default | Meaning |
|---|---|---|
| `working-directory` | `.` | where the service lives in the repo |
| `push` | `true` | push the image on pushes to `main`; `false` = build and scan only |

## The contract

What the workflow expects in the calling repo:

| It runs | So the repo needs |
|---|---|
| `uv sync --locked` | `pyproject.toml` and a matching `uv.lock` |
| `uv run ruff check .` and `uv run ruff format --check .` | `ruff` in the `dev` dependency group |
| `uv run mypy` | `mypy` in `dev`, and `[tool.mypy]` listing the files |
| `uv run pytest` | `pytest` in `dev`; the coverage gate is the repo's own `fail_under` |
| `docker build --build-arg GIT_SHA=<sha> .` | a `Dockerfile` that builds on arm64 |

The image is pushed as `693906847772.dkr.ecr.ap-south-1.amazonaws.com/svc/<repo name>:<commit SHA>`.
The ECR repository is created on the first push. AWS access is OIDC (role `idp-service-ci`,
which can only write ECR repos under `svc/*`); there are no stored keys.

## Releases

| Change | Version | How services get it |
|---|---|---|
| fix, or a check that existing services already pass | patch / minor (`v1.0.1`, `v1.1.0`) | Dependabot PR in each service repo, merged when its CI is green |
| a new requirement on the caller repo (the contract changes) | major (`v2.0.0`) | same PR, but it needs a change in the service first |

To release: merge to `main`, then tag the merge commit (`git tag v1.x.y && git push origin v1.x.y`).

## How changes here are tested

This repo's own CI (`.github/workflows/ci.yml`) lints every workflow with actionlint and then
runs `python-service.yml` for real against `tests/fixture-service`, a small service that
satisfies the contract, with `push: false`. A change that breaks the shared CI fails here,
before a service repo can pick it up.
