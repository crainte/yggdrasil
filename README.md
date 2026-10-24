# Yggdrasil

Onigiri project for the Yggdrasil Kubernetes cluster.

## Project model

Packages own Kubernetes desired state; this project selects and configures them.

* `oni.yaml` selects package instances, their namespaces, and their configuration files.
* Every other YAML file (besides `oni.lock`) configures one selected instance: typed package inputs, plus optional format-2 map-by-name patches of resources the package already declares (`deployments: {knot: {...}}`). Configuration cannot create resources; anything new belongs in a package.
* Secrets are provisioned by packages, not by loose manifests. Each package that needs a secret owns its `ExternalSecret` (Bifrost, PostgreSQL, Immich, Unpoller) and takes the store and remote key as inputs.

## Namespaces

Root instances set `namespace:`; roots with `createNamespace: true` make Onigiri generate the project-owned Namespace (for example `Namespace/dns` via `knot-resolver`). That Namespace is not owned by any reusable package. It is ensured before namespaced resources, participates in lifecycle readiness, and is retained: removing the root or the flag stops managing it but never deletes it.

## Locking

Root `version:` constraints name the software each package represents (`cert-manager: ^1.21`, `postgresql: ^16`); packages that represent no independently versioned software (`application: none`: routes, issuers, secret stores, credentials) are constrained by `revision:` instead. `oni.lock` freezes the exact Oni package revision, its version, and its artifact digest, so revision constraints are only added where the project deliberately restricts packaging revisions.

Container image tags declared by packages are pinned to registry digests in `oni.lock` (`images:`). `oni lock` needs registry access only for tags that are not locked yet; `oni check` and `oni render` never contact a registry. Re-resolve a moved tag explicitly with `oni update . --image <reference|all>`.

Packages and Kubernetes schema platforms come from Oni's built-in official repositories (`onigiri` and `kubernetes`), so `oni.yaml` declares no repositories. Their `repository.yaml` manifests provide the durable lock identities `omusubiya-packages` and `onigiri-schema-registry`, independent of transport (`oni repo list` shows the registrations; `oni repo set-remote` redirects them to a mirror). Only explicit resolution (`oni lock`, `oni update`, `oni outdated`) fetches them; `oni render`, `oni check`, and `oni plan` work offline from `oni.lock` and the local immutable artifact store.

To test unpublished package or schema changes from sibling checkouts without editing `oni.yaml`:

```sh
oni lock . \
  --repository-path onigiri=../omusubiya/packages \
  --repository-path kubernetes=../schema-registry
```
