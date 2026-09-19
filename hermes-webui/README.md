# hermes-webui — custom image with a headless browser

Derived from `ghcr.io/nesquena/hermes-webui:0.52.302`, adding the pieces the
upstream image cannot install at runtime.

## What the base image is missing

| Missing | Consequence |
|---|---|
| A browser | Hermes' browser tooling fails closed; no page interaction, no JS-rendered pages |
| Node.js ≥ 24 | `agent-browser` (the CLI Hermes drives browsers with) declares `engines: { node: ">=24.0.0" }`; Debian Trixie ships 20.x |

Neither can be fixed after startup: the container drops from `root` to the
unprivileged `hermeswebui` user before serving, so `apt` is not available, and
Chrome-for-Testing — the build `agent-browser install` reaches for — publishes no
Linux **ARM64** artifact at all.

## Build

```bash
docker build -t hermes-webui-web:0.52.302 hermes-webui
```

## Run — `shm_size` is required

Chromium allocates far more than Docker's default 64 MB of shared memory, and
when it runs out it exits within about a second of starting, with no useful
error anywhere. `--disable-dev-shm-usage` is already set, but it does not cover
every allocation.

```yaml
services:
  webui:
    image: hermes-webui-web:0.52.302
    shm_size: 1gb            # ← without this the browser dies ~1s after launch
    ports:
      - "8787:8787"
    volumes:
      - ./hermes:/home/hermeswebui/.hermes
      - ./workspace:/workspace
```

## Verify

```bash
docker run --rm hermes-webui-web:0.52.302 chromium --version      # Chromium 153.x
docker run --rm hermes-webui-web:0.52.302 agent-browser --version # agent-browser 0.38.x
```

Inside a running container, the checks Hermes itself performs:

```bash
su - hermeswebui -c 'chromium --version && agent-browser --version'
```

## Notes and caveats

- **This image was not built during authoring** — no Docker daemon was available
  where it was written. Every external fact it depends on *was* verified
  individually: the base tag exists as a multi-arch manifest (amd64 + arm64), the
  `chromium` and `fonts-liberation` packages resolve in Debian Trixie, and the
  Node checksums are byte-for-byte from nodejs.org's `SHASUMS256.txt` for
  v24.21.0. The build itself is unproven.
- **`shm_size` diagnosis is a strong inference, not a measurement.** The browser
  was observed dying ~1 s after launch with a live CDP port, while the same binary
  and flags launched by hand kept running — so the executable, libraries and flags
  are fine, and the remaining suspect is the 64 MB `/dev/shm` default.
- **ARM64 specifics.** Trixie renamed several packages for the 64-bit `time_t`
  transition (`libxt6` → `libxt6t64`, `libatk1.0-0` → `libatk1.0-0t64`, and so on).
  Only `chromium` and `fonts-liberation` are needed for the browser itself; those
  renames matter if Firefox-based engines are added later.
- **`AGENT_BROWSER_EXECUTABLE_PATH` matters on arm64.** Without it, agent-browser
  looks for a Chrome-for-Testing build that does not exist for this architecture.
- Two optional additions are deliberately **not** in this image:
  - **Camofox / Camoufox** (Firefox-based anti-detection engine) additionally needs
    GTK3, `libdbus-glib-1-2`, `libxt6t64`, `libxmu6`, `libxtst6`, `libxss1`,
    `libxrender1`, `libpci3` and `libgdk-pixbuf-2.0-0`. It does **not** defeat
    IP-reputation blocks — Reddit and Glassdoor answer it `403` just as they do
    plain Chrome — so it is only worth adding if a Firefox fingerprint is
    specifically wanted.
  - **SearXNG** as a separate service, for keyless unlimited search. Worth doing
    because public SearXNG instances refuse the JSON API that Hermes needs.
