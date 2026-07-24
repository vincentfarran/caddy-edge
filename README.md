# caddy-edge

Custom [Caddy](https://caddyserver.com/) build for the fleet's public edge — a single
binary carrying **both** plugins the HK edge needs:

- [`github.com/mholt/caddy-l4`](https://github.com/mholt/caddy-l4) — Layer-4 (TCP/UDP)
  routing (the existing HK edge routes depend on it).
- [`github.com/sablierapp/sablier/plugins/caddy`](https://github.com/sablierapp/sablier) —
  the [Sablier](https://sablierapp.dev/) scale-to-zero handler (`http.handlers.sablier`),
  which warms the on-demand `authentik@hk` group on first request.

It supersedes the two half-images this replaced: `caddy-l4` (layer4 only) and
`caddy-sablier` (sablier only). A Sablier Caddy plugin is compiled in, not loaded at
runtime, so both capabilities must live in one binary.

## Image

`ghcr.io/vincentfarran/caddy-edge`

| Tag | Meaning |
|-----|---------|
| `latest` | most recent `main` build |
| `sha-<commit>` | build for a specific commit |
| `@sha256:…` | immutable digest — **this is what downstream pins** |

Consumers (e.g. `compose/caddy@hk` in `infra-ops`) reference the image **by digest**, never
by tag, so a new build is only adopted by a deliberate one-line bump.

## Pinned build tuple

Everything is pinned in [`Dockerfile.caddy`](./Dockerfile.caddy) so a rebuild is
reproducible — an unpinned `xcaddy build` would resolve the current Caddy release and each
module's latest, which on an edge image is silent drift. Values were read from
`caddy build-info` on the proven binaries, not guessed:

| Component | Pin |
|-----------|-----|
| Caddy core | `v2.11.4` (builder base `caddy:2.11.4-builder`, `xcaddy build v2.11.4`, runtime `caddy:2.11.4-alpine`) |
| layer4 | `github.com/mholt/caddy-l4@v0.1.2` |
| sablier | `github.com/sablierapp/sablier/plugins/caddy@v0.0.0-20251109203149-96750a50da79` |

> ⚠️ The Sablier plugin lives in the Sablier **monorepo** at `.../sablier/plugins/caddy`.
> The separate `github.com/sablierapp/sablier-caddy-plugin` repo also exists but is **not**
> the module the proven binary was built from — use the monorepo path.

The build **self-verifies**: `Dockerfile.caddy` fails the build if the core version drifts
or either plugin is missing, so a broken image is never pushed.

## Build & publish

[`.github/workflows/publish.yml`](./.github/workflows/publish.yml) builds and pushes on:
push to `main`, manual `workflow_dispatch`, and a weekly cron (Mon 03:17). The weekly run
picks up base-image (alpine) security patches; the caddy binary itself stays reproducible
because core + plugins are pinned. Each run logs the pushed digest (**Show pushed digest**
step) — that is the value to pin downstream.

## Verify a build

```bash
docker run --rm ghcr.io/vincentfarran/caddy-edge:latest caddy version                          # v2.11.4
docker run --rm ghcr.io/vincentfarran/caddy-edge:latest caddy list-modules | grep -i sablier   # http.handlers.sablier
docker run --rm ghcr.io/vincentfarran/caddy-edge:latest caddy list-modules | grep -c '^layer4' # 43
docker inspect ghcr.io/vincentfarran/caddy-edge:latest --format '{{index .RepoDigests 0}}'     # digest to pin
```

## Bumping

To change a version: edit the pin in `Dockerfile.caddy`, push to `main`, let the Action
build, read the digest from its log, then bump the consumer (`compose/caddy@hk` `image:`)
to the new `@sha256:…` in a separate PR. Rebuild reproducibly; pin downstream by digest.
