# Overrides

## Helm Chart managed by flux

1. Remove the default kustomization file for overrides:

```shell
rm clusters/<CLUSTER_NAME>/override/system-services/kustomization.yaml
```

1. Add the values configmap with the desired configuration, for example:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: 02-gosec-helm-chart-override-values
  namespace: keos-core
data:
  values.yaml: |-
    ---
    gosec-authz:
      config:
        DYPLON_CACHE_ENABLED: "true"
```

## CR managed by flux

1. Disable the default deployment in the file clusters/<CLUSTER_NAME>/base/apps-domain.yaml:

```
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-apps-domain-components
  namespace: flux-system
spec:
  # ...existing code...
  patches:
    - patch: |
        $patch: delete
        apiVersion: all
        kind: all
        metadata:
          name: all
      target:
        labelSelector: toolkit.fluxcd.io/component=postgres
```

1. Copy the deployment to be edited to clusters/<CLUSTER_NAME>/overrides/apps:

```
cp -r domain/apps/talk-to-your-data/components/postgres clusters/<CLUSTER_NAME>/override/apps
```

1. Edit the label in the kustomization file clusters/<CLUSTER_NAME>/override/apps/postgres/kustomization.yaml:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: stratio-datastores
resources:
  - rbac.yaml
  - namespace.yaml
  - sync.yaml
labels:
  - pairs:
      toolkit.fluxcd.io/domain: apps
      toolkit.fluxcd.io/component: postgres-override
```

1. Modify clusters/<CLUSTER_NAME>/overrides/apps/postgres/sync.yaml, adding the desired patch:

```
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-postgres
spec:
  # ...existing code...
  patches:
    - patch: |
        - op: replace
          path: /spec/instances
          value: 0
      target:
        kind: PgCluster
        name: psql
        namespace: stratio-datastores
```
