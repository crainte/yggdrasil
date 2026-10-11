# Yggdrasil

Onigiri project for the Yggdrasil Kubernetes cluster.

This project currently describes Knot Resolver in the `dns` namespace. The namespace itself is intentionally not owned by the Knot package or this initial project revision and must exist before deployment.

Container image tags declared by packages are pinned to registry digests in `oni.lock` (`images:`). `oni lock` needs registry access only for tags that are not locked yet; `oni check` and `oni render` never contact a registry. Re-resolve a moved tag explicitly with `oni update . --image <reference|all>`.

The package and Kubernetes schema repositories are referenced by local filesystem paths for authoring. Their `repository.yaml` manifests provide the durable lock identities `omusubiya-packages` and `onigiri-schema-registry`, independent of checkout paths.
