# Yggdrasil

Onigiri project for the Yggdrasil Kubernetes cluster.

## Project model

Packages own Kubernetes desired state; this project selects and configures them.

* `oni.yaml` selects package instances, their namespaces, and their configuration files.
* Every other YAML file (besides `oni.lock`) configures one selected instance: typed package inputs, plus optional format-2 map-by-name patches of resources the package already declares (`deployments: {knot: {...}}`). Configuration cannot create resources; anything new belongs in a package.
* Secrets are provisioned by packages, not by loose manifests. Bifrost owns its `ExternalSecret`; `postgresql-credentials` provisions the Secret the generic `postgresql` package consumes (its name and key must match `postgresql.yaml`).

## Namespaces

Root instances set `namespace:`; roots with `createNamespace: true` make Onigiri generate the project-owned Namespace (for example `Namespace/dns` via `knot-resolver`). That Namespace is not owned by any reusable package. It is ensured before namespaced resources, participates in lifecycle readiness, and is retained: removing the root or the flag stops managing it but never deletes it.

## Locking

Container image tags declared by packages are pinned to registry digests in `oni.lock` (`images:`). `oni lock` needs registry access only for tags that are not locked yet; `oni check` and `oni render` never contact a registry. Re-resolve a moved tag explicitly with `oni update . --image <reference|all>`.

The package and Kubernetes schema repositories are consumed from their published Git remotes (`git:` in `oni.yaml`). Their `repository.yaml` manifests provide the durable lock identities `omusubiya-packages` and `onigiri-schema-registry`, independent of transport. Only explicit resolution (`oni lock`, `oni update`, `oni outdated`) fetches them; `oni render`, `oni check`, and `oni plan` work offline from `oni.lock` and the local immutable artifact store.

To test unpublished package or schema changes from sibling checkouts without editing `oni.yaml`:

```sh
oni lock . \
  --repository-path packages=../omusubiya/packages \
  --repository-path schemas=../schema-registry
```
