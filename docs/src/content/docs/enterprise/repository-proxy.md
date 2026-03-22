---
title: "Repository Proxy"
description: "Route APM package downloads through a VCS repository proxy for caching, access control, and air-gapped environments."
sidebar:
  order: 6
---

Organizations often need to route package downloads through a centralized proxy — for caching, access control, audit trails, or because direct internet access is restricted. APM supports this by downloading git archives over HTTP through a VCS repository proxy, rather than cloning repositories directly.

This is especially useful for:

- **Air-gapped environments** where direct access to GitHub/GitLab is blocked
- **Centralized access control** through a single gateway with token-based auth
- **Caching and bandwidth savings** by proxying repeated downloads
- **Audit and compliance** with centralized download logs

## Supported proxy: JFrog Artifactory

APM currently supports [JFrog Artifactory](https://jfrog.com/artifactory/) as a VCS repository proxy. Artifactory serves git archives over plain HTTP — APM downloads zip files from it the same way it would from GitHub or GitLab, with no special protocol or API required.

## Prerequisites

You need a **Remote VCS repository** in your Artifactory instance that points at the upstream git host (e.g., GitHub). This repository type caches git archives served over HTTP.

### Creating the repository

Create a Remote VCS repository in your Artifactory instance:

- **JFrog Cloud (SaaS)**: `https://<your-instance>.jfrog.io` → Administration → Repositories → Remote → New Remote Repository
- **Self-hosted**: `https://<your-server>/artifactory` → same navigation path

Choose a **repository key** — this is the name that appears in the URL path. Common conventions:

| Upstream host | Suggested repo key | Resulting URL path |
|---------------|-------------------|-------------------|
| GitHub | `github` | `artifactory/github/owner/repo` |
| GitLab | `gitlab` | `artifactory/gitlab/owner/repo` |
| Custom/internal | `vcs-proxy` | `artifactory/vcs-proxy/owner/repo` |

The repo key is free-form — use whatever name makes sense in your organization. APM does not enforce any specific value.

Set the repository **URL** to the upstream git host (e.g., `https://github.com`).

For detailed instructions, see [JFrog's Remote Repositories documentation](https://jfrog.com/help/r/jfrog-artifactory-documentation/remote-repositories).

### Generating a token

Create a **JFrog Access Token** with read permissions on the VCS repository:

- **JFrog Cloud**: User Profile → Access Tokens → Generate Token
- **Self-hosted**: Administration → Identity and Access → Access Tokens

For detailed instructions, see [JFrog's Access Tokens documentation](https://jfrog.com/help/r/jfrog-artifactory-documentation/access-tokens).

## Configuration

APM supports three operating modes, all configured via environment variables.

### Mode 1: Transparent proxy

All GitHub-hosted dependencies are silently routed through the proxy. Your `apm.yml` stays unchanged — no URL rewriting needed.

```bash
export ARTIFACTORY_BASE_URL=https://mycompany.jfrog.io/artifactory/github
export ARTIFACTORY_APM_TOKEN=your-access-token

apm install
```

APM rewrites download URLs on the fly — `microsoft/apm-sample-package` fetches from `https://mycompany.jfrog.io/artifactory/github/microsoft/apm-sample-package/archive/refs/heads/main.zip` instead of GitHub directly.

This mode only intercepts dependencies that resolve to GitHub hosts. Non-GitHub dependencies (GitLab, Bitbucket, Azure DevOps) are downloaded directly unless you use [air-gapped mode](#mode-3-air-gapped).

### Mode 2: Explicit FQDN

Point individual dependencies directly at your Artifactory instance in `apm.yml`:

```yaml
dependencies:
  apm:
    - mycompany.jfrog.io/artifactory/github/microsoft/apm-sample-package
    - mycompany.jfrog.io/artifactory/github/team/internal-skills#v2.0
    - mycompany.jfrog.io/artifactory/gitlab/acme/coding-standards
```

```bash
export ARTIFACTORY_APM_TOKEN=your-access-token
apm install
```

The URL format is:

```
<host>/artifactory/<repo-key>/<owner>/<repo>
```

APM detects proxy URLs by looking for `artifactory` as the first path segment. The second segment is the repository key you chose when creating the remote repository.

You can mix proxied and direct dependencies in the same manifest:

```yaml
dependencies:
  apm:
    - mycompany.jfrog.io/artifactory/github/team/private-skills  # via proxy
    - microsoft/apm-sample-package                                 # direct from GitHub
```

### Mode 3: Air-gapped

Block all direct git access. Every download must go through the proxy — if a dependency can't be routed, APM raises an error instead of reaching out to the public internet.

```bash
export ARTIFACTORY_BASE_URL=https://mycompany.jfrog.io/artifactory/github
export ARTIFACTORY_APM_TOKEN=your-access-token
export ARTIFACTORY_ONLY=true

apm install
```

When `ARTIFACTORY_ONLY` is set without `ARTIFACTORY_BASE_URL`, APM only allows dependencies with explicit Artifactory FQDN syntax (Mode 2). Any shorthand dependency (e.g., `microsoft/apm-sample-package`) will fail.

## Subdirectory packages

Subdirectory (monorepo) packages work through the proxy. APM downloads the full archive and extracts the requested subdirectory:

```yaml
dependencies:
  apm:
    # Explicit FQDN with subdirectory
    - mycompany.jfrog.io/artifactory/github/ComposioHQ/awesome-claude-skills/canvas-design#master
```

With transparent proxy mode, the syntax is identical to direct GitHub access:

```yaml
dependencies:
  apm:
    - ComposioHQ/awesome-claude-skills/canvas-design#master
```

Virtual file packages (e.g., `owner/repo/prompts/review.prompt.md`) also work through the proxy — APM fetches the archive and extracts the single file.

## Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `ARTIFACTORY_APM_TOKEN` | Yes | Access token with read permissions on the VCS repository. Sent as `Authorization: Bearer <token>`. |
| `ARTIFACTORY_BASE_URL` | For Mode 1 & 3 | Full URL to your VCS remote repository. Must include scheme (`https://` or `http://`). Example: `https://mycompany.jfrog.io/artifactory/github` |
| `ARTIFACTORY_ONLY` | For Mode 3 | Set to `true`, `1`, or `yes` to block direct git operations. |
| `ARTIFACTORY_MAX_ARCHIVE_MB` | No | Maximum allowed archive size in megabytes. Default: `500`. |

## CI/CD

Inject the proxy variables as secrets in your pipeline:

### GitHub Actions

```yaml
- name: Install APM dependencies
  env:
    ARTIFACTORY_BASE_URL: ${{ secrets.ARTIFACTORY_BASE_URL }}
    ARTIFACTORY_APM_TOKEN: ${{ secrets.ARTIFACTORY_APM_TOKEN }}
  run: apm install
```

### Azure Pipelines

```yaml
- script: apm install
  env:
    ARTIFACTORY_BASE_URL: $(ARTIFACTORY_BASE_URL)
    ARTIFACTORY_APM_TOKEN: $(ARTIFACTORY_APM_TOKEN)
```

## Debugging

Enable debug output to see exactly which URLs APM tries:

```bash
export APM_DEBUG=1
apm install --verbose
```

You should see lines like:

```
[DEBUG] Trying Artifactory archive: https://mycompany.jfrog.io/artifactory/github/microsoft/apm-sample-package/archive/refs/heads/main.zip
```

APM tries multiple URL patterns for each package (GitHub-style and GitLab-style archive paths), falling back automatically if the first pattern returns a 404.

## How it works

When APM resolves a dependency through the proxy, it:

1. Builds archive download URLs for the package (both GitHub-style `/archive/refs/heads/{ref}.zip` and GitLab-style `/-/archive/{ref}/{repo}-{ref}.zip`)
2. Sends an HTTP GET with `Authorization: Bearer <token>` to each URL until one succeeds
3. Extracts the zip archive, stripping the root directory prefix (e.g., `repo-main/`)
4. Validates the extracted package (must contain `apm.yml`, `SKILL.md`, or `plugin.json`)

No git clone happens — the proxy serves the archive over plain HTTP. Git credentials and SSH keys are not needed for proxied dependencies.

## Limitations

- **No commit SHA resolution**: Proxied dependencies resolve by branch/tag name, not by commit SHA. The lockfile records the ref name but `resolved_commit` is `null`.
- **GitHub-only transparent proxy**: Mode 1 only intercepts GitHub-hosted dependencies. For GitLab or Bitbucket, use explicit FQDN (Mode 2) or air-gapped mode (Mode 3) with a matching remote repository.
- **Archive-based only**: APM downloads zip archives, not git objects. Operations like `git log` or `git blame` on package contents are not available.
