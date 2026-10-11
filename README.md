# idp-platform

Shared CI for services on the IDP. A service repo does not copy a pipeline; it calls the
workflow here, pinned to a commit. One place decides what "CI" means, and every service
gets a fix or a new check through a version bump.

It is one of five repositories of the platform:

| Repository | Role |
|---|---|
| [idp-infra](https://github.com/surya-idp/idp-infra) | Terraform for AWS, `make up` / `make down`, the project's master documents |
| [idp-gitops](https://github.com/surya-idp/idp-gitops) | What runs in the cluster |
| [idp-portal](https://github.com/surya-idp/idp-portal) | Backstage and the service template that writes the caller file |
| **idp-platform** (this one) | The CI workflow every template service calls |
| [feature-flag-service](https://github.com/surya-idp/feature-flag-service) | The platform's first application, the flag API |

## Workflows

| Workflow | For | Does |
|---|---|---|
| `.github/workflows/python-service.yml` | Python services from the golden path | Three jobs, below |

| Job | Runs | Does |
|---|---|---|
| `check` | every push and pull request | lock file check, lint, format check, type check, tests with the repo's coverage gate |
| `image` | every push and pull request; pushes on `main` only | build the arm64 image, scan it with Trivy, then on `main`: push to ECR `svc/<repo>` **only if this commit is still the newest on the branch**, and attest the pushed image with signed **build provenance** and a signed **SBOM** |
| `docs` | every push and pull request; publishes on `main` only | if the repo has a `mkdocs.yml`: build its documentation site (a broken page fails the pull request) and publish it to the docs bucket, where Backstage reads it |

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
    uses: surya-idp/idp-platform/.github/workflows/python-service.yml@<commit SHA> # v2.1.1
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
| `techdocs-cli generate` (only if `mkdocs.yml` exists) | optional: `mkdocs.yml` with the `techdocs-core` plugin and a `docs/` folder |

The image is pushed as `693906847772.dkr.ecr.ap-south-1.amazonaws.com/svc/<repo name>:<commit SHA>`.
The ECR repository is created on the first push. Documentation is published to
`s3://idp-techdocs-693906847772/default/component/<repo name>/`. AWS access is OIDC (role
`idp-service-ci`, which can only write ECR repos under `svc/*` and docs under
`default/component/*`); there are no stored keys.

### Why only the newest commit is pushed

The cluster's Image Updater deploys the image that was **built last**. Two pushes seconds
apart start two runs. The `image` job has a concurrency group, so they build one at a time,
but GitHub does not start waiting jobs in push order: on 2026-10-11 the newer commit built
first, the older one built last, and the older code was deployed. So before pushing, the job
asks GitHub for the branch head and skips the push if a newer commit exists; that commit's own
run pushes. Either order then ends with the newest commit built last. A skipped commit is
still built and scanned, so its checks mean what they say.

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

In the cluster, the Kyverno policy `verify-image-provenance` (`idp-gitops/platform/kyverno/`)
checks the provenance of every image under `svc/` at admission. It reports and does not block
yet, because the platform's own two images are not built by this workflow.

One detail decides whether that policy passes: the certificate names this workflow **by the
ref the caller used**. Callers pin a commit, so the name ends in a commit id, not in a tag.
The policy therefore lists the commit of each release. See "Releases" below.

## Releases

| Change | Version | How services get it |
|---|---|---|
| fix, or a check that existing services already pass | patch / minor (`v1.0.1`, `v1.1.0`) | Dependabot PR in each service repo, merged when its CI is green |
| a new requirement on the caller repo (the contract changes) | major (`v2.0.0`) | same PR, but it needs a change in the service first |

To release, three steps. The second and third are easy to forget and nothing fails loudly if
they are:

1. Merge to `main`, then tag the merge commit:
   `git tag -a v2.x.y -m "..." && git push origin v2.x.y`, and note its commit id
   (`git rev-parse HEAD`).
2. In `idp-gitops`, add that commit id to the signer list in
   `platform/kyverno/verify-image-provenance.yaml`. Without it, images built by the new release
   are reported as unattested.
3. In `idp-portal`, set the pin in `templates/python-service/skeleton/.github/workflows/ci.yml`
   to the same commit id, so new services start on the new release. Merge step 2 first.

Existing services get the release through a Dependabot pull request in their own repository.

| Version | Change |
|---|---|
| v1.0.0 | check + image (lint, types, tests, build, scan, push) |
| v2.0.0 (`291df7b`) | attests every pushed image (provenance + SBOM). **Breaking:** callers must also grant `attestations: write` |
| v2.1.0 (`48676e8`) | builds the repo's docs (if it has a `mkdocs.yml`) and publishes them on main to the docs bucket for Backstage TechDocs; image builds of one branch run one at a time. No change needed in callers. The ordering claim made for this release was wrong: see v2.1.1 |
| v2.1.1 (`3d15dbf`) | an image is pushed only for the commit that is the branch head when its build runs. v2.1.0 built one at a time but not in push order: an older commit could be built last and deployed |

## How changes here are tested

This repo's own CI (`.github/workflows/ci.yml`) lints every workflow with actionlint and then
runs `python-service.yml` for real against `tests/fixture-service`, a small service that
satisfies the contract, with `registry-self-test: true`: the image is pushed to a registry that
exists only inside the job (with basic authentication, like ECR), attested there, and the
attestations are verified, including a check that a different signer workflow is rejected.
Nothing reaches AWS. A change that breaks the shared CI fails here, before a service repo can
pick it up.

The fixture also has a documentation page, so the `docs` job builds a site on every run.

What the self-test cannot cover: the AWS steps (OIDC login, creating the ECR repository, the
push to ECR, storing attestations in ECR, publishing docs to the bucket) and the "is this
still the newest commit" decision. Those run for the first time when a service pushes to
`main`. All of them were exercised on the cluster on 2026-10-11 with v2.1.1: a new service
was built, attested and documented, and of two commits pushed one second apart only the newer
one's image reached ECR.

## Known limits

- The `docs` job is not ordered the way the `image` job is: of two quick pushes, the older
  commit's documentation could be published last.
- One AWS role serves every service repository, so a service could overwrite another
  service's documentation. It cannot touch another service's images through this workflow's
  normal path, but the role's ECR permission is the whole `svc/*` prefix.
- One language. A Go or Node service needs a workflow of its own next to this one.
