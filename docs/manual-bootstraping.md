# Manual bootstraping

## On-premise bootstrap procedure

Make sure to set the default context in your kubeconfig to your cluster, then create the prerequisites:

```shell
export FLUX_GITHUB_USER=$(echo -n 'flux-bot-stratio' | base64 -w0)
export FLUX_GITHUB_PAT=$(echo -n 'github_pat_xxx' | base64 -w0)

cat <<EOF | kubectl apply -f -
---
apiVersion: v1
kind: Namespace
metadata:
  name: flux-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kustomize-controller
  namespace: flux-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-reconciler-flux-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kustomize-controller
  namespace: flux-system
---
apiVersion: v1
kind: Secret
metadata:
  annotations:
    replicator.v1.mittwald.de/replicate-to: .*
  name: flux-system
  namespace: flux-system
data:
  username: ${FLUX_GITHUB_USER}
  password: ${FLUX_GITHUB_PAT}
type: Opaque
EOF
```

Configure secrets decryption with SOPS, follow [this](https://fluxcd.io/flux/guides/mozilla-sops/) procedure to generate a GPG key, then create the following secret and add the annotation to the secret so it is available across namespaces:

```shell
gpg --export-secret-keys --armor "my-key-fingerprint" |
    kubectl create secret generic sops-gpg \
        --namespace=flux-system \
        --from-file=sops.asc=/dev/stdin
kubectl annotate secret sops-gpg --namespace flux-system replicator.v1.mittwald.de/replicate-to=".*"
```

Copy the cluster `_TEMPLATE` folder and replace the placeholders with the desired values:

```shell
export TARGET_DIR="keos-fleet/clusters/<CLUSTER_NAME>"
export CLUSTER_NAME="mycluster"

cp -r keos-fleet/clusters/_TEMPLATE "$TARGET_DIR"

find "$TARGET_DIR" -type f -exec sed -i \
    -e "s/_CLUSTER_NAME/${CLUSTER_NAME}$/g" \
    -e "s/_STRATIO_SIZE/S/g" \
    -e "s/_GIT_BRANCH/main/g" \
    -e "s/_CLUSTER_DOMAIN/k8s.eosdev2.labs.stratio.com/g" \
    -e "s/_CONTAINER_REGISTRY_URL/qa.int.stratio.com:11443/g" \
    -e "s/_CONTAINER_REGISTRY_REPOSITORY_PREFIX//g" \
    -e "s/_HELM_REGISTRY_PROVIDER/generic/g" \
    -e "s/_HELM_REGISTRY_TYPE/default/g" \
    -e "s/_HELM_REGISTRY_URL/http:\/\/qa.int.stratio.com/g" \
    -e "s/_HELM_REGISTRY_REPOSITORY_PREFIX/\/repository\/helm-all/g" \
    -e "s/_SOPS_SECRET_PROVIDED/true/g" {} +
```

Commit and push the changes to keos-fleet repository.

Finally, apply the flux-system manifests for the cluster:

```shell
kubectl kustomize clusters/<CLUSTER_NAME>/default/flux-system | kubectl apply -f -
```