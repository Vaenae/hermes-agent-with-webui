# Custom Hermes WebUI image: adds a real headless browser to the upstream image.
#
# Why this exists
# ---------------
# The upstream image ships no browser. Without one, Hermes' browser tools fail
# closed, and the browser cannot be added at runtime: the container drops to the
# unprivileged `hermeswebui` user before serving, so apt is unavailable, and
# Chrome-for-Testing publishes no Linux ARM64 build (agent-browser's own
# `install` refuses on arm64). Everything below therefore has to be baked in.
#
# What it adds
# ------------
#   1. Chromium from the distro, so Hermes' browser tools find a browser.
#   2. Node.js 24 + agent-browser (the CLI Hermes drives the browser through).
#      Debian trixie ships Node 20, but agent-browser requires >= 24.
#   3. The two env vars that point agent-browser at the distro Chromium.
#
# shm_size: Chromium needs more than Docker's default 64 MB of shared memory.
# Set `shm_size: 1gb` on the service — see README.md.

FROM ghcr.io/nesquena/hermes-webui:0.52.302

USER root
ENV DEBIAN_FRONTEND=noninteractive

# ── Chromium ───────────────────────────────────────────────────────────────
# The `chromium` package pulls its own runtime stack (GTK3, NSS, cairo, pango,
# libasound2, libgbm, libxkbcommon, atk/atspi, cups). Installing it also gives
# Hermes a browser on PATH, which is where its own availability check looks.
# fonts-liberation is not pulled in by --no-install-recommends but is required
# for legible headless rendering.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        chromium \
        fonts-liberation \
    && rm -rf /var/lib/apt/lists/* \
    && chromium --version

# ── Node.js 24 ─────────────────────────────────────────────────────────────
# agent-browser declares `engines: { node: ">=24.0.0" }`; trixie's nodejs is 20.x.
# Official upstream build, checksum pinned to nodejs.org's published SHASUMS256.txt
# for v24.21.0. TARGETARCH is unset under a plain (non-BuildKit) build, so fall
# back to the host architecture.
ARG NODE_VERSION=24.21.0
ARG TARGETARCH
RUN set -eux; \
    arch="${TARGETARCH:-$(dpkg --print-architecture)}"; \
    case "$arch" in \
        amd64) node_arch=x64; \
               sha256=fd8e59d5a511510f6a298afb548f18c7d2b1be404d8b4a27d94fbe49f56cb2d6 ;; \
        arm64) node_arch=arm64; \
               sha256=6ad1325edbdb5649c379b75a237147a666c95d4f9ae8d340fef2d1575d289ad2 ;; \
        *) echo "unsupported architecture: $arch" >&2; exit 1 ;; \
    esac; \
    curl -fsSL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-${node_arch}.tar.xz" \
        -o /tmp/node.tar.xz; \
    echo "${sha256}  /tmp/node.tar.xz" | sha256sum -c -; \
    tar -xJf /tmp/node.tar.xz -C /usr/local --strip-components=1 --no-same-owner; \
    rm -f /tmp/node.tar.xz; \
    node --version; \
    npm --version

# ── agent-browser ──────────────────────────────────────────────────────────
# Installed into npm's global prefix (/usr/local/bin), which is on PATH *and* in
# the list of directories Hermes searches for the browser CLI. Do NOT install it
# into $HERMES_HOME/node: that path is a bind-mounted volume at runtime, so
# anything baked there at build time would be shadowed by the mount.
# --allow-scripts runs agent-browser's postinstall; the fallback covers npm
# versions that predate the flag.
RUN npm install -g --allow-scripts=agent-browser agent-browser \
    || npm install -g agent-browser

# ── Point agent-browser at the distro Chromium ─────────────────────────────
# Without an explicit executable agent-browser looks for Chrome-for-Testing,
# which has no Linux ARM64 build. --no-sandbox is required because Chromium
# refuses to run as root otherwise, and --disable-dev-shm-usage keeps it off
# the small default /dev/shm.
ENV AGENT_BROWSER_EXECUTABLE_PATH=/usr/bin/chromium
ENV AGENT_BROWSER_ARGS=--no-sandbox,--disable-dev-shm-usage

# Fail the build, loudly, if any of the three pieces is not actually runnable.
RUN chromium --version \
    && node --version \
    && agent-browser --version

# The base image already ends with `USER root` and CMD ["/hermeswebui_init.bash"].
# Both are inherited: the entrypoint does root-only bind-mount preparation and
# then drops to `hermeswebui` before starting the server.
