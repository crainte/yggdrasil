# Cluster migration

This document tracks the migration from the legacy `local` cluster to `yggdrasil`.

## Operating rules

- Treat `local` as the authoritative source until each replacement is validated.
- Build reusable desired state as Onigiri packages; keep cluster-specific values here.
- Lock package artifacts and tagged container images in `oni.lock` before deployment.
- Use server-side dry-run, review, then server-side apply with field manager `onigiri`.
- Do not use `--force-conflicts` globally. Inspect and resolve each ownership conflict.
- Validate replacement traffic before retiring the corresponding legacy workload.
- Do not inspect or copy Secret values unless a migration explicitly requires it.

## Status

| Workload | Disposition | Status | Notes |
| --- | --- | --- | --- |
| Gateway front doors | Install | Complete | Internal Gateway `172.16.16.10` and public NAT-target Gateway `172.16.16.11` serve wildcard `*.itai.moe` TLS. The shared Cilium Ingress at `172.16.16.12` is being retired; both gateways are the only front doors. |
| Metrics Server | Install | Complete | Metrics Server 0.9.0 runs two anti-affined replicas; Metrics API and `kubectl top` are operational. |
| Grafana Alloy | Migrate | Complete | Alloy chart 1.13.1 / app v1.20.1; metrics-only; remote write to `10.10.10.189:9090`; no logs or SigNoz. |
| Chrony/NTP | Replace | Cutover pending | Replacement is healthy at `172.16.16.123`; change clients and observe traffic before retiring `172.16.32.109`. |
| ntfy | Migrate | Internal route live | `ntfy.itai.moe` routes through the internal Gateway; SQLite data was copied. Client/Cloudflare cutover and legacy retirement remain pending. |
| Grafana | Migrate | Desired state authored | Migrate Grafana `10.4.2` unchanged to internal `grafana.itai.moe`; copy its SQLite database and plugins consistently, and defer notification integrations and application upgrades. |
| Authentik | Migrate | Pending | Database is authoritative for internally managed providers, applications, and outposts. Use a supported upgrade path. |
| Docker Registry | Migrate | Pending | Preserve hostname, credentials, and required image blobs. |
| Immich | Reprovision | Deployed, setup pending | Fresh Immich `v3.3.3` serves internal `immich.itai.moe`. Create the admin account, then consider `IMMICH_ALLOW_SETUP=false`. Legacy data is intentionally abandoned; legacy workloads remain until retired. |
| LANraragi | Migrate | Complete | `v.0.9.60` and all persistent data are validated at internal `manga.itai.moe`; legacy remains stopped with its volumes retained for rollback. |
| Smokeping | Migrate and upgrade | Desired state authored | Upgrade `v0.8.1` to `v0.12.0`; preserve the six ICMP targets and expose metrics only inside the cluster. |
| Unpoller | Migrate and upgrade | Desired state authored | Upgrade `v2.11.2` to `v5.7.0`; source `up.conf` from Bitwarden through External Secrets and expose metrics only inside the cluster. |
| Vertical Pod Autoscaler | Replace | Desired state authored | Install VPA `1.8.1` CRDs and recommender only; no updater, admission controller, or VPA objects. |
| Another Autoscaler | Retire | Retirement pending | No annotated consumers were found; do not reproduce its cluster-wide mutation privileges on Yggdrasil. |
| Kener | Replace | Complete | Lighthouse is validated at `lighthouse.itai.moe` and routes through the internal Gateway; application bugs will be handled independently. Legacy Kener decommissioning remains a cleanup action. |
| SigNoz | Retire | Pending | Do not migrate. Confirm no remaining producers or consumers before removal. |
| OneDev | Retire | Pending | Test installation only; do not migrate. |
| MariaDB | Retire candidate | Pending | Home Assistant uses SQLite. Confirm no other consumers before removal. |
| Mosquitto | Retire candidate | Pending | No known clients; verify before removal. |
| Legacy DNS | Retire candidate | Pending | DHCP is believed to have cut over. Validate active client configuration and traffic first. |
| Gluster storage | Retire candidate | Pending | Use currently mounted claims as authority, not historical claims. The 250 GiB `default/home` PVC appears unused. |

## Completed work

### Gateway API front doors

