# Arcane — single-service production compose

One container, hardened, no bundled proxy. Put it behind the reverse proxy you
already run.

## Threat model, first

Arcane holds the Docker API, and the Docker API is equivalent to root on the
host. The hardening in `compose.yaml` limits what a container escape or a
compromised proxy gains — it does not make Arcane unprivileged. Run it on a
control-plane host, not beside tenant workloads.

## What each setting is for

`compose.yaml` carries no comments. This is the reference for it — read it
before loosening anything.

| Setting | Why it is there |
|---|---|
| `image:` a digest, not a tag | A tag is mutable and carries no supply-chain guarantee. |
| `BIND_ADDR=127.0.0.1` | Loopback-only publish. `0.0.0.0` puts the Docker control plane of the host on the network. |
| `*_FILE` secrets | Values never appear in `docker inspect` or the process environment. |
| `TRUSTED_PROXIES` | Only these sources may set `X-Forwarded-Proto`. Wrong value → the app believes it is on plain HTTP and issues the session cookie **without `Secure`**. Nothing visibly breaks; the session just stops being TLS-bound. |
| `ARCANE_SUBNET` fixed, not auto | A host-side proxy arrives from the gateway `.1` of this subnet, so `TRUSTED_PROXIES` must stay correct across recreates. |
| `PUID`/`PGID` | Arcane starts as root, prepares `/app/data`, then drops to this identity. |
| `group_add: DOCKER_GID` | The supplementary group that reaches the socket after the drop. |
| `/var/run/docker.sock` | Full Docker API access — the application's whole purpose, and the largest privilege in the file. |
| `cgroup: host` | Self-detection of its own container needs the host cgroup namespace. |
| `read_only: true` + `tmpfs /tmp` | Rootfs is immutable; the only writable paths are `/tmp`, `/app/data` and `/builds`. |
| `cap_drop: [ALL]` | Everything is dropped, then re-added one at a time below. |
| `cap_add: CHOWN, DAC_OVERRIDE, FOWNER, SETGID, SETUID` | Exactly what the startup path needs to chown the data roots and re-exec itself as `PUID:PGID`. **Nothing else.** Adding to this list widens what a container escape gains. |
| `no-new-privileges:true` | The dropped identity cannot regain privilege through setuid binaries. |
| `healthcheck: arcane health` | Reads `BASE_PATH` and probes the same prefix, so a sub-path deployment is not reported unhealthy while serving fine. |
| `arcane-builds` separate volume | Build contexts can be pruned without touching the database. |

## Install

`env.example` declares every value `compose.yaml` reads, with the default each
one falls back to. It is the only file you edit.

Every command below runs **from this directory**. `.env` is read from the
working directory, so `cd deploy/production` first — running elsewhere gives
`required variable ARCANE_IMAGE is missing a value`.

```bash
cd deploy/production
cp env.example .env
mkdir -p secrets && chmod 700 secrets
openssl rand -base64 48 | tr -d '\n' | head -c 43 > secrets/encryption_key
openssl rand -base64 48 | tr -d '\n' | head -c 43 > secrets/jwt_secret
chmod 600 secrets/*
```

Three values in `.env` have no usable default. `APP_URL` is whatever the
browser will use, sub-path and port included. The other two are read off this
host:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  alpine:3.21 stat -c '%g' /var/run/docker.sock

( set -a; . ./.env; set +a
  echo "ARCANE_IMAGE=ghcr.io/getarcaneapp/arcane@$(
    docker buildx imagetools inspect \
      "ghcr.io/getarcaneapp/arcane:${ARCANE_VERSION:?set ARCANE_VERSION in .env}" \
    | awk '/^Digest:/{print $2; exit}')" )

docker compose up -d
```

The `. ./.env` runs inside `( … )` on purpose. Compose gives shell variables
precedence over the `.env` file, so sourcing it into your own shell pins the
values as they were: every later edit to `.env` is silently ignored for the rest
of that session. If you have already hit that — `.env` looks right but Compose
keeps using an old value — run `unset ARCANE_IMAGE APP_URL DOCKER_GID` or start
a new shell.

`DOCKER_GID` is read from inside a container so it is right on Linux and on
Docker Desktop alike — `stat -c` is GNU-only and absent on macOS. The digest
line is printed ready to paste over `ARCANE_IMAGE=`; do not substitute
`--format '{{.Manifest.Digest}}'`, which does not resolve on a multi-arch index
and makes buildx silently print its full report instead.

`ARCANE_IMAGE` ships empty on purpose: an unresolved placeholder would pass
`docker compose config` and only fail at pull time with `invalid reference
format`. Empty makes it fail immediately, naming itself.

Arcane starts as root, prepares `/app/data`, then drops to `PUID:PGID` and
reaches the socket through the `DOCKER_GID` supplementary group.

## Two wiring modes

| | Mode A — proxy on the host | Mode B — proxy in a container |
|---|---|---|
| compose | `ports:` (default) | `expose: 3552` + `PROXY_NETWORK` |
| `.env` | `BIND_ADDR`, `BIND_PORT` | `PROXY_NETWORK`, no bind |
| proxy target | `http://$BIND_ADDR:$BIND_PORT` | `http://arcane:3552` |
| `TRUSTED_PROXIES` | leave unset — defaults to `ARCANE_SUBNET` | the proxy network's subnet |

Never set `BIND_ADDR=0.0.0.0`. That puts the Docker control plane of the host
on the network.

`TRUSTED_PROXIES` defaults to `ARCANE_SUBNET`, so Mode A needs only the one
value: change the subnet and proxy trust follows it. Set `TRUSTED_PROXIES`
explicitly only in Mode B, where the proxy arrives from a different network.

## Running a second stack on one host

