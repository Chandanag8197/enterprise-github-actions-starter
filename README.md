# Enterprise GitHub Actions Starter

This repository documents the **10 GitHub Actions most commonly used in enterprise CI/CD** and includes a sample workflow you can copy.

Repo: https://github.com/Chandanag8197/enterprise-github-actions-starter

---

## The 10 actions enterprises use most

These are the building blocks you will see in almost every production pipeline: checkout code, install a language, cache dependencies, scan for secrets/vulns, build a container, upload artifacts, and deploy.

### 1. `actions/checkout`
**What it does:** Clones the repo onto the runner so later steps can build/test it.

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0   # full history if you need changelog / tags
```

Enterprise notes: pin to a major version or SHA. For private submodules, pass a PAT or `token`.

---

### 2. `actions/setup-node` (or setup-python / setup-java / setup-go / setup-dotnet)
**What it does:** Installs a language runtime and optionally caches package manager deps.

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
```

Same family (pick one per stack):
- `actions/setup-python@v5`
- `actions/setup-java@v4`
- `actions/setup-go@v5`
- `actions/setup-dotnet@v4`

---

### 3. `actions/cache`
**What it does:** Speeds up CI by restoring `node_modules`, Maven/Gradle caches, pip wheels, etc.

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

Prefer the built-in cache in `setup-*` when it exists. Use this action when you need custom paths.

---

### 4. `actions/upload-artifact` + `actions/download-artifact`
**What it does:** Passes build outputs (jars, coverage reports, test results) between jobs or keeps them for 1-90 days.

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
    retention-days: 7
```

---

### 5. `docker/login-action` + `docker/setup-buildx-action` + `docker/build-push-action`
**What it does:** Authenticates to GHCR / ECR / ACR / Docker Hub, then builds and pushes an image.

```yaml
- uses: docker/setup-buildx-action@v3
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
- uses: docker/build-push-action@v6
  with:
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

This trio is the default container path in most enterprises.

---

### 6. `github/codeql-action` (init + analyze)
**What it does:** Static application security testing (SAST) on PRs and default branch. First-party GitHub Advanced Security.

```yaml
- uses: github/codeql-action/init@v3
  with:
    languages: javascript
- uses: github/codeql-action/analyze@v3
```

Often paired with Dependabot and secret scanning at org level.

---

### 7. Secret / vuln scanners: `trufflesecurity/trufflehog` or `aquasecurity/trivy-action`
**What it does:** Finds leaked credentials and container/OS/library CVEs before merge.

```yaml
- uses: aquasecurity/trivy-action@0.28.0
  with:
    scan-type: 'fs'
    scan-ref: '.'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

Enterprises usually allow-list only verified actions (GitHub or approved vendors).

---

### 8. Cloud auth: `aws-actions/configure-aws-credentials` / `azure/login` / `google-github-actions/auth`
**What it does:** OIDC federation into AWS, Azure, or GCP — no long-lived cloud keys in GitHub secrets.

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions
    aws-region: ap-south-1
```

Requires `permissions: id-token: write`. This is the enterprise standard over stored access keys.

---

### 9. `actions/github-script`
**What it does:** Run short JS against the GitHub API (comment on PRs, label issues, fail checks) without writing a custom action.

```yaml
- uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: 'CI passed. Ready for review.'
      })
```

---

### 10. Reusable workflows + `workflow_call` (and org templates)
**What it does:** One golden pipeline used by hundreds of repos. This is how enterprises stay consistent.

Caller:
```yaml
jobs:
  ci:
    uses: my-org/platform-workflows/.github/workflows/node-ci.yml@v1
    secrets: inherit
```

Called workflow lives in a central repo:
```yaml
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '20'
```

Org admins also restrict which actions can run (GitHub-verified + allow-list) and use rulesets so PRs cannot merge without required checks.

---

## Sample workflow in this repo

See `.github/workflows/ci.yml`.

It demonstrates:
1. checkout
2. setup-node + cache
3. lint / test placeholder
4. upload-artifact
5. github-script summary

Enable Actions on the repo, push to `main` or open a PR, and the workflow will run.

---

## Enterprise practices (short list)

| Practice | Why |
|---|---|
| Pin actions to SHA or major tag | Supply-chain attacks on tags |
| Least-privilege `permissions:` | Default GITHUB_TOKEN is too wide |
| OIDC to cloud (not static keys) | Rotate-less, auditable |
| Environments + required reviewers | Protect prod deploys |
| Action allow-list at org | Block untrusted Marketplace actions |
| Reusable workflows | One pipeline, many repos |
| Self-hosted / larger runners | Private network, more CPU |
| Required status checks + rulesets | No merge if CI/security fails |

---

## Next steps

1. Clone: `git clone https://github.com/Chandanag8197/enterprise-github-actions-starter.git`
2. Add a real app (Node, Python, Java) and wire tests into `ci.yml`
3. Add CodeQL and Trivy jobs when you have source to scan
4. If you deploy, add OIDC + environment `production`