- Installed official Gateway API v1.6.3 standard CRDs through package `gateway-api@1.6.3`.
- Upgraded Cilium to 1.20.3 and enabled its Gateway API controller.
- Package `cilium-gateways@1.0.0` owns Gateways `gateway/internal` (`172.16.16.10`) and `gateway/public` (`172.16.16.11`), both HTTPS-only with the DNS-01 wildcard certificate `gateway/wildcard`.
- `ntfy.itai.moe` and `lighthouse.itai.moe` both attach to the internal Gateway. No route currently attaches to the public Gateway.
- The router must NAT public TCP/443 to `172.16.16.11`; DNS for internal names must resolve to `172.16.16.10`.
- Cilium 1.20.2 and 1.20.3 produced empty Envoy EDS clusters when an HTTPRoute used a numeric backend port against a named Service port. Packages therefore own Gateway compatibility Services with unnamed ports: `ntfy-gateway-backend`, `lighthouse-gateway-backend`, and `immich-gateway-backend`.
- The shared Cilium Ingress at `172.16.16.12` is no longer needed: both Gateways serve the wildcard certificate. Package `ntfy@1.0.0` no longer authors an Ingress or per-host Certificate. The Lighthouse package still authors them and must drop them in a new revision before the `172.16.16.12` listener and `lighthouse-tls` Certificate disappear.
- Cilium Helm values are still operational state, not declarative Yggdrasil desired state; capture them in a Talos/Cilium bootstrap source before rebuilding the cluster.

### Immich reprovision state

- Initially deployed as `immich@3.2.4`, then upgraded to `immich@3.3.3` (no breaking changes in v3.3.0–v3.3.3; same PostgreSQL image). The single-replica server returns 503 briefly during rollouts.
- Package `immich@3.3.3` creates fresh retained storage: 100 GiB library, 20 GiB PostgreSQL, and 10 GiB ML cache.
- PostgreSQL uses `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`; Valkey is ephemeral.
- Database credentials come from Bitwarden keys `IMMICH_DB_USERNAME`, `IMMICH_DB_PASSWORD`, and `IMMICH_DB_DATABASE`.
- The route is internal-only at `immich.itai.moe`.
- Deployed after the Bitwarden keys were added: ExternalSecret Ready, all four workloads Ready, three Longhorn PVCs Bound.
- Verified `/api/server/ping` and the web UI return 200 through the internal Gateway, and the public Gateway returns 404 for the host.
- Database has `vchord 0.4.3`, `vector 0.8.1`, and data checksums on; the server reaches ML (`/ping` returns `pong`).
- Remaining: initial admin setup, mobile/client access over internal DNS, then retire legacy `local/immich` (PV reclaim policy is `Delete`).

### Metrics Server

- Added Onigiri format-2 package `metrics-server@0.9.0` from the upstream Kubernetes 1.21+ HA manifest.
- Runs two required-anti-affined replicas with a PDB requiring one available replica.
- Uses upstream RBAC, APIService registration, probes, resource requests, and hardened container security.
- The image tag is digest-pinned by `oni.lock`.
- Talos kubelet certificates do not contain node IP SANs, so the deployment uses `--kubelet-insecure-tls`; service-account authentication and kubelet authorization remain enforced.
- `v1beta1.metrics.k8s.io` is Available and both `kubectl top nodes` and `kubectl top pods -A` return metrics.

### Grafana Alloy

- Added Onigiri format-2 package `alloy@1.13.1` based on the official Grafana chart.
- Deployed a DaemonSet across all three Yggdrasil nodes.
- Collects Alloy, kubelet, cAdvisor, and explicitly annotated Pod metrics.
- Adds `cluster="yggdrasil"` to remote-written series.
- Uses least-privilege `nodes/metrics`; does not grant dangerous `nodes/proxy` access.
- Main and config-reloader image tags are digest-pinned by `oni.lock`.

### Onigiri ownership transition

The first full server-side apply exposed immutable selectors on pre-existing Deployments. These were deliberately deleted and recreated with corrected stable selectors:

- `external-secrets/bitwarden-sdk-server`
- `cert-manager/cert-manager`
- `cert-manager/cert-manager-cainjector`
- `cert-manager/cert-manager-webhook`
- `external-secrets/external-secrets`
- `external-secrets/external-secrets-cert-controller`
- `external-secrets/external-secrets-webhook`

Webhook Deployments were recreated first because a partial apply had already changed their Service selectors and temporarily removed endpoints. All replacements and webhook endpoints were verified healthy afterward.

Bifrost package `0.1.2` was corrected to declare image tag `0.1.2`; `oni.lock` pins its digest.

## Chrony/NTP plan

### Legacy state

