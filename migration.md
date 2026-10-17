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
| Grafana Alloy | Migrate | Complete | Alloy chart 1.13.1 / app v1.20.1; metrics-only; remote write to `10.10.10.189:9090`; no logs or SigNoz. |
| Chrony/NTP | Replace | In progress | Replace legacy `ntp/chronyd`; publish UDP/123 at `172.16.16.123`; validate clients before retiring `172.16.32.109`. |
| ntfy | Migrate | Pending | Inspect live state, persistence, hostname, and dependencies before packaging. |
| Grafana | Migrate | Pending | Preserve required dashboards, data sources, hostname, and persistence. |
| Authentik | Migrate | Pending | Database is authoritative for internally managed providers, applications, and outposts. Use a supported upgrade path. |
| Docker Registry | Migrate | Pending | Preserve hostname, credentials, and required image blobs. |
| Immich | Reprovision | Pending | Build cleanly on Yggdrasil rather than migrating existing application data. |
| Kener | Replace | Pending | A separately developed application will replace it. |
| SigNoz | Retire | Pending | Do not migrate. Confirm no remaining producers or consumers before removal. |
| OneDev | Retire | Pending | Test installation only; do not migrate. |
| MariaDB | Retire candidate | Pending | Home Assistant uses SQLite. Confirm no other consumers before removal. |
| Mosquitto | Retire candidate | Pending | No known clients; verify before removal. |
| Legacy DNS | Retire candidate | Pending | DHCP is believed to have cut over. Validate active client configuration and traffic first. |
| Gluster storage | Retire candidate | Pending | Use currently mounted claims as authority, not historical claims. The 250 GiB `default/home` PVC appears unused. |

## Completed work

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
- [ ] Deploy and verify UDP/123 at `172.16.16.123` from a LAN client.
- [ ] Change DHCP/NTP consumers to the new address.
- [ ] Observe traffic and synchronization before retiring legacy `172.16.32.109`.

## Validation commands

```sh
oni check .
oni render . | kubectl --context yggdrasil apply --server-side --dry-run=server --field-manager=onigiri -f -
oni render . | kubectl --context yggdrasil apply --server-side --field-manager=onigiri -f -
```

Do not add `--force-conflicts` unless one specific conflict has been reviewed and intentional ownership transfer is required.