`STACK_NAME`, `NETWORK_NAME`, `DATA_VOLUME` and `BUILDS_VOLUME` are fixed rather
than project-scoped so they survive `docker compose down`. The consequence: a
second copy of this stack started from another directory attaches to the *same*
volumes and both instances write one SQLite database. If you genuinely need two,
change all four — plus `BIND_PORT` and `ARCANE_SUBNET`.

## Wire it into your proxy

The prefix must reach Arcane **intact** — no stripping, in any proxy.

**nginx**

```nginx
location = /arcane { return 308 /arcane/; }

location /arcane/ {
    proxy_pass http://127.0.0.1:3552;   # no URI part, or the prefix is stripped

    # $http_host, not $host: $host drops a non-default port, and the WebSocket
    # origin check compares Origin against this header.
    proxy_set_header Host              $http_host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_http_version 1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection $connection_upgrade;   # needs the standard map

    proxy_buffering    off;                            # live streams
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    client_max_body_size 0;                            # image/project uploads
}
```

**Caddy** — `handle`, never `handle_path` (it strips the prefix):

```caddy
handle /arcane* {
	reverse_proxy 127.0.0.1:3552 {
		flush_interval -1
	}
}
```

**Traefik** — no `StripPrefix` middleware:

```yaml
http:
  routers:
    arcane:
      rule: "Host(`arcane.example.com`) && PathPrefix(`/arcane`)"
      service: arcane
  services:
    arcane:
      loadBalancer:
        passHostHeader: true
        servers:
          - url: "http://arcane:3552"
```

**Kubernetes ingress-nginx** — plain `path: /arcane`, `pathType: Prefix`, and no
`rewrite-target` annotation.

## The three rules that break sub-path hosting

1. **Never strip the prefix.** Arcane serves UI, API and WebSockets under
   `BASE_PATH` and must receive it.
2. **`APP_URL` must match what the browser uses** — same sub-path, host and
   port. OIDC redirect URIs and the WebSocket origin check are built from it.
3. **Forward `Host` with its port.** The origin check compares it against the
   browser's `Origin`. On `:443` this is free; on any other port nginx's `$host`
   turns every live stream into a 403.

## Verify

Every check reads `.env`, so none carry a hardcoded host, port or sub-path.
Directly against the container — expected results are `302` into `$BASE_PATH/`,
`200`, `404`, then `0` placeholders left in the served shell:

```bash
( set -a; . ./.env; set +a
  LOCAL="http://${BIND_ADDR:-127.0.0.1}:${BIND_PORT:-3552}"

  curl -so /dev/null -w '%{http_code} %{redirect_url}\n' "$LOCAL/"
  curl -so /dev/null -w '%{http_code}\n'                 "$LOCAL$BASE_PATH/api/health"
  curl -so /dev/null -w '%{http_code}\n'                 "$LOCAL/api/health"
  curl -s "$LOCAL$BASE_PATH/" | grep -c __ARCANE_BASE_PATH__ )

docker inspect "$(docker compose ps -q arcane)" \
  --format '{{.HostConfig.ReadonlyRootfs}} {{.HostConfig.CapDrop}} {{.HostConfig.SecurityOpt}}'
```

Through the proxy, two checks that fail silently if you skip them:

```bash
( set -a; . ./.env; set +a
ORIGIN=$(printf '%s' "$APP_URL" | sed -E 's#^(https?://[^/]+).*#\1#')

curl -s -o /dev/null -D - -X POST "$APP_URL/api/auth/login" \
  -H 'Content-Type: application/json' -d '{"username":"...","password":"..."}' \
  | grep -i set-cookie

curl -sk --http1.1 -o /dev/null -D - -b cookies.txt \
  -H 'Connection: Upgrade' -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Version: 13' \
  -H "Sec-WebSocket-Key: $(head -c16 /dev/urandom | base64)" \
  -H "Origin: $ORIGIN" \
  "$APP_URL/api/environments/0/ws/system/stats" | head -1 )
```

The cookie must come back as `__Host-token` with `Secure` once the proxy
terminates TLS; plain `token` means `TRUSTED_PROXIES` does not cover the proxy
and `X-Forwarded-Proto` is being ignored. The WebSocket must answer `101`, not
`403` — that is what catches a wrong `APP_URL` or a dropped `Host` port.

## Change the sub-path

Edit `BASE_PATH` and `APP_URL`, then `docker compose up -d`. No image rebuild:
the frontend ships a placeholder base that the backend substitutes as it serves
the assets.

## Upgrade, rollback, backup

```bash
( set -a; . ./.env; set +a
  docker run --rm -v "${DATA_VOLUME:-arcane-data}:/data:ro" -v "$PWD:/backup" alpine:3.21 \
    tar czf "/backup/${DATA_VOLUME:-arcane-data}-$(date +%F).tar.gz" -C /data . )

docker compose up -d
```

Upgrade and rollback are the same command; what changes is the `ARCANE_IMAGE`
digest in `.env`.

A downgrade needs `ALLOW_DOWNGRADE`, deliberately unset — restoring the backup
is safer and faster than a half-migrated database.

## Residual risks

| Risk | Disposition |
|---|---|
| Docker socket is root-equivalent | Accepted, inherent. Isolate the host. |
| Session cookie `Path=/` | Inherent: the `__Host-` prefix requires it. The cookie is still sent to every other app on the same origin — co-hosting untrusted apps means giving Arcane its own hostname instead of a sub-path. |
| Gravatar lookups | UI setting `enableGravatar`; off stops email-hash egress. |
| Telemetry heartbeat | `ANALYTICS_DISABLED=true`. |
| Registry checks reach the internet | Image-update discovery; use an internal mirror on a restricted-egress estate. |