- Namespace: `ntp`
- Deployment: `chronyd`, 2 replicas
- Image: `simonrupf/chronyd:0.6.1` (chrony 4.6.1)
- Service: `chrony`, UDP/123, LoadBalancer IP `172.16.32.109`
- No workload ConfigMap; the image generates its configuration.

### Target contract

- Dedicated `ntp` namespace.
- UDP LoadBalancer Service on `172.16.16.123`.
- Kubernetes-native package and explicit server policy/configuration.
- No Gateway API dependency for UDP NTP traffic.
- Keep legacy NTP running until the replacement answers requests and clients have moved.

### Chrony checklist

- [x] Select `simonrupf/chronyd:0.9.1` (chrony 4.8) with an explicit mounted `chrony.conf`.
- [x] Restrict NTP clients to RFC1918 networks in both chrony and a NetworkPolicy.
- [x] Run one Chrony pod per eligible node as a DaemonSet (rolling `maxUnavailable: 1`) with `externalTrafficPolicy: Local` for source-IP preservation and node-level availability. No PodDisruptionBudget: drains never evict DaemonSet pods, so it would not protect anything.
- [x] Note: Chrony reads `chrony.conf` only at start, and the generic DaemonSet has no stdlib rollout annotation; after changing `chrony.yaml` config, restart it explicitly (`kubectl -n ntp rollout restart daemonset/chrony`).
- [x] Add process liveness and synchronized-source readiness checks through `chronyc`.
- [x] Create and validate the Onigiri package using typed Service and NetworkPolicy collections.
- [x] Lock the tagged image digest.
- [x] Server-side dry-run against Yggdrasil.
- [x] Deploy and verify UDP/123 at `172.16.16.123` from a LAN client.
- [ ] Change DHCP/NTP consumers to the new address.
- [ ] Observe traffic and synchronization before retiring legacy `172.16.32.109`.

## ntfy plan

### Legacy state

- ntfy `2.15.0`, one StatefulSet replica.
- Public hostname `ntfy.hybridoptix.net` through legacy nginx Ingress `172.16.32.106` and Cloudflare.
- One 1 GiB Longhorn PVC containing `user.db` (users, ACLs, tokens) and `cache.db` (message cache), totaling about 276 KiB.
- Authentication defaults to deny-all; both databases must be preserved.

### Target state

- Package `ntfy@2.15.0`, one hardened StatefulSet replica with retained 1 GiB storage.
- Shared Cilium Ingress is enabled at `172.16.16.10` on ports 80/443.
- Public hostname is `ntfy.itai.moe` through the shared Cilium Ingress.
- Package-owned typed Certificate and Ingress resources use the reusable `letsencrypt` ClusterIssuer.
- Shared package `letsencrypt-cloudflare` owns the Cloudflare-token ExternalSecret and DNS-01 ClusterIssuer; its token must exist in Bitwarden as `CLOUDFLARE_API_TOKEN` and be scoped to DNS edit plus zone read for the intended zones.

### Cutover checklist

- [x] Inspect legacy workload and persistent state without exposing user or token values.
- [x] Enable and verify shared Cilium Ingress at `172.16.16.10`.
- [x] Build and server-side dry-run the ntfy workload package.
- [x] Select `ntfy.itai.moe` as the replacement hostname.
- [x] Author a shared Let's Encrypt DNS-01 ClusterIssuer package.
- [x] Add typed Certificate and Ingress collections for `ntfy.itai.moe`.
- [x] Add the zone-scoped `CLOUDFLARE_API_TOKEN` value to Bitwarden Secrets Manager.
- [x] Deploy ntfy with an empty retained PVC but do not route production traffic.
- [x] Stop legacy ntfy, copy `user.db` and `cache.db`, and start the replacement; SHA-256 hashes match for both databases.
- [ ] Validate users, ACLs/tokens, cached messages, publish, subscribe, and `/v1/health`; direct origin health at `172.16.16.10` passes.
- [ ] Change DNS/Cloudflare routing to the shared ingress and observe traffic. Public requests currently return Cloudflare HTTP 525 because the private ingress has no confirmed Cloudflare-to-origin path.
- [ ] Retire the legacy StatefulSet only after rollback is no longer needed; it remains healthy for rollback.

## Grafana plan

### Legacy state

