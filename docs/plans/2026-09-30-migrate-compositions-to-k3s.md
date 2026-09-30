# Migrate Compositions Stack from Docker Compose → k3s — Implementation Plan

> **REQUIRED SUB-SKILL:** Use the executing-plans skill to implement this plan task-by-task.

**Goal:** Run every service in this repo (pihole+unbound, traefik dashboard+trala, authentik, paperless+paperless-ai, immich, vikunja, zerobyte, cloudflared) as k3s manifests in a single `compositions` namespace, replacing the per-folder docker-compose projects with zero data migration.

**Architecture:** Mirror the already-proven `nzb` migration (/home/alistair/repos/nzb): plain YAML via Kustomize applied with `kubectl apply -k k8s/`, one namespace (`compositions`), hostPath storage pointed at the existing compose data dirs on the same host (manjaro), Traefik from the k3s-bundled addon (configured via HelmChartConfig) with IngressRoute CRDs replacing docker labels. No Helm, no GitOps — same tooling decisions as nzb.

**Tech Stack:** k3s v1.37+k3s1 (single node, manjaro @ 192.168.1.108), kubectl + kustomize (built in), Traefik v3 (bundled chart, HelmChartConfig-configured), Let's Encrypt via Cloudflare DNS challenge, hostPath volumes, kubernetes Secrets (kustomize secretGenerator), git workflow (author on Mac → push → pull on manjaro → apply).

---

## 0. Ground truth (verified 30 Sep 2026 — do not re-derive, but re-verify in Task 1)

- **Cluster:** k3s v1.37.0+k3s1 on `manjaro` (192.168.1.108, single control-plane node, containerd 2.3.4-k3s1). Reachable from the Mac: `kubectl` (context `default` points at it) and `ssh alistair@manjaro.local`.
- **Docker stack lives on the same host** at `/home/alistair/repos/compositions` (git working copy, same origin `git@github.com:shwoop/compositions.git`, HEAD 58200ea as the Mac copy). Docker daemon accessible as `docker -c manjaro`. → **zero-copy hostPath migration**, exactly like nzb.
- **Currently running in docker on manjaro:** `authentik-server/worker/db/redis`, `pihole-pihole-1` + `pihole-unbound-1`, `paperless-webserver/db/paperless-ai`, `zerobyte`, and the nzb stack (jellyfin, gluetun+sonarr+radarr+bazarr, sabnzbd — **not in scope**, keep running).
- **NOT currently running in docker (verify at cutover):** immich (server/ml/db/redis), vikunja (app/db), traefik, trala, cloudflared. Their compose dirs exist.
- **DNS topology (must survive migration unchanged):**
  - LAN clients → pihole (host port **53/tcp+udp**) → unbound (upstream, container port 53).
  - unbound `a-records.conf` pins `*.ferguson.fr` A → `192.168.1.108` (the node). **DNS records do not change during migration** — the node IP keeps serving 80/443 whether docker traefik or k3s traefik owns it.
  - `pihole2.ferguson.fr` = second pihole on 192.168.1.11; **may query this unbound's published host port 5335** → keep 5335 reachable (verify).
  - Public traffic → Cloudflare tunnel (cloudflared, remote-managed via TUNNEL_TOKEN) → Lan-bound origin. The tunnel's origin URLs (defined in the Zero Trust dashboard, not locally) must be re-verified for the k8s cloudflared task.
- **Traefik today (docker):** traefik v3, ports 80:80/443:443, Cloudflare **DNS challenge** (`--certificatesresolvers.le.acme.dnschallenge.provider=cloudflare`, `CF_DNS_API_TOKEN` env, storage `./letsencrypt/acme.json`), http→https redirect, dashboard at `traefik.ferguson.fr` via `api@internal`. All app routing is docker-provider labels — these all move to IngressRoute CRDs.
- **Cluster Traefik state (the "doesn't k3s come with traefik" question, answered):** k3s *ships* a bundled Traefik addon (HelmCharts `traefik` + `traefik-crd` are visible in `kube-system` marked `BOOTSTRAP=True`), but on this cluster **it was never installed** — no traefik Deployment and no traefik pods exist (matches the nzb doc: "helm install failed at cluster bring-up", before the iptables fix). Nothing is listening on 80/443 right now (docker traefik is stopped), so the addon can be brought up without a port conflict. → Phase 1 exists to get the bundled Traefik *actually running*; everything after depends on it.
- **Cluster health caution:** `local-path-provisioner` (1853 restarts) and `metrics-server` (1847 restarts) have crashlooped historically (iptables split-brain era). Health must be re-verified in Task 1 before we trust the cluster.

### Port inventory (host ports currently consumed by docker)

| Host port | Owner (docker) | k8s equivalent |
|---|---|---|
| 53/tcp+udp | pihole | pihole `hostPort: 53` |
| 5335/tcp+udp | unbound | unbound `hostPort: 5335` (verify pihole2 need) |
| 90/tcp | pihole web | pihole `hostPort: 90` (optional; primary path = ingress) |
| 80,443/tcp | traefik | bundled k3s Traefik (hostPort, chart default) |
| 2283 | immich | **drop** — ingress only |
| 3000 | paperless-ai | **drop** — ingress only |
| 9000,9443 | authentik | **drop** — ingress only (svc :9000) |
| — | all others | no host ports, ClusterIP + IngressRoute |

---

## 1. Decisions (fixed up front — mirror nzb unless noted)

