# SearXNG sidecar

A self-hosted SearXNG instance, so Hermes' `web_search` has a search backend that
needs no API key and has no quota.

## Why self-host instead of using a public instance

Public SearXNG instances are not usable as a backend. Fourteen were tested while
setting this up — `searx.be`, `searx.tiekoetter.com`, `search.inetol.net`,
`baresearch.org`, `priv.au`, `opnxng.com`, `paulgo.io`, `searx.work` and others —
and **every one of them refused the JSON API**: 403, 429, or an
anti-bot interstitial. Every provider comparison in this repo's companion PR also
assumes you control the instance, because `search.formats` is what gates JSON
output and you cannot change it on someone else's server.

## Why not a custom image

There is no reason to build one. `searxng/searxng` is maintained upstream and
publishes a multi-arch manifest that includes **arm64**; the only thing that needs
to change is configuration, and that is a mounted file. Rebuilding upstream would
add a maintenance burden and a supply chain of our own for zero gain.

## What `settings.yml` does

Four overrides, on top of the image's defaults:

| Setting | Why |
|---|---|
| `use_default_settings: true` | Loads SearXNG's own defaults first and deep-merges this file over them, so engines and everything else stay in step with the pinned image. Without it a `settings.yml` must be complete — the upstream file is ~3400 lines. |
| `search.formats: [html, json]` | **The setting this whole sidecar exists for.** Upstream ships `[html]` only, and an unlisted format answers 403 rather than falling back. |
| `server.limiter: false` | The instance is only reachable from the compose network. Turning the limiter on additionally requires the `valkey` service. |
| `general.instance_name` | Cosmetic, and useful as a check: if the UI shows it, the merge worked. |

`server.secret_key` in the file is a placeholder — the image's entrypoint
rewrites it from `$SEARXNG_SECRET`, which compose supplies from `.env`.

## Wiring

`SEARXNG_URL=http://searxng:8080` is set on the `webui` service. Point Hermes at
it once, inside the container:

```bash
docker compose exec webui hermes config set web.search_backend searxng
```

## Verify

```bash
docker compose up -d
docker compose exec searxng curl -s "http://localhost:8080/search?q=test&format=json" | head -c 200
```

A JSON body means the format override took. An HTML page or 403 means
`searxng/settings.yml` is not the file being served — check that the mount landed
at `/etc/searxng/settings.yml` (that path is the image's
`__SEARXNG_SETTINGS_PATH`).

## Caveats

- **The settings file was exercised, but not inside this image.** The same
  `settings.yml` was run against a locally built SearXNG from the source tree at
  **2026.9.18+c0042ad** and served JSON with 28 results in ~0.8 s, with the
  `instance_name` override visible in the UI — which proves the
  `use_default_settings` deep-merge. The pinned image here is **2026.9.8-3fdc6d753**.
  The merge lives in `searx/settings_loader.py` and there is no reason to expect
  a behaviour change across those two releases, but it was not verified on the
  image itself, because no Docker daemon was available.
- **The image tag is pinned for a reason.** `latest` tracks a project that does
  occasionally rename settings; the file is written against this version's merge
  behaviour. Bump deliberately.
- The host directory is mounted at `/etc/searxng`, so the container can write
  beside `settings.yml` (its entrypoint may create `limiter.toml` and similar).
  Expect those files to appear in this directory.