- Namespace and workload: `monitoring/grafana`, one healthy Deployment replica running Grafana `10.4.2` with no restarts.
- The active image digest is `sha256:7d5faae481a4c6f436c99e98af11534f7fd5e8d3e35213552dd1dd02bc393d2e`.
- A 1 GiB RWO Longhorn claim is mounted at `/var/lib/grafana`; about 6.3 MiB is used. The authoritative SQLite database is `/var/lib/grafana/grafana.db`.
- Persisted plugins are `grafana-piechart-panel`, `marcusolsson-treemap-panel`, `mxswat-separator-panel`, and `nikosc-percenttrend-panel`.
- The legacy hostname is `grafana.hybridoptix.net` through nginx Ingress `172.16.32.106`.
- The pod also has a `grafana-ntfy` sidecar with a plaintext credential, while Grafana SMTP points to the old-cluster `email-to-signal` service. Neither integration is part of the initial migration.

### Target contract

- Migrate on Grafana `10.4.2` using the exact locked image digest; perform a separately planned upgrade after migration and plugin/dashboard validation.
- Run one StatefulSet replica with a retained 1 GiB RWO Longhorn claim and internal-only HTTPRoute for `grafana.itai.moe`.
- Preserve `/var/lib/grafana` as a consistent unit so the SQLite database, alerting state, and installed plugins move together.
- Do not copy the embedded ntfy credential or configure SMTP during the initial migration. Reintroduce notifications later with supported endpoints and managed secrets.
- Keep the legacy instance stopped but recoverable during final synchronization and validation. Never run both instances against independently writable copies during cutover.

### Cutover checklist

- [x] Inventory the active workload, image digest, route, persistent data, plugins, resources, and notification integrations.
- [x] Select Grafana `10.4.2`, internal hostname `grafana.itai.moe`, and defer ntfy/SMTP integrations.
- [x] Author the Onigiri package and Yggdrasil configuration without deploying them.
- [x] Lock the package and image digest, run `oni check`, and review the rendered Grafana resources.
- [ ] Deploy the replacement with an empty retained claim while production remains on the legacy instance.
- [ ] Stop legacy Grafana before copying SQLite; record file counts, byte totals, and checksums for `/var/lib/grafana`.
- [ ] Copy the complete persisted directory to the replacement claim, preserve ownership for UID/GID 472, and start the replacement.
- [ ] Validate login, users and organizations, data sources, dashboards, alert rules, all four persisted plugins, and restart persistence.
- [ ] Add internal DNS for `grafana.itai.moe`, validate the internal Gateway route, and observe client traffic.
- [ ] Retain the stopped legacy workload and Longhorn volume through the rollback window.
- [ ] Separately plan a supported Grafana upgrade and replacement or removal of incompatible legacy plugins.

## LANraragi plan

### Legacy state

- Namespace and workload: `manga/lanraragi`, one healthy Deployment replica with no container restarts.
- Image: `difegue/lanraragi:v.0.9.60`, currently running digest `sha256:985566a301ab40834dca01c23d240bc88bb850b7e1f2cce30b535744b993210d`.
- Service: ClusterIP `manga/lanraragi`, TCP port 3000.
- Runtime usage observed during discovery was approximately 42 millicores and 257 MiB memory; the Deployment declares no requests or limits.
- Active persistent data is split across three RWO Longhorn claims:
  - `database`: 4 GiB provisioned, about 44 MiB used; mounted at `/home/koyomi/lanraragi/database` and containing Redis persistence files.
  - `content`: 20 GiB provisioned, about 11.6 GiB used; mounted at `/home/koyomi/lanraragi/content`.
  - `thumbs`: 4 GiB provisioned, about 76 MiB used; mounted at `/home/koyomi/lanraragi/thumbs`.
- The older Gluster claims `lanraragi-db` and `lanraragi-content` remain bound but are not mounted by the active Deployment. Do not treat them as authoritative without a separate content comparison.
- `manga.hybridoptix.net` currently enters through the Authentik embedded outpost rather than directly through the LANraragi Service. The replacement intentionally does not preserve that authentication layer yet.

### Target contract

- Build a reusable Onigiri LANraragi package and keep deployment-specific storage sizes, hostname, and route configuration in Yggdrasil.
- Run one replica with stable selectors, explicit CPU/memory requests, HTTP readiness/liveness checks, and three retained RWO Longhorn claims for database, content, and thumbnails.
- Migrate on LANraragi `v.0.9.60` and pin its image digest in `oni.lock`. Treat a later application upgrade as separate work.
- Preserve the database and content exactly. Thumbnails are copied by default, even if they may be regenerable, unless testing proves regeneration is safe and acceptable.
- Expose `manga.itai.moe` only through the internal Gateway. Do not attach the route to the public Gateway or put Authentik in front of it during this migration.
- Keep the legacy instance stopped but recoverable during final synchronization and validation; do not allow both instances to write to divergent copies.

