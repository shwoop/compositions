# Migrating Compositions Stack from Docker Compose → k3s

**Assessment:** 30 Sep 2026
**Cluster:** k3s v1.37.0+k3s1, single node `manjaro` (192.168.1.108)
**Full implementation plan:** `docs/plans/2026-09-30-migrate-compositions-to-k3s.md`

Design and decision write-up for moving every per-folder docker-compose project
in this repo into a single k3s namespace `compositions`. Read this first; execute
the plan file task-by-task.

---

## TL;DR

- Same-host migration → **zero data copy**: docker stack and k3s both run on
  `manjaro` at `/home/alistair/repos/compositions` (git working copy, same HEAD
  as the Mac). Storage = hostPath pointed at the existing data dirs.
- Tooling mirrors the already-proven `../nzb` migration: plain YAML via
  Kustomize, `kubectl apply -k k8s/`, hostPath, IngressRoute CRDs, secrets via
  kustomize `secretGenerator`.
- **Traefik: use the k3s-bundled one** (see below) — it just needs to actually
  get installed; it never came up at cluster bring-up.
- DNS does not change: unbound's `a-records.conf` already pins every
  `*.ferguson.fr` subdomain to 192.168.1.108, and that node keeps serving
  80/443 whether docker or k3s Traefik owns the ports.
- Cutover is **per-app, sequential** — docker and k8s must never write the same
  pgdata/redisdata at the same time.

---

## What do we do about Traefik?

**k3s ships Traefik as its default ingress controller.** It is deployed as
bundled Helm charts (`traefik` + `traefik-crd`) into `kube-system` at first
boot, and configured via `HelmChartConfig`. We do **not** deploy a separate
Traefik — the docker one is retired.

**Caveat for this cluster:** the bundled addon was never actually installed.
The bootstrap HelmCharts exist (marked `BOOTSTRAP=True`) but there is no
Traefik Deployment and no pods — the install failed at cluster bring-up, before
the iptables fix (same story documented in `../nzb/MIGRATE-TO-K8S.md`). Ports
80/443 are currently free (docker traefik is stopped). So migration step 1
(Task 2 of the plan) is: force the addon install (delete the bootstrap
charts + restart helm-controller), with a fallback of `--disable traefik` +
official chart if it refuses.

**Config mapping (docker labels → k8s):**

| Docker Traefik today | k3s equivalent |
|---|---|
| `providers.docker` + `traefik.*` labels | IngressRoute CRDs (traefik.io/v1alpha1), one per host |
| `certificatesresolvers.le.acme.dnschallenge.provider=cloudflare` + `CF_DNS_API_TOKEN` env | HelmChartConfig `additionalArguments` + Secret `cf-dns-api-token` (kube-system) |
| `./letsencrypt/acme.json` | Traefik PVC (`/data/acme.json`, 256Mi local-path) |
| http→https redirect entrypoint | HelmChartConfig `ports.web.redirectTo.websecure.enabled` |
| dashboard via `api@internal` label | IngressRoute → `services: [{name: api@internal, kind: TraefikService}]` |
| pihole redirect-regex label | `Middleware` CRD (`redirectRegex`) |

We keep the **Cloudflare DNS challenge** (proven in docker for `ferguson.fr`),
not nzb's http-challenge variant.

---

## Current architecture (docker compose)

```
LAN clients ──▶ pihole (host :53) ──▶ unbound (a-records.conf → *.ferguson.fr = 192.168.1.108)
                                                │
public   ──▶ cloudflared tunnel ──▶ (origin) ──┴─▶ traefik (host :80/:443, CF DNS challenge)
                                                        │  docker "traefik" external network
                                  ┌─────────────────────┼─────────────────────────────┐
                              authentik            paperless+ai               immich
                           server/worker/db/redis   webserver/db/broker       server/ml/db/redis
                                                        │                            │
                                  vikunja+db   zerobyte (backs up paperless media    │
                                                 + immich images via hostPath)      │
                                  trala        pihole ✓     unbound ✓               ▼
```

Each compose dir (**authentik, cloudflared, immich, paperless, pihole, traefik,
vikunja, zerobyte**) is independent; several share the external `traefik` and
`tunnel` docker networks. `zerobyte` mounts `PAPERLESS_MEDIA_DIR` and
`IMMICH_IMAGES` to back them up.

