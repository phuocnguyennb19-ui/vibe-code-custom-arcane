# deploy/

Everything an operator needs to run Arcane, in one place. Nothing in here is
used by the build, the test suite, or CI — it is safe to read top to bottom and
copy out.

```
deploy/
├── production/     the hardened deployment: one service, your own proxy
└── examples/       minimal starting points, one per topology
```

## Which one do I want?

| Situation | Start from |
|---|---|
| Production, behind a proxy you already run | `production/compose.yaml` |
| Trying Arcane out on a laptop | `examples/compose.basic.yaml` |
| Behind a reverse proxy, served at the origin root | `examples/compose.proxy.yaml` |
| Served from a URL sub-path (`https://host/arcane`) | `examples/compose.subpath.yaml` + `examples/nginx.subpath.conf` |
| Adding a remote Docker host as an agent | `examples/compose.agent.yaml` |
| Agent over an outbound-only edge tunnel | `examples/compose.agent-edge.yaml` |

The examples are deliberately minimal — they show one topology each and skip
hardening so the topology stays readable. `production/` is the one to deploy:
pinned digest, Docker secrets, read-only rootfs, dropped capabilities,
non-root runtime identity, resource ceilings. See `production/README.md`.

## The three rules for sub-path hosting

They cause every failure worth knowing about, so they are repeated wherever
they apply:

1. **Never strip the prefix at the proxy.** nginx: `proxy_pass` with no URI
   part. Caddy: `handle`, never `handle_path`. Traefik: no `StripPrefix`.
   ingress-nginx: no `rewrite-target`.
2. **`APP_URL` must carry the same sub-path, host and port the browser uses.**
   OIDC redirect URIs and the WebSocket origin check are built from it.
3. **Forward `Host` with its port.** nginx's `$host` drops a non-default port
   and turns every live stream into a 403; use `$http_host`.

## What is deliberately NOT here

These files stay where their tooling expects to find them. Moving them breaks
something concrete:

| Path | Why it cannot move |
|---|---|
| `docker/Dockerfile*` | Build inputs. Referenced by `Justfile`, `.goreleaser.yaml`, `docker/compose.dev.yaml` and `tests/setup/*.yaml`. |
| `docker/compose.dev.yaml` | Development stack, documented throughout `CONTRIBUTING.md`. |
| `tests/setup/compose*.yaml` | Test fixtures selected by the CI matrix (`COMPOSE_FILE: setup/…`). |
| `.devcontainer/` | The path is mandated by the Dev Containers spec. |

The split is by audience, not by file type: `deploy/` is for whoever runs
Arcane, `docker/` is for whoever builds it, `tests/` is for CI.