### Cutover checklist

- [x] Inventory the active Deployment, image, Service, probes, storage mounts, data sizes, and current routing path.
- [x] Select internal-only hostname `manga.itai.moe`.
- [x] Defer Authentik integration; route directly to LANraragi through the internal Gateway.
- [x] Select LANraragi `v.0.9.60` for migration; defer upgrades until after migration.
- [x] Test `v.0.9.60` with the copied Redis persistence files; Redis loaded the dataset and answered `PONG`.
- [x] Author the Onigiri package with retained database, content, and thumbnail claims, explicit resources, and no public route by default.
- [x] Lock the package and image digest, run `oni check`, and validate the desired resources.
- [x] Deploy the replacement with empty retained claims while leaving production traffic on the legacy instance.
- [x] Stop legacy LANraragi and copy all three mounted filesystems consistently. SHA-256 verification matched all 5 database files, 5,476 content files, and 1,246 thumbnail files.
- [x] Validate login/authentication, archive count and metadata, search, thumbnail display/regeneration, archive reading, uploads/imports, restart persistence, and route recovery.
- [x] Confirm internal DNS and intended-client access for `manga.itai.moe`; the internal Gateway route is Accepted, has resolved references, and returns HTTP 200.
- [x] Keep the legacy Deployment scaled to zero and retain its active Longhorn volumes for rollback.
- [ ] After the rollback window, separately classify and remove the two unmounted Gluster claims only after verification.

## Monitoring exporters and autoscaling plan

### Smokeping

- Legacy `monitoring/smokeping` runs `quay.io/superq/smokeping-prober:v0.8.1`, probes six public DNS addresses every second, and exposes port 9374 through LoadBalancer `172.16.32.115`.
- Replace it with Smokeping Prober `v0.12.0`, preserving targets `1.1.1.1`, `1.0.0.1`, `8.8.8.8`, `8.8.4.4`, `9.9.9.9`, and `149.112.112.112`.
- The replacement has a ClusterIP Service and pod scrape annotations for Alloy. It does not receive a LAN VIP.
- ICMP remains privileged because upstream documents privileged operation and `NET_RAW` alone is not reliably sufficient for this prober.
- [x] Inventory the legacy image, targets, interval, port, and exposure.
- [x] Author the reusable package and Yggdrasil configuration without deploying it.
- [ ] Lock and review the image digest and rendered resources.
- [ ] Deploy, confirm all targets produce fresh metrics, and validate Alloy remote-write samples before retiring the legacy Deployment and Service.

### Unpoller

- Legacy `monitoring/unifi-poller` runs `golift/unifi-poller:2.11.2`, reads credential-bearing configuration from Secret `unifi-poller`, and exposes TCP/UDP 9130 through LoadBalancer `172.16.32.113`.
- Replace it with `ghcr.io/unpoller/unpoller:v5.7.0`. Prometheus metrics use TCP 9130 through a ClusterIP Service and pod scrape annotations; no replacement LAN VIP or UDP Service is created.
- The package consumes Secret `unpoller-config`, key `up.conf`, mounted read-only. The `unpoller` package owns the ExternalSecret that syncs remote key `UNPOLLER_UP_CONF` from the Bitwarden ClusterSecretStore; no credential is stored in Git.
- The Bitwarden store namespace allow-list includes `monitoring` for this ExternalSecret.
- [x] Inventory the legacy image, Service, Secret shape, and configuration source.
- [x] Select Unpoller `v5.7.0` and cluster-internal metrics.
- [x] Author the workload package, with its bundled ExternalSecret, without copying the legacy credential into Git.
- [ ] Add a v5-compatible complete `up.conf` as Bitwarden key `UNPOLLER_UP_CONF` using a read-only UniFi account.
- [ ] Lock and review the image digest and rendered resources.
- [ ] Deploy and validate controller authentication, expected v5 metrics, Alloy scraping, dashboards, and alerts before retiring the legacy Deployment, Service, and Secret.

### Vertical Pod Autoscaler

