# Running Talon in Docker

The root `Dockerfile` builds the production image and `docker-compose.yml`
runs it. The image runs as UID 1000 with `HOME=/home/bun`, and all persistent
state lives in bind mounts:

| Container path        | What it holds                                                     |
| --------------------- | ----------------------------------------------------------------- |
| `/home/bun/.talon`    | Config, sessions, workspace, memory, bridge keys                  |
| `/home/bun/.claude`   | Claude Code credentials (Claude backend)                          |
| `/home/bun/.gemini`   | Antigravity OAuth cache + shared MCP config (agy backend)         |

```bash
docker compose up -d --build
docker compose logs -f talon
```

See [`packaging/README.md`](../packaging/README.md#docker-image) for how the
image itself is built.

## Antigravity (`agy`) backend

The image ships the tools agy shells out to (`git`, `ripgrep`). Two things
have to come from you: the `agy` binary (there is no npm package to install
it from) and a one-time Google sign-in (there is no API key).

### 1. Provide the binary

Pick one:

- **Bind-mount the host's binary** (the default in `docker-compose.agy.yml`).
  If `agy` isn't at `/usr/local/bin/agy` on the host, point at it:

  ```bash
  export AGY_BINARY_HOST="$(command -v agy)"
  ```

- **Bake it into the image.** Pass the URL of the Linux binary for your
  architecture and its SHA-256. The build verifies the digest and fails on a
  mismatch:

  ```bash
  AGY_DOWNLOAD_URL=https://…/agy-linux-amd64 \
  AGY_SHA256=<sha256> \
  docker compose -f docker-compose.yml -f docker-compose.agy.yml build
  ```

  Then delete the `/usr/local/bin/agy` mount line from
  `docker-compose.agy.yml`, because a mount there would hide the baked copy.

Either way the binary ends up at `/usr/local/bin/agy`, which is on `PATH`,
so no `agyBinary` / `AGY_BINARY` setting is needed.

### 2. Sign in once

agy caches its OAuth token at
`~/.gemini/antigravity-cli/antigravity-oauth-token`, and headless runs reuse
it. The compose override mounts the host's `~/.gemini`, so:

- **Signed in on the host already?** Nothing to do. The container reuses the
  cache.
- **No browser on the host?** Sign in with `agy` on any desktop, then copy
  `~/.gemini/antigravity-cli/antigravity-oauth-token` into the mounted
  `~/.gemini/antigravity-cli/` on the Docker host (owner UID 1000, mode
  `0600`).
- **Or try it in the container:** `docker compose exec -it talon agy`. If the
  CLI prints a sign-in URL you can open elsewhere, finish there. If it can
  only open a local browser, use the copy route above.

### 3. Run it

```bash
docker compose -f docker-compose.yml -f docker-compose.agy.yml up -d --build
```

and set `"backend": "agy"` in `~/.talon/config.json`, or switch per chat with
`/model`. `docker compose exec talon bun src/cli.ts doctor` (or
`node --import tsx src/cli.ts doctor` on the Node image) checks the binary,
its version, the cached sign-in and the model list.

`~/.gemini/config/mcp_config.json` is **shared** with any agy you run on the
host. Talon only writes keys under its own `__talon__` prefix and preserves
everything else. But a Talon on the host and one in the container, both on
agy, prune each other's entries at startup. See
[`docker/agy-test/README.md`](../docker/agy-test/README.md#coexistence-with-production).
