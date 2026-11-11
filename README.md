# Flux configuration

GitOps configuration for the `fium` Kubernetes cluster. Flux reads this repository and reconciles the cluster with the manifests under `clusters/fium`.

## How it works

The configuration contains two Flux resources:

- A `GitRepository` checks the public `main` branch every minute over HTTPS, without credentials.
- A `Kustomization` builds and applies `clusters/fium` when the Git revision changes. It also reconciles every ten minutes to correct drift.

The cluster directory includes the Flux controllers and these two resources, so Flux manages its own installation.

Pruning is enabled. Removing a managed resource from this configuration also removes it from the cluster.

## Applications

Each file in `clusters/fium/apps` holds one application: a `GitRepository` for the application repository and a `HelmRelease` that installs its `infra/chart`. Flux applies chart changes from every commit on the application's `main` branch.

The `HelmRelease` values select the deployed image. An image push alone does not deploy it. To deploy another build, change `image.tag` in the application file and publish the change to `main`. Keep the tag quoted, because a short commit SHA can contain only digits.

Drift detection is enabled. Flux reverts manual changes to the resources of a chart on the next reconciliation.

Removing an application file uninstalls its Helm release.

Application credentials are managed through OpenBao and External Secrets. Git contains configuration and Secret references, while credential values remain outside the repository.

## Repository layout

All cluster configuration lives under `clusters/fium`:

| File | Purpose |
| --- | --- |
| `kustomization.yaml` | Cluster entry point. Includes the `flux-system` and `apps` directories. |
| `flux-system/kustomization.yaml` | Lists the Flux manifests to apply. |
| `flux-system/flux-components.yaml` | Generated Flux v2.10.0 controllers, CRDs, RBAC, and network policies. |
| `flux-system/repo.yaml` | Git source URL, branch, and polling interval. |
| `flux-system/reconciliation.yaml` | Reconciliation path, interval, and pruning settings. |
| `apps/kustomization.yaml` | Lists the application files to apply. |
| `apps/<name>.yaml` | Chart source and Helm release of one application. |

## Check the deployment

Use the Flux v2.10.0 CLI from the NixOS Kubernetes client tools. Run these commands inside the cluster's kubie session:

```sh
flux check
flux get sources git
flux get kustomizations
```

The source and Kustomization report `READY=True` when Flux has fetched and applied the Git revision. Their `REVISION` fields identify the commit.

To render the configuration locally without changing the cluster, run this from the repository root:

```sh
kubectl kustomize clusters/fium
```

## Update the Flux installation

The installation manifest is generated with the matching Flux CLI version. From the repository root:

```sh
flux install --version=v2.10.0 --export > clusters/fium/flux-system/flux-components.yaml
```

Flux applies the updated controllers and CRDs after the manifest is published to `main`.

## Bootstrap

The initial installation requires cluster-admin access and a published `main` branch. Run these commands from the repository root inside the target cluster's kubie session.

Confirm the active cluster:

```sh
kubectl config current-context
```

Install the Flux components, wait for the source and reconciliation APIs, then configure Git sync:

```sh
kubectl apply --server-side --field-manager=flux-client-side-apply -f clusters/fium/flux-system/flux-components.yaml
kubectl wait --for=condition=Established --timeout=120s crd/gitrepositories.source.toolkit.fluxcd.io crd/kustomizations.kustomize.toolkit.fluxcd.io
kubectl apply --server-side --field-manager=flux-client-side-apply -f clusters/fium/flux-system/repo.yaml -f clusters/fium/flux-system/reconciliation.yaml
```

After bootstrap, changes published to `main` are reconciled automatically.
