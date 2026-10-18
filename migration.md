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
| Gateway front doors | Install | Complete | Internal Gateway `172.16.16.10` and public NAT-target Gateway `172.16.16.11` serve wildcard `*.itai.moe` TLS. Legacy Cilium Ingress remains on `172.16.16.12` for rollback. |
| Metrics Server | Install | Complete | Metrics Server 0.9.0 runs two anti-affined replicas; Metrics API and `kubectl top` are operational. |
| Grafana Alloy | Migrate | Complete | Alloy chart 1.13.1 / app v1.20.1; metrics-only; remote write to `10.10.10.189:9090`; no logs or SigNoz. |
| Chrony/NTP | Replace | Cutover pending | Replacement is healthy at `172.16.16.123`; change clients and observe traffic before retiring `172.16.32.109`. |
| ntfy | Migrate | Internal route live | `ntfy.itai.moe` routes through the internal Gateway; SQLite data was copied. Client/Cloudflare cutover and legacy retirement remain pending. |
| Grafana | Migrate | Pending | Preserve required dashboards, data sources, hostname, and persistence. |
| Authentik | Migrate | Pending | Database is authoritative for internally managed providers, applications, and outposts. Use a supported upgrade path. |
| Docker Registry | Migrate | Pending | Preserve hostname, credentials, and required image blobs. |
| Immich | Reprovision | Deployed, setup pending | Fresh Immich `v3.3.3` serves internal `immich.itai.moe`. Create the admin account, then consider `IMMICH_ALLOW_SETUP=false`. Legacy data is intentionally abandoned; legacy workloads remain until retired. |
| Kener | Replace | Complete | Lighthouse is validated at `lighthouse.itai.moe` and routes through the public Gateway; application bugs will be handled independently. Legacy Kener decommissioning remains a cleanup action. |
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
- `ntfy.itai.moe` attaches to the internal Gateway; `lighthouse.itai.moe` attaches to the public Gateway. Both routes return HTTP 200 from their selected VIPs.
- The router must NAT public TCP/443 to `172.16.16.11`; DNS for internal names must resolve to `172.16.16.10`.
- Cilium 1.20.2 and 1.20.3 produced empty Envoy EDS clusters when an HTTPRoute used a numeric backend port against a named Service port. Packages therefore own Gateway compatibility Services with unnamed ports: `ntfy-gateway-backend`, `lighthouse-gateway-backend`, and `immich-gateway-backend`.
- The previous shared Cilium Ingress is retained temporarily at `172.16.16.12` for rollback. Remove Ingress resources only after DNS, NAT, and client paths are validated.
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
