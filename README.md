# Yggdrasil

Onigiri project for the Yggdrasil Kubernetes cluster.

This project owns the retained `dns` Namespace and describes Knot Resolver within it. Namespace lifecycle is cluster-level project intent and remains separate from the reusable Knot package.

Container image tags declared by packages are pinned to registry digests in `oni.lock` (`images:`). `oni lock` needs registry access only for tags that are not locked yet; `oni check` and `oni render` never contact a registry. Re-resolve a moved tag explicitly with `oni update . --image <reference|all>`.

The package and Kubernetes schema repositories are referenced by local filesystem paths for authoring. Their `repository.yaml` manifests provide the durable lock identities `omusubiya-packages` and `onigiri-schema-registry`, independent of checkout paths.