| # | Decision | Choice | Why |
|---|---|---|---|
| D1 | Tooling | Kustomize + `kubectl apply -k k8s/`, single kustomize root | Same as nzb; ~19 services, one namespace, one host; Helm/GitOps add nothing |
| D2 | Namespace | single `compositions` | Per user requirement; apps reach each other by in-namespace DNS |
| D3 | Service names | **app-scoped** (`authentik-db`, `immich-db`, …, `paperless-broker`) | **Collisions:** each compose has its own `db`, `redis`, `broker`; in one namespace all must be unique. This is the main mechanical delta vs nzb |
| D4 | Ingress | Traefik IngressRoute CRDs (traefik.io/v1alpha1), entryPoint `websecure`, `tls.certResolver: le` | Matches nzb; the CRDs arrive with the bundled addon |
| D5 | Traefik | k3s-bundled addon + HelmChartConfig; **Cloudflare DNS challenge** (not nzb's httpchallenge) | Current docker config is proven for `ferguson.fr`; token-based DNS challenge works regardless of local DNS |
| D6 | Storage | `hostPath` zero-copy; paths in one `config-paths.patch.yaml` like nzb | Data lives next to the compose files at `/home/alistair/repos/compositions/…` |
| D7 | Secrets | kustomize `secretGenerator` from gitignored `k8s/secrets/*.env` (nzb pattern) | Bonus: removes plaintext secrets from `.env` handling |
| D8 | cloudflared | migrate into k8s **last**, keep origin behavior | Tunnel is remote-managed; only the egress point moves to a pod on the same node |
| D9 | Cutover | per-app: stop docker app stack → apply k8s app → verify → next | hostPath data dirs must never be written by docker *and* k8s at once (postgres double-open = corruption); DNS needs no change |
| D10 | Probes/resources | minimal readiness probes + `requests`/`limits` where compose had limits | nzb called these out as "better in k8s"; keep light, add later if wanted |
| D11 | Images | central kustomize `images:` block; `imagePullPolicy: Always`; weekly `kubectl rollout restart` | Same as nzb |

---

## 2. Workflow & validation loop (apply to every task)

- **Author on the Mac** (this repo), **apply on manjaro**: `git add -A && git commit && git push` → on manjaro: `cd /home/alistair/repos/compositions && git pull` → validate/apply there.
- Validation loop for any change (never `apply` blind):
  ```bash
  kubectl apply --dry-run=client -k k8s/     # renders + client-side checks, no cluster change
  kubectl diff -k k8s/                        # what would change
  kubectl apply -k k8s/                       # the actual change
  ```
  Commands in this plan assume they run **on manjaro** unless prefixed `[Mac]`.
- Commits: one per task (message `chore(k8s): <task>`).

---

## Task 1: Cluster + host pre-flight

**Files:** none (read-only checks).

**Step 1: Verify k3s health** `[on manjaro]`

```bash
kubectl get nodes -o wide
kubectl get pods -A                          # expect all Running, no CrashLoopBackOff
kubectl -n kube-system get pods              # coredns/local-path-provisioner/metrics-server 1/1 Running
```
Expected: node `Ready`, `v1.37.0+k3s1`; all pods Running. If addons are crash-looping with `dial tcp 10.43.0.1:443`, apply the iptables legacy fix documented at the top of `/home/alistair/repos/nzb/MIGRATE-TO-K8S.md` (symlink `iptables-restore`/`iptables-save` → legacy variants, `sudo systemctl restart k3s`, re-verify).

**Step 2: Confirm no more cluster problems** `[on manjaro]`

```bash
curl -s --max-time 4 -o /dev/null -w '%{http_code}\n' http://10.43.0.10/   # CoreDNS svc → 200/404 (connectivity ok)
iptables-restore -V     # must say "(legacy)"
```

**Step 3: Inventory what's actually running** `[Mac]`

```bash
docker -c manjaro ps --format '{{.Names}}\t{{.Image}}'
ss -tlnp | grep -E ':80 |:443 |:53 |:5335'      # [on manjaro] port ownership
```
Expected/record: which compose folders are up (autentik, pihole+unbound, paperless, zerobyte; immich/vikunja/traefik/trala/cloudflared appeared stopped on 30 Sep — confirm), and that 80/443 are free (docker traefik stopped) so the bundled Traefik can bind.

**Step 4: Record the real data paths** `[on manjaro]` — these feed `config-paths.patch.yaml` in Task 4/5:

```bash
cd /home/alistair/repos/compositions
grep -hE '^(IMMICH_IMAGES|MODEL_CACHE|PAPERLESS_MEDIA_DIR|VIKUNJA_FILES)=' immich/.env paperless/.env vikunja/.env
ls -d */pgdata */redisdata */data */media */export */consume */model-cache */upload */files */var-lib-zerobyte */-data 2>/dev/null
```
Expected: absolute paths like `/drives/…` or `/home/alistair/…` and the set of data dirs per app. Record them into the table in Appendix B.

**Step 5: Verify pihole2 dependency** `[on manjaro]`:

```bash
dig @192.168.1.11 +short ferguson.fr   # or ask: does pihole2 use this unbound's 5335? grep its config if accessible
```
If pihole2 upstreams to this host's 5335 → keep unbound `hostPort: 5335` (planned anyway). If unused, drop it.

**Commit:** `chore: pre-flight notes for k3s migration` (README of findings under `docs/` only, or skip commit if nothing written — record findings in this plan's Appendix B).

---

## Task 2: Get the bundled Traefik addon installed

> This is the answer to "what do we do about traefik": we use the one k3s ships — we just have to get it started, because it never came up at bring-up.

**Files:**
- Verify/modify (on manjaro): none in repo; cluster state only.

**Step 1: Diagnose why the addon is absent** `[on manjaro]`

```bash
kubectl -n kube-system get helmchart traefik traefik-crd        # exist? (observed: yes, BOOTSTRAP=True)
kubectl -n kube-system get deploy,svc,pods -l app.kubernetes.io/name=traefik   # nothing (observed)
kubectl -n kube-system logs deploy/helm-controller --tail=100    # look for traefik chart install errors
```
Expected: helmcharts exist but no workload — the chart-controller failed at boot and hasn't retried.

**Step 2: Force the bootstrap charts to (re)install** `[on manjaro]`

```bash
kubectl -n kube-system delete helmchart traefik traefik-crd        # k3s recreates on controller restart
kubectl -n kube-system rollout restart deploy/helm-controller
kubectl -n kube-system get pods -w                                  # watch for traefik-xxx pod(s)
```
Expected: traefik Deployment + pods appear in kube-system; `kubectl -n kube-system get svc traefik` shows NodePort; the traefik pod binds **host ports 80/443** (chart default) — if 80/443 were occupied the pod would fail `bind: address already in use` (see Task 1 confirmation they're free; if docker traefik has since started, stop it: `docker -c manjaro stop traefik`).

**Step 3: Verify CRDs and ingress class** `[on manjaro]`

```bash
kubectl get crd | grep traefik                 # IngressRoute, Middleware, TraefikService ... (traefik.io/v1alpha1)
kubectl get ingressclass                       # traefik, default
kubectl -n kube-system logs deploy/traefik --tail=30   # url prefix @ http://+:80, http://+:443
```

**Fallback if the addon refuses to install** (only if Step 2 ×2 fails): add `--disable traefik --disable traefik-crd` to `/etc/rancher/k3s/config.yaml`, restart k3s, and deploy the official traefik chart manually (`helm repo add traefik …` — requires installing helm) using the same HelmChartConfig values from Task 3. Do not spend more than ~30 min on the addon; the fallback is mechanical.

**Commit:** nothing in repo (cluster-only). Record outcome in this plan's checklist.

---

## Task 3: Configure bundled Traefik (LE via Cloudflare DNS challenge)

**Files:**
- Create: `k8s/optional/traefik-le.yaml` (HelmChartConfig — adapted from nzb's `optional/traefik-le.yaml`, which used httpchallenge; **we keep the docker-proven Cloudflare DNS challenge**)
- Create: `k8s/secrets/cf-dns-api-token.env` (git-ignored — copy value from `traefik/.env`'s `CF_DNS_API_TOKEN`)
- Modify: `k8s/.gitignore` (add `k8s/secrets/`)

**Step 1: Write the HelmChartConfig** `[Mac]`

```yaml
# k8s/optional/traefik-le.yaml
# OPT-IN: configure the k3s-bundled Traefik for Let's Encrypt via the
# Cloudflare DNS challenge (same as the old docker traefik).
#
#   kubectl create secret generic cf-dns-api-token \
#     -n kube-system --from-env-file=k8s/secrets/cf-dns-api-token.env
#   kubectl apply -f k8s/optional/traefik-le.yaml     # restarts the cluster's Traefik
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    additionalArguments:
      - "--certificatesresolvers.le.acme.email=admin@ferguson.fr"
      - "--certificatesresolvers.le.acme.storage=/data/acme.json"
      - "--certificatesresolvers.le.acme.dnschallenge=true"
      - "--certificatesresolvers.le.acme.dnschallenge.provider=cloudflare"
      - "--certificatesresolvers.le.acme.dnschallenge.resolvers=1.1.1.1:53,8.8.8.8:53"
    envFrom:
      - secretRef:
          name: cf-dns-api-token        # CF_DNS_API_TOKEN (see above)
    ports:
      web:
        redirectTo:
          websecure:
            enabled: true               # http → https, same as docker traefik
    persistence:
      enabled: true
      storageClass: local-path
      size: 256Mi                       # keeps acme.json across restarts (LE rate limits)
```

**Step 2: Create the token secret + apply** `[on manjaro]`

```bash
cp k8s/secrets/cf-dns-api-token.env.example k8s/secrets/cf-dns-api-token.env   # fill from traefik/.env
kubectl create secret generic cf-dns-api-token -n kube-system \
  --from-env-file=k8s/secrets/cf-dns-api-token.env --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f k8s/optional/traefik-le.yaml
```

**Step 3: Verify** `[on manjaro]`

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=traefik -w        # rollout restart, then 1/1 Running
kubectl -n kube-system logs deploy/traefik --tail=30 | grep -i "le\b\|certificatesresolvers"  # resolver registered
kubectl -n kube-system exec deploy/traefik -- env | grep CF_DNS_API_TOKEN    # token present (value redacted)
curl -s -o /dev/null -w '%{http_code}\n' https://192.168.1.108/              # 404 from traefik (self-signed ok)
```
Expected: traefik 1/1 Running, `le` resolver in logs, `CF_DNS_API_TOKEN` set, and 80/443 answering from k3s. A `404` on https://node is *good* (no routes yet).

**Commit:** `chore(k8s): traefik HelmChartConfig with cloudflare LE` (+ `.gitignore` update).

---

## Task 4: Scaffold the kustomize project

**Files:**
- Create: `k8s/namespace.yaml`
- Create: `k8s/kustomization.yaml`
- Create: `k8s/config-paths.patch.yaml` (initially a stub; filled per-app in later tasks)
- Create: `k8s/secrets/.gitkeep` + `k8s/secrets/*.env.example` per app
- Modify: `.gitignore` (add `/k8s/secrets/`)

**Step 1: namespace** `[Mac]`

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: compositions
```

**Step 2: kustomization root** — starts with namespace only; each Task adds its app to `resources`.

```yaml
# k8s/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: compositions

resources:
  - namespace.yaml
  # apps get added here, one line each, as their tasks land:
  # - pihole.yaml        (Task 5)
  # - trala.yaml         (Task 6)
  # - authentik.yaml     (Task 7)
  # - paperless.yaml     (Task 8)
  # - immich.yaml        (Task 9)
  # - vikunja.yaml       (Task 10)
  # - zerobyte.yaml      (Task 11)
  # - cloudflared.yaml   (Task 12)
  # - ingress.yaml       (Task 13 — enable only after Traefik+LE are verified)

patches:
  - path: config-paths.patch.yaml    # single place for absolute hostPath dirs on manjaro

secretGenerator:
  # added per-app as each task lands (files git-ignored under k8s/secrets/)

images:
  # central tag bumps, filled in as apps land (nzb pattern)
  - name: pihole/pihole
    newTag: latest
```

**Step 3: secrets layout** — one env file per app; kustomize turns `KEY=VALUE` lines into Secret data:

```
k8s/secrets/
  authentik.env.example     # AUTHENTIK_SECRET_KEY, AUTHENTIK_POSTGRESQL__USER/PASSWORD/…
  paperless.env.example     # POSTGRES_PASSWORD, PAPERLESS_DBPASS, PAPERLESS_SECRET_KEY
  immich.env.example        # DB_PASSWORD, DB_USERNAME, DB_DATABASE_NAME, OAUTH_CLIENT_ID, OAUTH_CLIENT_SECRET
  vikunja.env.example       # DB_PASSWORD, VIKUNJA_DATABASE_PASSWORD, VIKUNJA_SERVICE_JWTSECRET
  pihole.env.example        # FTLCONF_webserver_api_password
  cloudflared.env.example   # TUNNEL_TOKEN
  cf-dns-api-token.env.example  # CF_DNS_API_TOKEN (kube-system; Task 3)
```
Copy each from the app's existing git-ignored `.env` when the app task runs. `cp <app>/.env k8s/secrets/<app>.env` then prune keys not needed in-cluster; keep values identical.

**Step 4: validate + commit** `[on manjaro]` after `git pull`

```bash
kubectl apply --dry-run=client -k k8s/     # expect: Namespace only
git add -A && git commit -m "chore(k8s): scaffold kustomize root + secrets layout" && git push   # [Mac]
```

---

## Task 5: pihole + unbound (DNS must survive)

**Files:**
- Create: `k8s/pihole.yaml` (Deployment pihole, Service pihole, Deployment unbound, Service unbound, Middleware pihole-redirect)

**Step 1: Write `k8s/pihole.yaml`** `[Mac]` — note the host ports (53/5335), capabilities, and the FTLCONF upstream pointing at the `unbound` Service. Volume paths are placeholders; `config-paths.patch.yaml` points them at the real dirs (Step 3).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pihole
  labels: { app: pihole }
spec:
  replicas: 1
  selector: { matchLabels: { app: pihole } }
  template:
    metadata: { labels: { app: pihole } }
    spec:
      containers:
        - name: pihole
          image: pihole/pihole:latest
          imagePullPolicy: Always
          env:
            - { name: TZ, value: Europe/Paris }
            - { name: FTLCONF_dns_listeningMode, value: all }
            - { name: FTLCONF_dns_upstreams, value: "unbound#53" }   # k8s Service name, port 53
            - name: FTLCONF_webserver_api_password
              valueFrom:
                secretKeyRef: { name: pihole-secrets, key: FTLCONF_webserver_api_password }
          ports:
            - { name: dns-tcp, containerPort: 53, hostPort: 53, protocol: TCP }
            - { name: dns-udp, containerPort: 53, hostPort: 53, protocol: UDP }
            - { name: web, containerPort: 80, hostPort: 90 }        # keep direct :90 admin access
          securityContext:
            capabilities: { add: [NET_ADMIN, SYS_TIME, SYS_NICE] }
          readinessProbe:
            tcpSocket: { port: 80 }
          volumeMounts:
            - { name: etc-pihole, mountPath: /etc/pihole }
            - { name: etc-dnsmasq, mountPath: /etc/dnsmasq.d }
      volumes:
        - name: etc-pihole
          hostPath: { path: /srv/compositions/pihole/etc-pihole, type: DirectoryOrCreate }
        - name: etc-dnsmasq
          hostPath: { path: /srv/compositions/pihole/etc-dnsmasq.d, type: DirectoryOrCreate }
---
apiVersion: v1
kind: Service
metadata:
  name: pihole
spec:
  selector: { app: pihole }
  ports:
    - { name: web, port: 80, targetPort: web }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unbound
  labels: { app: unbound }
spec:
  replicas: 1
  selector: { matchLabels: { app: unbound } }
  template:
    metadata: { labels: { app: unbound } }
    spec:
      containers:
        - name: unbound
          image: mvance/unbound:latest
          imagePullPolicy: Always
          ports:
            - { name: dns-tcp, containerPort: 53, hostPort: 5335, protocol: TCP }  # keep host 5335 for pihole2
            - { name: dns-udp, containerPort: 53, hostPort: 5335, protocol: UDP }
          readinessProbe:
            tcpSocket: { port: 53 }
          volumeMounts:
            - name: a-records
              mountPath: /opt/unbound/etc/unbound/a-records.conf
              subPath: a-records.conf
              readOnly: true
      volumes:
        - name: a-records
          hostPath: { path: /srv/compositions/pihole/unbound-config, type: Directory }
---
apiVersion: v1
kind: Service
metadata:
  name: unbound          # cluster-internal: pihole's FTLCONF_dns_upstreams → unbound:53
spec:
  selector: { app: unbound }
  ports:
    - { name: dns, port: 53, targetPort: dns-udp, protocol: UDP }
    - { name: dns-tcp, port: 53, targetPort: dns-tcp, protocol: TCP }
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: pihole-redirect
spec:
  redirectRegex:
    regex: "^(https?://[^/]+)/?$"
    replacement: '${1}/admin/'
```

> **Pitfalls to check when it runs:** pihole listens on host port 53 — the k3s node's own resolver (systemd-resolved/dnsmasq) must not hold 0.0.0.0:53 (docker already bound it before, so likely fine; verify with Task 1's `ss` output). If kubelet/DNS conflict, this pod will fail to bind — see rollback note in Task 15.

**Step 2: `config-paths.patch.yaml` entry** (overrides the two pihole + one unbound placeholder paths):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pihole
spec:
  template:
    spec:
      volumes:
        - name: etc-pihole
          hostPath: { path: /home/alistair/repos/compositions/pihole/etc-pihole, type: Directory }
        - name: etc-dnsmasq
          hostPath: { path: /home/alistair/repos/compositions/pihole/etc-dnsmasq.d, type: Directory }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unbound
spec:
  template:
    spec:
      volumes:
        - name: a-records
          hostPath: { path: /home/alistair/repos/compositions/pihole/unbound-config, type: Directory }
```

**Step 3: secrets + apply** `[Mac → manjaro]`

```bash
cp pihole/.env k8s/secrets/pihole.env                  # [Mac] (fields: FTLCONF_webserver_api_password)
# edit: keep only the fields listed in pihole.env.example
# add to kustomization.yaml:
#   secretGenerator:
#     - name: pihole-secrets
#       envs: [secrets/pihole.env]
git add -A && git commit -m "chore(k8s): pihole + unbound manifests" && git push   # [Mac]
# [on manjaro]
git pull
kubectl apply --dry-run=client -k k8s/ && kubectl apply -k k8s/
kubectl -n compositions get pods -w        # pihole & unbound Running
kubectl -n compositions logs deploy/pihole --tail=20
```

**Step 4: verify DNS on the LAN** `[Mac]` (pihole still owned by docker until Task 15 — this step only proves the manifests are sound before cutover):

```bash
kubectl port-forward -n compositions svc/unbound 5353:53 &   # local probe
dig @127.0.0.1 -p 5353 ferguson.fr +short                    # should return 192.168.1.108
kubectl logs -n compositions deploy/pihole --tail=20 | grep -i "listening\|FTL"   # healthy boot
```
**Do NOT apply Step 4's cutover yet** — the docker pihole still owns host 53. Real cutover happens in Task 15. This task's apply proves the DNS pods run (they just can't bind 53 while docker pihole holds it — if binding fails here, defer the hostPort apply to Task 15 and note it).

**Commit:** `chore(k8s): pihole + unbound (dns)`

---

## Task 6: traefik dashboard + trala

**Files:**
- Create: `k8s/trala.yaml` (Deployment + Service trala)
- (dashboard IngressRoute lives in `k8s/ingress.yaml`, Task 13)

**Step 1: `k8s/trala.yaml`** `[Mac]`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trala
  labels: { app: trala }
spec:
  replicas: 1
  selector: { matchLabels: { app: trala } }
  template:
    metadata: { labels: { app: trala } }
    spec:
      containers:
        - name: trala
          image: ghcr.io/dannybouwers/trala:latest
          imagePullPolicy: Always
          ports:
            - { name: http, containerPort: 8080 }
          readinessProbe:
            httpGet: { path: /, port: 8080 }
          volumeMounts:
            - { name: config, mountPath: /config/configuration.yml, subPath: configuration.yml, readOnly: true }
            - { name: icons, mountPath: /icons, readOnly: true }
      volumes:
        - name: config
          hostPath: { path: /srv/compositions/traefik/trala-config, type: Directory }
        - name: icons
          hostPath: { path: /srv/compositions/traefik/trala-config/icons, type: Directory }
---
apiVersion: v1
kind: Service
metadata:
  name: trala
spec:
  selector: { app: trala }
  ports:
    - { name: http, port: 8080, targetPort: 8080 }
```

**Step 2:** add `- trala.yaml` to kustomization `resources`, add patch for the two hostPath dirs (`/home/alistair/repos/compositions/traefik/trala-config` …). Apply via the standard loop. Verify: `kubectl -n compositions get pods -l app=trala` Running; trala has no host-facing requirement.

**Commit:** `chore(k8s): trala`

> Notes: the old docker traefik container itself is **replaced** by the bundled k3s Traefik (Task 2/3) — no manifest for it. Its dashboard is served by an IngressRoute (`api@internal`, Task 13). The `./letsencrypt` dir stays untouched on disk (acme.json now lives in the k3s traefik's PVC); consider deleting after cutover.

---

## Task 7: authentik (server + worker + db + redis)

**Files:**
- Create: `k8s/authentik.yaml` — Deployments: `authentik-server`, `authentik-worker`, `authentik-db`, `authentik-redis` + Services `authentik-server`, `authentik-db`, `authentik-redis`

**Step 1: env/host renames (D3)** — the docker `.env`/compose envs reference plain `db`/`redis`; in k8s they must point at the app-scoped Services:

| env key (compose) | value (docker) | value (k8s) |
|---|---|---|
| `AUTHENTIK_REDIS__HOST` | `redis` | `authentik-redis` |
| `AUTHENTIK_POSTGRESQL__HOST` | `db` | `authentik-db` |
| `AUTHENTIK_POSTGRESQL__USER/NAME/PASSWORD`, `AUTHENTIK_SECRET_KEY` | from `.env` | from Secret `authentik-secrets` |
| db `POSTGRES_DB/USER/PASSWORD` | from `.env` | same Secret |

**Step 2: `k8s/authentik.yaml`** (key excerpts — full file is server/worker/db/redis Deployments + Services):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: authentik-server
  labels: { app: authentik-server }
spec:
  replicas: 1
  selector: { matchLabels: { app: authentik-server } }
  template:
    metadata: { labels: { app: authentik-server } }
    spec:
      containers:
        - name: server
          image: ghcr.io/goauthentik/server:2026.2
          imagePullPolicy: Always
          args: ["server"]                       # image entrypoint needs the role arg
          ports:
            - { name: http, containerPort: 9000 }
          readinessProbe:
            httpGet: { path: /-/health/live/, port: http }    # verify path; authentik has /-/health/live
          envFrom:
            - secretRef: { name: authentik-secrets }
          env:
            - { name: AUTHENTIK_REDIS__HOST, value: authentik-redis }
            - { name: AUTHENTIK_POSTGRESQL__HOST, value: authentik-db }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: authentik-worker
  labels: { app: authentik-worker }
spec:
  replicas: 1
  selector: { matchLabels: { app: authentik-worker } }
  template:
    metadata: { labels: { app: authentik-worker } }
    spec:
      containers:
        - name: worker
          image: ghcr.io/goauthentik/server:2026.2
          imagePullPolicy: Always
          args: ["worker"]
          envFrom:
            - secretRef: { name: authentik-secrets }
          env:
            - { name: AUTHENTIK_REDIS__HOST, value: authentik-redis }
            - { name: AUTHENTIK_POSTGRESQL__HOST, value: authentik-db }
---
# authentik-db: image docker.io/library/postgres:16, env POSTGRES_* from authentik-secrets,
#   hostPath pgdata, readinessProbe:
#     exec: ["sh","-c","pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"]
# authentik-redis: image docker.io/library/redis:7, hostPath redisdata, no probe needed
---
apiVersion: v1
kind: Service
metadata:
  name: authentik-server
spec:
  selector: { app: authentik-server }
  ports:
    - { name: http, port: 9000, targetPort: http }
---
apiVersion: v1
kind: Service
metadata:
  name: authentik-db        # ClusterIP only; nothing outside the namespace needs it
spec:
  selector: { app: authentik-db }
  ports:
    - { name: pg, port: 5432, targetPort: 5432 }
---
apiVersion: v1
kind: Service
metadata:
  name: authentik-redis
spec:
  selector: { app: authentik-redis }
  ports:
    - { name: redis, port: 6379, targetPort: 6379 }
```

> **Pitfall:** the ghcr.io/goauthentik/server image's entrypoint runs the `server` role by default in docker via the compose service *name* implicit behavior — in k8s there is no such inference; pass `args: ["server"]` / `["worker"]` explicitly (verify against the image's entrypoint docs `docker inspect` if boot fails).

**Step 3:** `cp authentik/.env k8s/secrets/authentik.env` (prune to the keys in the example; do not commit), add `authentik.yaml` + secretGenerator entry, patch pgdata/redisdata hostPaths (`/home/alistair/repos/compositions/authentik/pgdata` …), apply via standard loop.

**Step 4: verify** `[on manjaro]`

```bash
kubectl -n compositions get pods -l 'app in (authentik-server,authentik-worker,authentik-db,authentik-redis)'
kubectl -n compositions logs deploy/authentik-server --tail=30        # boots without db/redis errors
kubectl port-forward -n compositions svc/authentik-server 9000 &      # http://localhost:9000/if/flow/…  loads
```

**Commit:** `chore(k8s): authentik`

---

## Task 8: paperless + paperless-ai

**Files:**
- Create: `k8s/paperless.yaml` — Deployments `paperless-webserver`, `paperless-db` (postgres:16), `paperless-broker` (redis:7), `paperless-ai` + Services

**Step 1: env/host renames**

| env key | value (docker) | value (k8s) |
|---|---|---|
| `PAPERLESS_DBHOST` | `db` | `paperless-db` |
| `PAPERLESS_REDIS` | `redis://broker:6379` | `redis://paperless-broker:6379` |
| `PAPERLESS_URL` | from `.env` | `https://paperless.ferguson.fr` |
| `PAPERLESS_CSRF_TRUSTED_ORIGINS` | from `.env` | `https://paperless.ferguson.fr` (TLS via le) |
| `PAPERLESS_DBPASS`, `PAPERLESS_SECRET_KEY`, db `POSTGRES_*` | from `.env` | Secret `paperless-secrets` |

**Step 2: manifests** — same shapes as Task 7 (postgres/redis Deployments + Services; webserver with hostPath volumes `data`, `media`, `export`, `consume`; paperless-ai with hostPath `paperless-ai-data` on `/app/data`, port 3000, HTTP-only, needs env to reach the webserver — **verify**: docker paperless-ai had no env, so confirm it finds paperless via env default; if it can't, add `PAPERLESS_URL=https://paperless.ferguson.fr` (or the in-cluster URL) to its env and test).

**Step 3:** secrets (`cp paperless/.env k8s/secrets/paperless.env`), kustomize entries, hostPath patches:
- `/home/alistair/repos/compositions/paperless/{pgdata,redisdata,data,media,export,consume,paperless-ai-data}`
- **Media dir: read the value of `PAPERLESS_MEDIA_DIR` from paperless/.env (Appendix B) and use that absolute path for the `media` volume** — it may live outside the repo (shared with zerobyte).

**Step 4: verify** — webserver boots (logs show "paperless-ngx ready"), paperless-ai serves on 3000 via port-forward; check a document API call works (existing data intact: `curl http://localhost:8000/api/…` via port-forward, or the UI).

**Commit:** `chore(k8s): paperless + paperless-ai`

---

## Task 9: immich (server + machine-learning + db + redis)

**Files:**
- Create: `k8s/immich.yaml` — Deployments `immich-server`, `immich-machine-learning`, `immich-db` (tensorchord/pgvecto-rs:pg16-v0.2.0), `immich-redis` + Services

**Step 1: env/host renames**

| env key | value (docker) | value (k8s) |
|---|---|---|
| `DB_HOSTNAME` | `db` | `immich-db` |
| `REDIS_HOSTNAME` | `redis` | `immich-redis` |
| `IMMICH_MACHINE_LEARNING_URL` | `http://machine-learning:3003` | `http://immich-machine-learning:3003` |
| `DB_USERNAME/DB_PASSWORD/DB_DATABASE_NAME` | from `.env` | Secret `immich-secrets` |
| immich OIDC (OAUTH_CLIENT_*) | from `.env` | keep; URLs point at authentik ingress (auth.ferguson.fr) — unchanged |

**Step 2: manifests** — server ports 2283, volumes `upload` (hostPath = value of `IMMICH_IMAGES` from immich/.env; mount `/usr/src/app/upload`) + `/etc/localtime` ro; ml volume `model-cache` (hostPath = `MODEL_CACHE`) on `/cache`; db hostPath `pgdata`; redis hostPath `redisdata`. Readiness probes: server `httpGet /` on 2283 (or `/api/server-info/ping`), db `pg_isready`.

**Step 3:** secrets + kustomize + config-paths patches (Appendix B paths); apply via standard loop.

**Step 4: verify**

```bash
kubectl -n compositions get pods -l 'app in (immich-server,immich-machine-learning,immich-db,immich-redis)'
kubectl -n compositions logs deploy/immich-server --tail=30        # no db/redis connect errors
kubectl port-forward -n compositions svc/immich-server 2283 &
curl -s localhost:2283/api/server-info/ping
```
Expected: `{"res":"pong"}`. Existing photo library intact (hostPath on the real `IMMICH_IMAGES`).

**Commit:** `chore(k8s): immich`

---

## Task 10: vikunja (+ db)

**Files:**
- Create: `k8s/vikunja.yaml` — Deployments `vikunja`, `vikunja-db` + Services

**Step 1: env renames** — `VIKUNJA_DATABASE_HOST: vikunja-db`; `VIKUNJA_SERVICE_PUBLICURL`: `https://planning.ferguson.fr`; `VIKUNJA_DATABASE_PASSWORD` + db `POSTGRES_*` from Secret `vikunja-secrets`; `VIKUNJA_FILES` → hostPath.

**Step 2: replicate compose's `depends_on: db (service_healthy)`** with an initContainer:

```yaml
initContainers:
  - name: wait-for-db
    image: docker.io/library/postgres:16
    command: ["sh", "-c", "until pg_isready -h vikunja-db -U vikunja; do echo waiting; sleep 2; done"]
```

**Step 3:** secrets, kustomize, patches; apply; verify `kubectl logs -n compositions deploy/vikunja --tail=20` shows DB connected + port-forward 3456 loads the UI with existing tasks.

**Commit:** `chore(k8s): vikunja`

---

## Task 11: zerobyte (multi-mount backup)

**Files:**
- Create: `k8s/zerobyte.yaml` (Deployment + Service)

**Step 1: volumes (all hostPath)** — from compose:

| volume (compose) | mount | hostPath (manjaro) |
|---|---|---|
| `./var-lib-zerobyte` | `/var/lib/zerobyte` | `/home/alistair/repos/compositions/zerobyte/var-lib-zerobyte` |
| `~/.config/rclone` | `/root/.config/rclone` (ro) | `/home/alistair/.config/rclone` |
| `./cache-restic` | `/root/.cache/restic` | `/home/alistair/repos/compositions/zerobyte/cache-restic` |
| `${PAPERLESS_MEDIA_DIR}` | `/data/paperless/media` (ro) | value from Appendix B |
| `${IMMICH_IMAGES}` | `/data/immich/images` (ro) | value from Appendix B |

**Step 2:** env `TZ` + `TRUSTED_ORIGINS` (keep compose value; ensure `https://zerobyte.ferguson.fr` is present for TLS). Port 4096, readiness `httpGet /` on 4096. `readOnly: true` on the two shared mounts (paperless/immich dirs) so zerobyte can't mutate sources.

**Step 3:** kustomize entry + patches; apply; verify: `kubectl logs -n compositions deploy/zerobyte --tail=20` (rclone config parses), port-forward 4096 → UI.

**Commit:** `chore(k8s): zerobyte`

---

## Task 12: cloudflared (tunnel)

**Files:**
- Create: `k8s/cloudflared.yaml` (Deployment + Secret usage)

**Step 1:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  labels: { app: cloudflared }
spec:
  replicas: 1
  selector: { matchLabels: { app: cloudflared } }
  template:
    metadata: { labels: { app: cloudflared } }
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          imagePullPolicy: Always
          args: ["tunnel", "run", "--token", "$(TUNNEL_TOKEN)"]   # remote-managed tunnel (same as compose image default)
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef: { name: cloudflared-tunnel, key: TUNNEL_TOKEN }
```

**Step 2:** secrets (`cp cloudflared/.env k8s/secrets/cloudflared.env`), kustomize, apply without any Service.

**Step 3: verify origin reachability** — the tunnel's public-hostname origins are defined in the Cloudflare Zero Trust dashboard, **not** in this repo. They currently point at the docker `traefik` origin (docker network DNS name `traefik` was resolvable on the `tunnel` docker network). Options, in order of preference:
1. If origin = `https://192.168.1.108:443` (node IP): nothing to change — the k8s Traefik owns 80/443 on the node.
2. If origin = a docker network name (e.g. `http://traefik:80`): point it at `https://192.168.1.108:443` in the dashboard (or optionally the Traefik Service ClusterIP — but the node IP is simpler and survives Traefik restarts).

Verify after cutover: `curl -s -o /dev/null -w '%{http_code}\n' https://<public-hostname>` from the WAN (or `dig` via a public resolver + curl) → expect 404 from traefik (no routes yet) then real app responses after Task 13/15.

```bash
kubectl -n compositions logs deploy/cloudflared --tail=20   # "Registered tunnel connection"
```

**Commit:** `chore(k8s): cloudflared`

---

## Task 13: ingress — all routes + TLS

**Files:**
- Create: `k8s/ingress.yaml` (Middleware already in `pihole.yaml`; IngressRoutes here) — **keep commented out of `kustomization.yaml` until LE is proven** (nzb did exactly this).

**Step 1: `k8s/ingress.yaml`** — one IngressRoute per host (entryPoint `websecure`, `tls.certResolver: le` commented until verified):

| IngressRoute | match Host | service:port |
|---|---|---|
| `trala` | `ferguson.fr` | trala:8080 |
| `pihole` | `pihole.ferguson.fr` | pihole:80 (middlewares: pihole-redirect) |
| `traefik-dashboard` | `traefik.ferguson.fr` | `api@internal` (TraefikService) |
| `auth` | `auth.ferguson.fr` | authentik-server:9000 |
| `paperless` | `paperless.ferguson.fr` | paperless-webserver:8000 |
| `paperless-ai` | `paperless-ai.ferguson.fr` | paperless-ai:3000 |
| `pics` | `pics.ferguson.fr` | immich-server:2283 |
| `planning` | `planning.ferguson.fr` | vikunja:3456 |
| `zerobyte` | `zerobyte.ferguson.fr` | zerobyte:4096 |

Shape (all identical except the table values):

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: auth
spec:
  entryPoints: [websecure]
  routes:
    - kind: Rule
      match: Host(`auth.ferguson.fr`)
      services:
        - name: authentik-server
          port: 9000
  # tls:
  #   certResolver: le
---
# … one block per row above (the dashboard IngressRoute uses:
#   services: [{ name: api@internal, kind: TraefikService }])
```

**Step 2: prove LE + TLS before enabling everything** `[on manjaro]`

```bash
# temporarily uncomment tls: {certResolver: le} on ONE route (auth) and add ingress.yaml to resources
kubectl apply -k k8s/
kubectl -n compositions get ingressroute
kubectl -n kube-system logs deploy/traefik --tail=50 | grep -iE "acme|error"
curl -s -o /dev/null -w '%{http_code}\n' -k https://192.168.1.108/ -H "Host: auth.ferguson.fr"   # 401/302 = traefik proxy works
curl -s https://auth.ferguson.fr --resolve auth.ferguson.fr:443:192.168.1.108 -o /dev/null -w '%{http_code}\n'  # 200/302 once cert issued
```
Expected: LE DNS challenge succeeds (TXT via Cloudflare), cert issued, https://auth.ferguson.fr answers. If issuance fails, check `CF_DNS_API_TOKEN` env + DNS TXT propagation in traefik logs.

**Step 3:** uncomment `tls:` on all routes, apply, verify each host returns the app (or a known 4xx, not a TLS/route error). Also confirm `http://…` redirects to `https://` (helm chart `ports.web.redirectTo`, Task 3).

**Commit:** `chore(k8s): ingress + TLS`

---

## Task 14: full dry-run review before cutover

**Files:** none (review only).

**Step 1:** `kubectl apply --dry-run=client -k k8s/` then `kubectl diff -k k8s/` `[on manjaro]`; walk the diff:
- every Deployment matches its docker counterpart (image, env, ports, volumes);
- every hostPath in `config-paths.patch.yaml` resolves to an existing dir on manjaro (`ls -d` each — expect ~22 paths);
- no service-name collisions remaining (`kubectl get svc -n compositions` → unique names; Appendix A table holds);
- secrets generated (`kubectl get secret -n compositions | grep -E 'authentik|paperless|immich|vikunja|pihole|cloudflared'`).

**Step 2:** health-check the running addons once more (`kubectl get pods -A | grep -v Running`) — don't cut over on a sick cluster.

**Commit:** nothing (or a notes commit if findings).

---

## Task 15: cutover (per-app, no big-bang)

> Ordering principle: an app's hostPath data must never be written by docker **and** k8s simultaneously. For each app: stop the docker stack → apply (already applied in earlier tasks) → verify → next. DNS is untouched the whole time (records already point at the node).

**Step 1: Traefik.** If docker traefik is running: `docker -c manjaro stop traefik` → verify `ss -tlnp | grep :80` free → (k3s traefik should be running from Task 2/3; if it started before this, skip).

**Step 2: per-app flip** (apps currently running in docker: authentik, paperless, zerobyte, pihole+unbound; immich/vikunja/trala/cloudflared were already stopped — just stay applied):

```bash
# [on manjaro] example: authentik
cd /home/alistair/repos/compositions/authentik
docker compose down            # stops server/worker/db/redis, removes its containers
kubectl -n compositions get pods -l 'app in (authentik-server,authentik-worker,authentik-db,authentik-redis)'
kubectl -n compositions rollout status deploy/authentik-server -n compositions
# verify: curl via node IP with Host header, or kubectl port-forward; then check a login page loads
```
Repeat for `paperless` (webserver/db/paperless-ai), `zerobyte`, and finally **pihole+unbound** (DNS): stop docker pihole → k8s pihole binds host 53 (hostPort) → `dig @192.168.1.108 ferguson.fr` works from the LAN → check `192.168.1.11` pihole2 still resolves (its upstream 5335 must be served by k8s unbound).

```bash
# pihole flip
docker -c manjaro stop pihole-pihole-1 pihole-unbound-1
kubectl -n compositions rollout status deploy/pihole deploy/unbound
dig @192.168.1.108 ferguson.fr +short                  # expect 192.168.1.108 (unbound A record)
dig @192.168.1.108 paperless.ferguson.fr +short        # resolves
# and from a LAN client: DNS + admin UI both work
```

**Step 3: tunnel** — confirm cloudflared k8s pod registered (Task 12 Step 3). If the dashboard origins needed changing, do it now and re-check public hostnames.

**Step 4: full smoke test** — table in Appendix F, or just: every URL in the README's DNS list returns its app over `https://…`. Include: https://ferguson.fr (trala), auth (login flow), paperless (documents visible with old data), pics (albums), planning (tasks), zerobyte (backups list), pihole admin.

**Step 5: rollback plan if an app misbehaves:** `kubectl -n compositions scale deploy <app> --replicas=0` then `cd <app> && docker compose up -d` [on manjaro] — old containers take over using the same data dirs (app-level rollback; note postgres on the same pgdata: scale k8s db down *before* starting docker db).

**Commit:** none required (cluster ops); record cutover checklist results in the README (Task 16).

---

## Task 16: decommission docker + docs + clean up

**Files:**
- Modify: `README.md` (update DNS table: all services now https://…, served by k3s; remove the "everything http because CBA" note)
- Delete: nothing in git that's still referenced; keep compose files for rollback until stable (then optionally move to `legacy-docker/` in a follow-up)

**Step 1: stop & remove remainder** `[on manjaro]` — for each migrated app dir: `docker compose down`; then remove the docker networks used by the stack (`docker network rm traefik tunnel` — the `traefik`/`tunnel` overlay networks; confirm no other compose projects use them: nzb does NOT use them, verified in its compose: it uses default). Leave the nzb docker stack running (separate project).

**Step 2:** verify nothing depends on docker: `docker -c manjaro ps` should list only nzb containers (jellyfin, gluetun/sonarr/radarr/bazarr, sabnzbd).

**Step 3:** update README (URLs table; k8s ops pointer: apply/rollout commands; note secrets location `k8s/secrets/` git-ignored). Commit: `docs: cutover to k3s complete`.

**Step 4 (follow-up, not blocking):** weekly image refresh — `kubectl -n compositions rollout restart deploy/<app>` after bumping `images:` tags in `kustomization.yaml`. Consider Renovate for tag bumps (nzb's roadmap).

---

## Appendix A — Service/Deployment name map (docker → k8s)

| docker service (per folder) | k8s Deployment | k8s Service | notes |
|---|---|---|---|
| pihole/pihole | `pihole` | `pihole` (80) | hostPort 53, 90 |
| pihole/unbound | `unbound` | `unbound` (53) | hostPort 5335 |
| traefik/trala | `trala` | `trala` (8080) | docker `traefik` itself → bundled k3s Traefik (no manifest) |
| authentik/server | `authentik-server` | `authentik-server` (9000) | drops host 9000/9443 |
| authentik/worker | `authentik-worker` | — (no svc) | |
| authentik/db | `authentik-db` | `authentik-db` (5432) | was `db` |
| authentik/redis | `authentik-redis` | `authentik-redis` (6379) | was `redis` |
| paperless/webserver | `paperless-webserver` | `paperless-webserver` (8000) | |
| paperless/db | `paperless-db` | `paperless-db` (5432) | was `db` |
| paperless/broker | `paperless-broker` | `paperless-broker` (6379) | was `broker` |
| paperless/paperless-ai | `paperless-ai` | `paperless-ai` (3000) | drops host 3000 |
| immich/server | `immich-server` | `immich-server` (2283) | drops host 2283 |
| immich/machine-learning | `immich-machine-learning` | `immich-machine-learning` (3003) | |
| immich/db | `immich-db` | `immich-db` (5432) | was `db`, pgvecto-rs image |
| immich/redis | `immich-redis` | `immich-redis` (6379) | was `redis` |
| vikunja/vikunja | `vikunja` | `vikunja` (3456) | |
| vikunja/db | `vikunja-db` | `vikunja-db` (5432) | was `db` |
| zerobyte/zerobyte | `zerobyte` | `zerobyte` (4096) | |
| cloudflared/cloudflared | `cloudflared` | — (no svc) | |

## Appendix B — hostPath volume map (fill Task 1 → used in config-paths.patch.yaml)

| app | volume | hostPath (manjaro) |
|---|---|---|
| authentik | pgdata / redisdata | `/home/alistair/repos/compositions/authentik/{pgdata,redisdata}` |
| paperless | pgdata / redisdata / data / export / consume / paperless-ai-data | `/home/alistair/repos/compositions/paperless/{…}` |
| paperless | media | `${PAPERLESS_MEDIA_DIR}` (**resolve from paperless/.env**) |
| immich | pgdata / redisdata | `/home/alistair/repos/compositions/immich/{pgdata,redisdata}` |
| immich | upload / model-cache | `${IMMICH_IMAGES}` / `${MODEL_CACHE}` (**resolve from immich/.env**) |
| vikunja | pgdata / files | `/home/alistair/repos/compositions/vikunja/{pgdata}`, `${VIKUNJA_FILES}` |
| pihole | etc-pihole / etc-dnsmasq.d | `/home/alistair/repos/compositions/pihole/{etc-pihole,etc-dnsmasq.d}` |
| unbound | unbound-config | `/home/alistair/repos/compositions/pihole/unbound-config` |
| trala | trala-config (+icons) | `/home/alistair/repos/compositions/traefik/trala-config[/icons]` |
| zerobyte | var-lib / cache-restic | `/home/alistair/repos/compositions/zerobyte/{var-lib-zerobyte,cache-restic}` |
| zerobyte | rclone config | `/home/alistair/.config/rclone` (ro) |
| zerobyte | /data/paperless/media, /data/immich/images | shared with paperless/immich entries (ro) |

## Appendix C — secrets inventory

| Secret (ns) | from | keys |
|---|---|---|
| `cf-dns-api-token` (kube-system) | `k8s/secrets/cf-dns-api-token.env` | CF_DNS_API_TOKEN |
| `authentik-secrets` | authentik/.env | AUTHENTIK_SECRET_KEY, AUTHENTIK_POSTGRESQL__USER/NAME/PASSWORD, (POSTGRES_DB/USER/PASSWORD for db) |
| `paperless-secrets` | paperless/.env | POSTGRES_PASSWORD, PAPERLESS_DBPASS, PAPERLESS_SECRET_KEY |
| `immich-secrets` | immich/.env | DB_PASSWORD, DB_USERNAME, DB_DATABASE_NAME, OAUTH_CLIENT_ID/SECRET |
| `vikunja-secrets` | vikunja/.env | POSTGRES_PASSWORD, VIKUNJA_DATABASE_PASSWORD, VIKUNJA_SERVICE_JWTSECRET |
| `pihole-secrets` | pihole/.env | FTLCONF_webserver_api_password |
| `cloudflared-tunnel` | cloudflared/.env | TUNNEL_TOKEN |

## Appendix D — cluster/ops cheatsheet (post-migration)

```bash
kubectl apply -k k8s/                       # apply everything (run from repo root on manjaro)
kubectl diff -k k8s/                        # review
kubectl -n compositions get pods            # status
kubectl -n compositions rollout restart deploy/<app>   # image refresh after tag bump
kubectl port-forward -n compositions svc/<svc> <local>:<port>  # reach a service without ingress
kubectl -n kube-system logs deploy/traefik --tail=50 | grep -i error   # ingress debugging
```

## Appendix E — risk register

| Risk | Mitigation |
|---|---|
| Bundled Traefik addon won't install (it never has on this box) | Task 2 diagnosis + controller restart; fallback `--disable traefik` + official chart; hard cap 30 min |
| hostPort 53 conflicts with node resolver | Verified free today (docker owns it); flip docker→k8s pihole in one step; if kubelet complains, re-check `ss` before applying |
| Double-writer data corruption during cutover | Per-app sequential cutover (Task 15); rollback = scale k8s to 0 + docker compose up |
| pihole2 (192.168.1.11) breaks on 5335 | Keep unbound hostPort 5335; verify in Task 1/15 |
| Local-path provisioner instability (1853 restarts) | We only use hostPath + traefik PVC (local-path); watch Task 1 health; re-apply iptables fix if addons degrade |
| Tunnel origin stops resolving after cloudflared moves | Check dashboard origins (Task 12); node IP origin is robust |
| LE certs fail (rate limits / token) | Keep acme.json persistence (256Mi PVC); token from proven docker config; staging CA as debug lever (put `caserver=https://acme-staging…` temporarily) |
| Cluster silently degrades after `pacman -Syu` | iptables-legacy symlink check (nzb doc) — add to README ops notes |

## Appendix F — smoke-test checklist (Task 15 Step 4)

- [ ] `https://ferguson.fr` → trala dashboard
- [ ] `https://auth.ferguson.fr` → authentik login (and OIDC works for immich)
- [ ] `https://paperless.ferguson.fr` → documents still present (old DB intact)
- [ ] `https://paperless-ai.ferguson.fr` → AI chat finds the same docs
- [ ] `https://pics.ferguson.fr` → photo library intact (old upload dir)
- [ ] `https://planning.ferguson.fr` → tasks present
- [ ] `https://zerobyte.ferguson.fr` → backups list, rclone remotes visible
- [ ] `https://pihole.ferguson.fr` → admin UI (redirect middleware works)
- [ ] `https://traefik.ferguson.fr` → dashboard (api@internal works)
- [ ] `dig @192.168.1.108 ferguson.fr` → 192.168.1.108; pihole2 on 1.11 still resolves
- [ ] public URL via tunnel (after Task 12/15) → app answer
- [ ] `dig @<node> TXT _acme-challenge…` (optional, during issuance only)