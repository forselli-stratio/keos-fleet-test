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
export KEY_FP="my-key-fingerprint"
gpg --export-secret-keys --armor "${KEY_FP}" |
    kubectl create secret generic sops-gpg \
        --namespace=flux-system \
        --from-file=sops.asc=/dev/stdin
kubectl annotate secret sops-gpg --namespace flux-system replicator.v1.mittwald.de/replicate-to=".*"
```

Copy the cluster `_TEMPLATE` folder and replace the placeholders with the desired values:

```shell
export CLUSTER_NAME="mycluster"
export TARGET_DIR="clusters/${CLUSTER_NAME}>"

cp -r clusters/_TEMPLATE "$TARGET_DIR"

find "$TARGET_DIR" -type f -exec sed -i \
    -e "s/_CLUSTER_NAME/${CLUSTER_NAME}/g" \
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

Configure SOPS:

```shell
cat <<EOF >"${TARGET_DIR}"/secrets/.sops.yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    pgp: ${KEY_FP}
EOF
gpg -a --export $KEY_FP > "${TARGET_DIR}"/secrets/.sops.pub.asc
```

Edit the secret for dg-s3-agent with a valid one and encrypt it using SOPS (follow the [instructions for a provided secret in secrets-operator repo](https://github.com/Stratio/secrets-operator?tab=readme-ov-file#synchronizing-custom-secrets)):

```shell
cat <<EOF >"${TARGET_DIR}"/secrets/dg-s3-agent/secret.yaml
apiVersion: v1
kind: Secret
metadata:
    name: customer-s3-secret
data:
    bundle: $DG_S3_AGENT_SECRET_BUNDLE
EOF

sops --encrypt --config "${TARGET_DIR}"/secrets/.sops.yaml --in-place "${TARGET_DIR}"/secrets/dg-s3-agent/secret.yaml
```

Commit and push the changes to keos-fleet repository.

Finally, apply the flux-system manifests for the cluster:

```shell
kubectl kustomize clusters/<CLUSTER_NAME>/default/flux-system | kubectl apply -f -
```