# Hermes Agent with WebUI

Custom deployment assets for Hermes WebUI and its private SearXNG search
sidecar.

## Container images

Only the customized Hermes WebUI is built in this repository. SearXNG uses its
official, pinned upstream image directly because its behavior is customized by
`searxng/settings.yml`, not by changing the image.

| Service | Image |
|---|---|
| Hermes WebUI | `ghcr.io/vaenae/hermes-webui` |
| SearXNG | `docker.io/searxng/searxng:2026.9.8-3fdc6d753` |

The WebUI image adds Chromium, Node.js 24, and `agent-browser` to the upstream
Hermes WebUI image. It supports both `linux/amd64` and `linux/arm64`.

## Publishing the WebUI image

`.github/workflows/publish-hermes-webui.yml` builds and publishes the WebUI
image to GitHub Container Registry (GHCR). It runs when:

- a WebUI or workflow file is pushed to `main`;
- a `v*` Git tag is pushed; or
- it is started manually from GitHub's **Actions** tab.

The workflow publishes these tags:

- `latest` from the default branch;
- `sha-<full Git commit>` for immutable deployments; and
- the Git tag, such as `v1.0.0`, for tagged releases.

The workflow authenticates with GitHub's repository-scoped `GITHUB_TOKEN`; no
registry secret needs to be added to the repository. Repository workflow
permissions must allow GitHub Actions to create packages.

The first published GHCR package is private by default. Either make the package
public in GitHub under **Packages > hermes-webui > Package settings**, or log the
Coolify deployment server into `ghcr.io` with a token that has `read:packages`.

## Docker Compose

Create a local `.env` from `.env.example`, set a strong `SEARXNG_SECRET`, and
start the stack:

```sh
cp .env.example .env
docker compose up -d
```

By default Compose pulls:

```text
ghcr.io/vaenae/hermes-webui:latest
```

For a reproducible deployment, set `HERMES_WEBUI_IMAGE` to a release or commit
tag before starting the stack:

```dotenv
HERMES_WEBUI_IMAGE=ghcr.io/vaenae/hermes-webui:sha-<full-git-commit>
```

SearXNG is reachable only inside the Compose network at
`http://searxng:8080`; no host port is published.

## Coolify

In the existing Coolify Compose service, replace the upstream WebUI image with
the published image while retaining the existing environment, health check,
domain, and volumes:

```yaml
hermes-webui:
  image: ghcr.io/vaenae/hermes-webui:latest
```

Use a release or `sha-*` tag instead of `latest` after validating the first
build. Replacing a container image does not remove the named Hermes volumes,
but important persistent data should still be backed up before deployment.

Do not put API keys or other runtime secrets in the Dockerfile or GitHub image.
Keep them in Coolify environment variables.
