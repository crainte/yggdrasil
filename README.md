# Yggdrasil

Onigiri project for the Yggdrasil Kubernetes cluster.

This project describes Knot Resolver in the `dns` Namespace. The `knot-resolver` root sets `namespace: dns` and `createNamespace: true`, so Onigiri generates the project-owned `Namespace/dns`. That Namespace is not owned by the reusable Knot package. It is ensured before namespaced resources, participates in lifecycle readiness, and is retained: removing the root or the flag stops managing it but never deletes it.

Container image tags declared by packages are pinned to registry digests in `oni.lock` (`images:`). `oni lock` needs registry access only for tags that are not locked yet; `oni check` and `oni render` never contact a registry. Re-resolve a moved tag explicitly with `oni update . --image <reference|all>`.

The package and Kubernetes schema repositories are referenced by local filesystem paths for authoring. Their `repository.yaml` manifests provide the durable lock identities `omusubiya-packages` and `onigiri-schema-registry`, independent of checkout paths.