## Target architecture (k3s)

```
one namespace: compositions
   Deployments:  pihole, unbound, trala,
                 authentik-server, authentik-worker, authentik-db, authentik-redis,
                 paperless-webserver, paperless-db, paperless-broker, paperless-ai,
                 immich-server, immich-machine-learning, immich-db, immich-redis,
                 vikunja, vikunja-db, zerobyte, cloudflared
   Services:     ClusterIP per app (names app-scoped → see name collisions below)
   IngressRoutes: one per host (websecure, tls resolver `le`)

traefik (kube-system, bundled) ──▶ Services ──▶ Pods
storage: hostPath → existing /home/alistair/repos/compositions/<app>/… (zero-copy)
secrets: kustomize secretGenerator from git-ignored k8s/secrets/*.env
```

### The one structural gotcha: name collisions

Every compose defines its own `db` / `redis` / `broker` service. In a single
namespace those collide, so all services/apps get app-scoped names and env vars
are rewritten to match (full table in plan Appendix A):

| docker env | k8s value |
|---|---|
| `AUTHENTIK_POSTGRESQL__HOST=db` | `authentik-db` |
| `AUTHENTIK_REDIS__HOST=redis` | `authentik-redis` |
| `PAPERLESS_DBHOST=db`, `PAPERLESS_REDIS=redis://broker:6379` | `paperless-db`, `redis://paperless-broker:6379` |
| `DB_HOSTNAME=db`, `REDIS_HOSTNAME=redis` (immich) | `immich-db`, `immich-redis` |
| `IMMICH_MACHINE_LEARNING_URL=http://machine-learning:3003` | `immich-machine-learning` |
| `VIKUNJA_DATABASE_HOST=db`, `FTLCONF_dns_upstreams=unbound#53` | `vikunja-db`, unchanged (`unbound` svc) |

---

## Compose → Kubernetes mapping

| Docker Compose | Kubernetes | Notes |
|---|---|---|
| `ports: HOST:CONTAINER` (http apps) | Service (ClusterIP) + `containerPort`; **drop host ports** | Traefik is in-cluster on host 80/443 |
| pihole `53:53/tcp+udp`, `5335:53` (unbound) | `hostPort` 53 / 5335 | LAN DNS must keep working; pihole2 on 192.168.1.11 may use :5335 |
| pihole web `90:80` | `hostPort: 90` (optional) | primary path = ingress `pihole.ferguson.fr` |
| `volumes: ./pgdata` etc. | `hostPath` → same absolute dir (patch file) | zero data migration; single node |
| `~/.config/rclone` (zerobyte) | `hostPath` `/home/alistair/.config/rclone` (ro) | |
| `${PAPERLESS_MEDIA_DIR}`, `${IMMICH_IMAGES}` | resolved absolute hostPaths (shared with zerobyte) | read from app `.env` at plan Task 1 |
| `cap_add: NET_ADMIN, CAP_SYS_TIME, CAP_SYS_NICE` (pihole) | `securityContext.capabilities.add` | |
| `traefik.*` labels | IngressRoute CRD (+ Middleware for pihole redirect) | |
| `depends_on: db (service_healthy)` (vikunja) | initContainer `pg_isready` wait | |
| `restart: unless-stopped` | Deployment `replicas: 1` | k8s self-heals |
| `.env` secrets in plaintext | kustomize `secretGenerator` from `k8s/secrets/*.env` (git-ignored) | bonus: secrets out of compose env |
| images pinned per compose | central `images:` block in `kustomization.yaml` | weekly `kubectl rollout restart` |

---

## Decisions (mirror ../nzb unless noted)

