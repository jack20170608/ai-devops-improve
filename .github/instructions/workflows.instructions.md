---
applyTo: ".github/workflows/**/*.yml,.github/workflows/**/*.yaml"
---

# GitHub Actions instructions

Human-readable source: `10-Devops-SOP/03-secure-coding-standard.md`.

## Permissions and credentials

- Set default `GITHUB_TOKEN` permissions to read-only.
- Declare the minimum required permissions at job level.
- Use OIDC for cloud access; do not add long-lived cloud credentials.
- Never print secrets or assume transformed secret values will be masked.
- Protect production environments with required reviewers.

Good:

```yaml
permissions:
  contents: read

jobs:
  deploy:
    environment: production
    permissions:
      contents: read
      id-token: write
```

Do not use:

```yaml
permissions: write-all
```

## Actions and scripts

- Pin every third-party action and reusable workflow to a reviewed full commit SHA.
- Do not use floating references such as `@main`.
- Review action source and update pinned SHAs deliberately.
- Do not interpolate untrusted GitHub context directly into shell code.
- Pass untrusted values through environment variables and quote them in scripts.
- Set `persist-credentials: false` on checkout unless later Git operations require credentials.

Unsafe:

```yaml
- uses: third-party/build-action@main
- run: echo "${{ github.event.pull_request.title }}"
```

Safer:

```yaml
- uses: actions/checkout@<reviewed-full-commit-sha>
  with:
    persist-credentials: false
- name: Validate PR title
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: ./scripts/validate-pr-title "$PR_TITLE"
```

## Trust separation

- Do not use privileged workflows to checkout and execute untrusted pull-request code.
- Avoid `pull_request_target` unless the design prevents execution of untrusted content.
- Separate untrusted build/test jobs from publish/deploy jobs.
- Treat artifacts, caches, outputs, and generated metadata from untrusted jobs as untrusted.
- Do not run public or fork pull requests on persistent privileged self-hosted runners.
- Use isolated, ephemeral, credential-free runners for untrusted code.

## Reproducibility and gates

- Use the repository Wrapper and the same validation command developers run locally.
- Cache only immutable or integrity-checked inputs.
- Do not hide failed tests with `continue-on-error` unless the check is explicitly informational.
- Production changes require build, tests, code scanning, dependency review, secret policy, and required human approval.
- Release jobs must record source revision and artifact digest and produce an SBOM and provenance where supported.

Before completion:

- Check permissions for every job.
- Check every external action is pinned to a full SHA.
- Check untrusted values never become executable shell text.
- Check privileged and untrusted work are separated.
- Check secrets cannot be exposed through logs, artifacts, caches, or outputs.

