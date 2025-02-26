# OVERRIDES

## Version de un artefacto

## Chart de Helm gestionada por flux

1.

## CR gestionado por flux

1. Deshabilitamos el despligue por defecto en fichero clusters/eosdev2/base/apps-domain.yaml:

```
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-apps-domain-components
  namespace: flux-system
spec:
  # ...omitted for brevity
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

1. Copiamos el despliegue en clusters/eosdev2/overrides/apps

```
cp -r domain/apps/talk-to-your-data/components/postgres clusters/eosdev2/override/apps
```

1. Editamos la label del fichero kustomization clusters/eosdev2/override/apps/postgres/kustomization.yaml

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

1. Modificamos clusters/eosdev2/overrides/apps/postgres/sync.yaml, añadiendo el patch que queramos hacer:

```
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-postgres
spec:
  # ...omitted for brevity
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