| # | Decision | Choice | Why |
|---|---|---|---|
| D1 | Tooling | Kustomize + `kubectl apply -k k8s/` | Same as nzb; no Helm/GitOps for one namespace, one host |
| D2 | Namespace | single `compositions` | requirement — apps talk by in-namespace DNS |
| D3 | Service naming | app-scoped (`authentik-db` …) | collision avoidance (see above) |
| D4 | Ingress | Traefik IngressRoute CRDs, entryPoint `websecure`, `tls.certResolver: le` | matches nzb; CRDs arrive with the addon |
| D5 | Traefik | k3s-bundled + HelmChartConfig, **Cloudflare DNS challenge** | proven config for `ferguson.fr`; works regardless of local DNS |
| D6 | Storage | hostPath zero-copy via `config-paths.patch.yaml` | data sits next to compose files on manjaro |
| D7 | Secrets | secretGenerator from git-ignored `k8s/secrets/*.env` | removes plaintext from `.env` workflow |
| D8 | cloudflared | migrate into k8s last; verify tunnel origin | remote-managed tunnel; only egress point moves |
| D9 | Cutover | per-app: `docker compose down` → apply → verify | never two writers on one pgdata |
| D10 | Probes/resource | minimal readiness probes + requests/limits | nzb's "better in k8s" list, kept light |

---

## Phases / tasks (execute the plan file for detail)

| Phase | Tasks | Deliverable |
|---|---|---|
| 0 Pre-flight | 1 | healthy cluster (iptables check), running-stack inventory, real data paths, pihole2/5335 |
| 1 Traefik | 2–3 | bundled addon **installed**; HelmChartConfig with CF DNS challenge; LE resolver up; 80/443 owned by k3s |
| 2 Scaffold | 4 | `k8s/` kustomize root, namespace `compositions`, secrets layout, validate+commit |
| 3 Apps | 5–12 | pihole+unbound, trala, authentik, paperless+ai, immich, vikunja, zerobyte, cloudflared |
| 4 Ingress | 13 | all IngressRoutes + `le` TLS; prove one host first |
| 5 Cutover | 14–15 | dry-run review; sequential flips; DNS flip last; smoke tests |
| 6 Decommission | 16 | docker compose down per app, remove `traefik`/`tunnel` networks, README update |

Full 16-task implementation plan with complete manifests, exact commands,
expected outputs, per-task commits, risk register and smoke-test checklist:
**`docs/plans/2026-09-30-migrate-compositions-to-k3s.md`**

## Storage

All stateful volumes → `hostPath` pointing at the existing compose dirs on
manjaro (single-node cluster ⇒ no RWX / pinning concerns). Single edit point:
`k8s/config-paths.patch.yaml` (same mechanism as nzb). Absolute paths are
recorded in plan Appendix B (some come from `.env` values — `PAPERLESS_MEDIA_DIR`,
`IMMICH_IMAGES`, `MODEL_CACHE`, `VIKUNJA_FILES` — fixed at Task 1).

## Secrets

Per-app `k8s/secrets/<app>.env` (git-ignored, copied from each app's existing
`.env`), consumed by kustomize `secretGenerator`. `cf-dns-api-token` is created
directly in `kube-system` (referenced by the Traefik HelmChartConfig).
Inventory + keys: plan Appendix C.

## Not in scope

- The nzb stack (gluetun/*arr, sabnzbd, jellyfin) — already migrated, stays in
  namespace `nzb`; keep it running in docker? No — it is *already running* in
  k8s; its docker containers (`docker -c manjaro ps`) were observed on 30 Sep
  and should be decommissioned by that project, not this one.
- `pihole2` on 192.168.1.11 — third-party DNS consumer; only dependency is this
  unbound's host port 5335 (verify, keep reachable).
- `/opencloud` (mentioned in README, no compose dir in the repo yet).

## Implementation status

- [ ] Cluster + host pre-flight (Task 1)
- [ ] Bundled Traefik addon installed (Task 2)
- [ ] Traefik HelmChartConfig + LE via Cloudflare DNS challenge (Task 3)
- [ ] kustomize scaffold (Task 4)
- [ ] pihole + unbound (Task 5)
- [ ] trala (Task 6)
- [ ] authentik (Task 7)
- [ ] paperless + paperless-ai (Task 8)
- [ ] immich (Task 9)
- [ ] vikunja (Task 10)
- [ ] zerobyte (Task 11)
- [ ] cloudflared (Task 12)
- [ ] IngressRoutes + TLS (Task 13)
- [ ] Dry-run review (Task 14)
- [ ] Cutover & smoke tests (Task 15)
- [ ] Docker decommission + docs (Task 16)