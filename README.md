# Azure AKS RBAC

AKS authorization can be done in two ways. Both authenticate users with Microsoft Entra ID.

| Approach | Where permissions live | Use when |
|----------|------------------------|----------|
| **Azure RBAC for Kubernetes authorization** | Azure role assignments (`az role assignment create`) | You want one place (Azure) to manage access to the cluster and its namespaces |
| **Kubernetes RBAC with Entra groups** | `Role`/`RoleBinding` objects in the cluster ([`aks-rbac.yaml`](aks-rbac.yaml)) | You want standard Kubernetes RBAC, GitOps-managed, with Entra groups as subjects |

For pod-to-Azure access (as opposed to user access), use **Workload Identity** (section 3).

> Replace every `<placeholder>` and all-zero GUID with your own values. Nothing here has been run against a real subscription.

## Variables

```bash
RG=rg-app-demo
CLUSTER=aks-app-demo
LOCATION=westeurope
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
```

## 1. Option A: Azure RBAC for Kubernetes authorization

### Create the cluster

```bash
az group create -n $RG -l $LOCATION

az aks create -g $RG -n $CLUSTER \
  --enable-aad \
  --enable-azure-rbac \
  --aad-admin-group-object-ids <admin-group-object-id> \
  --enable-oidc-issuer --enable-workload-identity \
  --generate-ssh-keys
```

For an existing cluster: `az aks update -g $RG -n $CLUSTER --enable-aad --enable-azure-rbac`.

### Assign roles

Users need `Azure Kubernetes Service Cluster User Role` to download a kubeconfig, plus a data-plane role for what they can do.

| Built-in role | Access |
|---------------|--------|
| Azure Kubernetes Service Cluster User Role | `az aks get-credentials` (kubeconfig only, no permissions) |
| Azure Kubernetes Service RBAC Reader | Read most objects in a namespace, not Secrets |
| Azure Kubernetes Service RBAC Writer | Read/write most objects in a namespace |
| Azure Kubernetes Service RBAC Admin | Admin within a namespace, including Roles/RoleBindings |
| Azure Kubernetes Service RBAC Cluster Admin | Full control over the whole cluster |

```bash
AKS_ID=$(az aks show -g $RG -n $CLUSTER --query id -o tsv)

# Let the group fetch credentials
az role assignment create --assignee <group-object-id> \
  --role "Azure Kubernetes Service Cluster User Role" --scope $AKS_ID

# Write access limited to the app-demo namespace
az role assignment create --assignee <group-object-id> \
  --role "Azure Kubernetes Service RBAC Writer" \
  --scope $AKS_ID/namespaces/app-demo

# Read-only for auditors, same namespace
az role assignment create --assignee <auditor-group-object-id> \
  --role "Azure Kubernetes Service RBAC Reader" \
  --scope $AKS_ID/namespaces/app-demo
```

Role assignments can take a few minutes to propagate.

## 2. Option B: Kubernetes RBAC with Entra groups

Create the cluster with Entra integration but without `--enable-azure-rbac`:

```bash
az aks create -g $RG -n $CLUSTER --enable-aad \
  --aad-admin-group-object-ids <admin-group-object-id> --generate-ssh-keys
```

Get the group object IDs and put them in [`aks-rbac.yaml`](aks-rbac.yaml):

```bash
az ad group show --group "app-demo-developers" --query id -o tsv
```

Users still need the Cluster User Role from section 1 to get a kubeconfig. Then apply:

```bash
az aks get-credentials -g $RG -n $CLUSTER --admin   # admin credentials for bootstrap only
kubectl apply -f k8s/namespace.yaml
kubectl apply -f rbac/aks/aks-rbac.yaml
```

| Kind | Name | Purpose |
|------|------|---------|
| Role + RoleBinding | `app-demo-developer` | Entra group can manage pods, services, deployments, exec and port-forward in `app-demo` |
| RoleBinding | `app-demo-auditor-binding` | Entra group gets the built-in `view` ClusterRole, limited to `app-demo` |
| ServiceAccount | `nodeapp-sa` | Annotated for Workload Identity |

Do not use `--admin` for day-to-day work; it bypasses Entra ID.

## 3. Workload Identity for the app pods

Gives the pod an Azure identity without storing credentials. Requires `--enable-oidc-issuer --enable-workload-identity` on the cluster.

```bash
# Managed identity for the app
az identity create -g $RG -n id-nodeapp
CLIENT_ID=$(az identity show -g $RG -n id-nodeapp --query clientId -o tsv)
ISSUER=$(az aks show -g $RG -n $CLUSTER --query oidcIssuerProfile.issuerUrl -o tsv)

# Trust the Kubernetes ServiceAccount
az identity federated-credential create -g $RG --identity-name id-nodeapp \
  --name nodeapp-fed --issuer $ISSUER \
  --subject system:serviceaccount:app-demo:nodeapp-sa \
  --audience api://AzureADTokenExchange

# Example Azure permission: read secrets from a Key Vault
az role assignment create --assignee $CLIENT_ID \
  --role "Key Vault Secrets User" \
  --scope $(az keyvault show -n <vault-name> --query id -o tsv)
```

Set the `client-id` annotation on `nodeapp-sa` to `$CLIENT_ID`, and add the label to the pod template in [`k8s/app-deployment.yaml`](../../k8s/app-deployment.yaml):

```yaml
spec:
  template:
    metadata:
      labels:
        app: nodeapp
        azure.workload.identity/use: "true"
```

## Verification

```bash
az aks get-credentials -g $RG -n $CLUSTER
kubelogin convert-kubeconfig -l azurecli     # needed for Entra auth with kubectl

kubectl auth can-i create deployments -n app-demo   # yes for developers/writers
kubectl auth can-i create deployments -n default    # no (namespace-scoped grant)
kubectl get pods -n app-demo
```

To list assignments: `az role assignment list --scope $AKS_ID --all -o table`.

## Cleanup

```bash
az role assignment delete --assignee <group-object-id> --scope $AKS_ID/namespaces/app-demo
kubectl delete -f rbac/aks/aks-rbac.yaml
az group delete -n $RG --yes --no-wait   # deletes the whole resource group
```