- Legacy VPA `0.10.0` includes recommender and admission controller; its updater exists but is scaled to zero. Six legacy VPA objects mostly target workloads that are absent or intentionally replaced on Yggdrasil.
- Install VPA `1.8.1`, which supports Kubernetes 1.36, as recommendation-only infrastructure: exact upstream CRDs, recommender ServiceAccount/RBAC, recommender Deployment, and internal metrics endpoint.
- Do not install the updater or admission controller. Do not migrate legacy VPA objects. Introduce new VPA objects deliberately later with `updateMode: Off` while evaluating recommendations.
- [x] Inventory legacy components and VPA objects.
- [x] Select recommendation-only operation and VPA `1.8.1`.
- [x] Author exact upstream CRDs and recommender-only desired state.
- [ ] Lock and review the image digest, CRDs, RBAC, and rendered resources.
- [ ] Deploy and validate CRDs, Metrics API access, recommender health, checkpoint behavior, and Alloy scraping.
- [ ] Add individually reviewed recommendation-only VPA objects only for current Yggdrasil workloads.

### Another Autoscaler retirement

- Legacy `kube-system/another-autoscaler` runs `ghcr.io/dignajar/another-autoscaler:0.7` with cluster-wide Deployment update and patch privileges.
- No scheduling annotations consumed by Another Autoscaler were found on legacy or Yggdrasil workloads, and the project is old and apparently unmaintained.
- Do not migrate it. Confirm once more that no annotated consumer has appeared, then remove its Deployment, ConfigMap, ServiceAccount, ClusterRole, and ClusterRoleBinding from the legacy cluster.

## Lighthouse replacement plan

### Target topology

- Lighthouse `0.1.0` is sourced from the project-scoped development repository `../uptime/oni-packages`.
- Region `us-west` has three regional coordinators in a StatefulSet, each with a retained 5 GiB PVC and required hostname anti-affinity; locality is represented by agent zone `us-pnw`.
- Three global coordinators run in a separate StatefulSet, each with a retained 1 GiB catalogue PVC and required hostname anti-affinity.
- One in-cluster agent `yggdrasil-01` represents zone `us-pnw`; future GCP and Unraid agents join region `us-west` with their own locality-specific zones and credentials.
- Future independent Taiwan regions are named `tw-taoyuan` and `tw-new-taipei` and report to the global tier.
- `lighthouse.itai.moe` routes only to the global Service through shared Cilium Ingress `172.16.16.10`; public reverse-proxy exposure is deferred.
- The regional coordinator Service remains cluster-internal until a deliberate remote-agent connectivity design is selected.

### Deployment state

- [x] Author a format-2 Lighthouse package with source fragments.
- [x] Lock the package and `ghcr.io/crainte/lighthouse:0.1.0` image digest.
- [x] Pass `oni check` and Kubernetes server-side dry-run.
- [x] Deploy the namespace, Services, StatefulSets, agent Deployment, Certificate, Ingress, and retained PVCs.
- [x] Issue the Let's Encrypt certificate for `lighthouse.itai.moe`.
- [x] Add `LIGHTHOUSE_GLOBAL_PEER_TOKEN` and `LIGHTHOUSE_ADMIN_PASSWORD` to Bitwarden Secrets Manager.
- [x] Add `LIGHTHOUSE_COORDINATOR_PEER_TOKEN` to Bitwarden Secrets Manager.
- [x] Register region `us-west` through the global Admin UI and store its one-time registration key as `LIGHTHOUSE_REGION_KEY_US` for the regional coordinators only.
- [x] Mint the `us-pnw` / `us-west` agent credential through Lighthouse for agent `yggdrasil-01`, then store it as `LIGHTHOUSE_AGENT_TOKEN_YGGDRASIL` in Bitwarden Secrets Manager.
- [x] Validate all three coordinator peers, all three global peers, the in-cluster agent, and regional-to-global reporting.
- [x] Validate the global dashboard through `lighthouse.itai.moe` from the intended internal client path.
- [ ] Add GCP and Unraid agent credentials and choose a secure route to the regional coordinator API.
- [x] Complete the Kener replacement after Lighthouse checks and routing validation; application bugs are tracked independently.
- [ ] Decommission the legacy Kener Deployment and retain or remove its PVCs according to the chosen rollback window.

## Validation commands

```sh
oni check .
oni render . | kubectl --context yggdrasil apply --server-side --dry-run=server --field-manager=onigiri -f -
oni render . | kubectl --context yggdrasil apply --server-side --field-manager=onigiri -f -
```

Do not add `--force-conflicts` unless one specific conflict has been reviewed and intentional ownership transfer is required.
