# idp-platform

Shared CI for services on the IDP. A service repo does not copy a pipeline; it calls the
workflow here, pinned to a commit. One place decides what "CI" means, and every service
gets a fix or a new check through a version bump.

Related repos: `idp-portal` (Backstage and the service template), `idp-gitops` (what runs in
the cluster), `idp-infra` (AWS, Terraform).

## Workflows

| Workflow | For | Does |
|---|---|---|
| `.github/workflows/python-service.yml` | Python services from the golden path | lint, format check, type check, tests with coverage gate, arm64 image build, Trivy scan, push to ECR `svc/<repo>` on `main`, then signed **build provenance** and a signed **SBOM** for the pushed image |

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
    uses: surya-idp/idp-platform/.github/workflows/python-service.yml@<commit SHA> # v2.0.0
    permissions:
      contents: read
      id-token: write       # AWS login and Sigstore signing, both without stored keys
      attestations: write   # required since v2.0.0
```

Pin a full commit SHA, not a tag or branch: a tag can be moved, a SHA cannot. The comment
after it tells Dependabot which release the SHA is, so it can open the bump PR.

| Input | Default | Meaning |
|---|---|---|
| `working-directory` | `.` | where the service lives in the repo |
| `push` | `true` | push the image on pushes to `main`; `false` = build and scan only |
| `registry-self-test` | `false` | only for this repo's own CI: push to a throwaway registry inside the job, attest, verify |

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

## What the attestations are, and how to check one

A tag in a registry says nothing about where an image came from. After pushing, the workflow
creates two **attestations** for the image digest and stores them next to the image:

| Attestation | Says | Predicate type |
|---|---|---|
| Build provenance (SLSA) | this digest was built by this workflow, from this repo and commit, on this runner | `https://slsa.dev/provenance/v1` |
| SBOM (CycloneDX) | every package inside the image, with versions | `https://cyclonedx.org/bom` |

Each is signed with a certificate that Sigstore issues to the workflow run for a few minutes
(keyless: there is no signing key to store, rotate or leak) and recorded in a public
transparency log.

```bash
# needs read access to the registry (for ECR: aws ecr get-login-password | docker login ...)
gh attestation verify oci://693906847772.dkr.ecr.ap-south-1.amazonaws.com/svc/<name>@sha256:<digest> \
  --owner surya-idp \
  --signer-workflow surya-idp/idp-platform/.github/workflows/python-service.yml
# the SBOM: add  --predicate-type https://cyclonedx.org/bom
```

`--signer-workflow` is why this lives in a shared workflow: the signer recorded in the
certificate is **this file**, not the service repo. A service cannot produce an attestation
that passes this check with a pipeline of its own.

Not done yet: nothing in the cluster **refuses** an image without a valid attestation. That is
an admission policy (Kyverno `verifyImages`), planned with the policy suite.

## Releases

| Change | Version | How services get it |
|---|---|---|
| fix, or a check that existing services already pass | patch / minor (`v1.0.1`, `v1.1.0`) | Dependabot PR in each service repo, merged when its CI is green |
| a new requirement on the caller repo (the contract changes) | major (`v2.0.0`) | same PR, but it needs a change in the service first |

To release: merge to `main`, then tag the merge commit (`git tag v2.x.y && git push origin v2.x.y`).

| Version | Change |
|---|---|
| v1.0.0 | check + image (lint, types, tests, build, scan, push) |
| v2.0.0 | attests every pushed image (provenance + SBOM). **Breaking:** callers must also grant `attestations: write` |
| v2.1.0 | builds the repo's docs (if it has a `mkdocs.yml`) and publishes them on main to the docs bucket for Backstage TechDocs; image builds of one branch run one at a time, so the newest commit is always built last. No change needed in callers |
| v2.1.1 | an image is pushed only for the commit that is the branch head when its build runs. v2.1.0 built one at a time but not in push order: an older commit could be built last and deployed |

## How changes here are tested

This repo's own CI (`.github/workflows/ci.yml`) lints every workflow with actionlint and then
runs `python-service.yml` for real against `tests/fixture-service`, a small service that
satisfies the contract, with `registry-self-test: true`: the image is pushed to a registry that
exists only inside the job (with basic authentication, like ECR), attested there, and the
attestations are verified, including a check that a different signer workflow is rejected.
Nothing reaches AWS. A change that breaks the shared CI fails here, before a service repo can
pick it up.

What the self-test cannot cover: the AWS steps (OIDC login, creating the ECR repository, the
push to ECR, storing attestations in ECR). Those run for the first time when a service pushes
to `main`.
