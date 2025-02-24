# OVERRIDES

## Chart de Helm gestionada por flux

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

1. Añadimos un nuevo bloque de overrides:

```
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-apps-domain-components-overrides
  namespace: flux-system
spec:
  serviceAccountName: kustomize-controller
  dependsOn:
    - name: flux-apps-domain-base
  interval: 5m
  retryInterval: 3m
  path: ./clusters/eosdev2/overrides/apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: flux-runtime-info
```

1. Creamos nuestro directorio de overrides en clusters/eosdev2/overrides

1. Creamos nuestro despliegue modificado en clusters/eosdev2/overrides/apps

```
mkdir -p clusters/eosdev2/overrides/apps
cp -r domain/apps/components/postgres clusters/eosdev2/overrides/apps
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

