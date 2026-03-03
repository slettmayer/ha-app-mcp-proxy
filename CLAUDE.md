# CLAUDE.md

## Project Overview

Home Assistant add-on that bridges stdio-based MCP servers (launched via `npx` or `uvx`) to SSE/StreamableHTTP endpoints using [mcp-proxy](https://github.com/sparfenyuk/mcp-proxy). Packaged as a Docker container running locally on HA.

## Repository Structure

```
ha-app-mcp-proxy/
├── repository.yaml                    # HA add-on repo manifest
├── .github/workflows/build.yaml       # CI/CD with HA builder
└── mcp-proxy/                         # The add-on
    ├── config.yaml                    # Add-on manifest (version, image, ports, options)
    ├── build.yaml                     # Base image per architecture
    ├── Dockerfile
    ├── CHANGELOG.md                   # Displayed in HA UI on updates
    ├── DOCS.md                        # User-facing documentation
    ├── translations/en.yaml           # UI labels
    └── rootfs/etc/
        ├── cont-init.d/
        │   └── mcp-proxy-init.sh      # Creates default config, validates JSON
        └── services.d/mcp-proxy/
            ├── run                    # Main daemon
            └── finish                 # Exit handler
```

## Release Workflow

Every release must update these three things:

1. **Bump version** in `mcp-proxy/config.yaml` — the `version` field, following semver
2. **Update `mcp-proxy/CHANGELOG.md`** — add a section for the new version with changes
3. **The `image` field in `config.yaml`** stays as `ghcr.io/slettmayer/mcp-proxy` (no tag) — HA appends the version automatically

After merging to main, create a GitHub release:
```bash
gh release create v<version> --target main --title "v<version>" --notes "..."
```

## HA Add-on Lessons Learned

### Image pulling vs local builds
- `config.yaml` **must** have an `image` field (e.g. `ghcr.io/slettmayer/mcp-proxy`) for HA to pull pre-built images from GHCR
- Without `image`, HA always builds locally from the Dockerfile

### s6 service scripts
- **Dockerfile `ENV PATH` is NOT available** inside s6 service scripts (`services.d/*/run`). Always use full paths to binaries installed via `uv tool` (e.g. `/usr/local/uv-tools/bin/mcp-proxy`)
- **`execlineb` scripts can't find s6 binaries** in HA's s6-overlay v3. Write `finish` scripts in bash with `#!/usr/bin/with-contenv bashio` instead
- Use `/run/s6/basedir/bin/halt` to take down the supervision tree (not `s6-svscanctl`)

### CI/CD (GitHub Actions)
- The HA builder (`home-assistant/builder@master`) needs `--docker-hub` and `--image` flags to form a valid Docker tag, even for `--test` builds — without them it produces `/:version`
- GHCR push requires a `docker/login-action` step with `GITHUB_TOKEN` before the builder step
- The workflow builds for both `aarch64` and `amd64` via matrix strategy

### Configuration
- MCP server config is a JSON file at `/addon-configs/mcp_proxy/servers.json` (mapped to `/config/servers.json` inside the container via `map: addon_config:rw`)
- HA UI options only handle `log_level` and `pass_environment` — deeply nested server config doesn't map well to HA's options schema

### Base image
- Uses Debian trixie (`ghcr.io/home-assistant/{arch}-base-debian:trixie`) for glibc compatibility with arbitrary third-party MCP servers
- `uv` and `uvx` are copied from `ghcr.io/astral-sh/uv:latest` (multi-arch)
- `UV_PYTHON_PREFERENCE=only-system` prevents uvx from downloading its own Python at runtime

## Branch Protection

- `main` requires PRs with passing CI checks — no direct pushes
