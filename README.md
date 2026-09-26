# Azure AKS Workload Identity with Azure Key Vault

This project demonstrates how to securely access secrets stored in **Azure Key Vault** from an **Azure Kubernetes Service (AKS)** workload using:

* Azure Kubernetes Service (AKS)
* Azure Key Vault
* Azure Workload Identity
* Microsoft Entra ID / OIDC
* Secrets Store CSI Driver
* Azure Key Vault Provider for Secrets Store CSI Driver
* User Assigned Managed Identity
* Kubernetes ServiceAccount

The implementation avoids storing Azure credentials, client secrets, or Key Vault secrets directly inside Kubernetes manifests.

---

## Architecture

```text
                         ┌──────────────────────────┐
                         │      Azure Key Vault     │
                         │                          │
                         │  secret1 = HelloFrom...  │
                         └────────────┬─────────────┘
                                      │
                                      │ RBAC
                                      │ Key Vault
                                      │ Secrets User
                                      ▼
                         ┌──────────────────────────┐
                         │ User Assigned Managed    │
                         │ Identity                 │
                         │                          │
                         │ Client ID                │
                         │ Principal ID             │
                         └────────────┬─────────────┘
                                      │
                              Federated Identity
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │      AKS OIDC Issuer     │
                         │                          │
                         │ ServiceAccount token     │
                         └────────────┬─────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                           AKS Cluster                            │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Kubernetes Pod                                            │  │
│  │                                                           │  │
│  │ ServiceAccount                                             │  │
│  │ workload-identity-sa                                      │  │
│  │                                                           │  │
│  │ Label:                                                    │  │
│  │ azure.workload.identity/use=true                          │  │
│  │                                                           │  │
│  │              ┌──────────────────────────────┐             │  │
│  │              │ Secrets Store CSI Driver     │             │  │
│  │              └──────────────┬───────────────┘             │  │
│  │                             │                             │  │
│  │                             ▼                             │  │
│  │                  /mnt/secrets-store/                       │  │
│  │                             │                             │  │
│  │                         secret1                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

# Why This Architecture?

A common but insecure approach is to store cloud credentials or application secrets directly inside Kubernetes.

For example:

```yaml
env:
  - name: AZURE_CLIENT_SECRET
    value: "my-secret"
```

This creates several problems:

* Secrets can end up in Git repositories.
* Kubernetes manifests may expose sensitive information.
* Long-lived Azure credentials are difficult to rotate.
* Developers and applications may have more privileges than necessary.
* Credential management becomes an operational burden.

This project uses **Workload Identity** instead.

The workload receives an identity based on its Kubernetes ServiceAccount. Azure verifies the ServiceAccount's OIDC token and exchanges that trust relationship for access to Azure resources.

The application therefore does **not** need:

```text
Client Secret
Password
Certificate
Azure CLI credentials
Static Azure credentials
```

---

# Components

| Component                      | Purpose                                              |
| ------------------------------ | ---------------------------------------------------- |
| AKS                            | Managed Kubernetes cluster                           |
| Azure Key Vault                | Secure secret storage                                |
| Workload Identity              | Provides Azure identity to Kubernetes workloads      |
| OIDC Issuer                    | Establishes trust between AKS and Microsoft Entra ID |
| User Assigned Managed Identity | Azure identity used by the workload                  |
| Federated Identity Credential  | Maps Kubernetes ServiceAccount to Azure identity     |
| Secrets Store CSI Driver       | Mounts external secrets into Kubernetes Pods         |
| Azure Key Vault Provider       | Retrieves secrets from Key Vault                     |
| Kubernetes ServiceAccount      | Identity associated with the workload                |
| SecretProviderClass            | Defines which Key Vault objects should be mounted    |

---

# Prerequisites

Install the following tools:

### Azure CLI

Verify:

```bash
az version
```

### kubectl

```bash
kubectl version --client
```

### Azure Account

Login:

```bash
az login
```

Verify the current subscription:

```bash
az account show
```

---

# Project Variables

The following environment variables are used throughout the project.

```bash
export SUBSCRIPTION_ID="<YOUR_SUBSCRIPTION_ID>"
export RESOURCE_GROUP="keyvault-demo"
export LOCATION="eastus"

export CLUSTER_NAME="keyvault-demo-cluster"

export KEYVAULT_NAME="aks-demo-chetan"

export UAMI="azurekeyvaultsecretsprovider-keyvault-demo-cluster"

export FEDERATED_IDENTITY_NAME="aksfederatedidentity"

export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SERVICE_ACCOUNT_NAMESPACE="default"
```

Set the Azure subscription:

```bash
az account set --subscription "$SUBSCRIPTION_ID"
```

> **Security:** Never commit your real subscription ID, credentials, secrets, or other sensitive values unnecessarily into Git.

---

# 1. Register Azure Resource Providers

Register the required Azure providers:

```bash
az provider register \
  --namespace Microsoft.ContainerService \
  --wait

az provider register \
  --namespace Microsoft.KeyVault \
  --wait
```

Verify:

```bash
az provider show \
  --namespace Microsoft.ContainerService \
  --query registrationState

az provider show \
  --namespace Microsoft.KeyVault \
  --query registrationState
```

Expected:

```text
"Registered"
```

---

# 2. Create Resource Group

```bash
az group create \
  --name "$RESOURCE_GROUP" \
  --location "$LOCATION"
```

Verify:

```bash
az group show \
  --name "$RESOURCE_GROUP"
```

---

# 3. Create AKS Cluster

Create the AKS cluster with:

* Azure Key Vault Secrets Provider addon
* OIDC issuer
* Workload Identity

```bash
az aks create \
  --name "$CLUSTER_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --node-count 1 \
  --node-vm-size Standard_D2s_v7 \
  --enable-addons azure-keyvault-secrets-provider \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --generate-ssh-keys
```

### Why these options matter

#### Azure Key Vault Secrets Provider

```text
--enable-addons azure-keyvault-secrets-provider
```

Installs/configures the Secrets Store CSI Driver and Azure Key Vault provider integration.

#### OIDC

```text
--enable-oidc-issuer
```

Provides the OIDC issuer required for Workload Identity.

#### Workload Identity

```text
--enable-workload-identity
```

Enables Azure Workload Identity support in AKS.

---

# 4. Configure kubectl

Retrieve the cluster credentials:

```bash
az aks get-credentials \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --overwrite-existing
```

Verify:

```bash
kubectl get nodes
```

Expected:

```text
NAME                              STATUS   ROLES    AGE   VERSION
aks-...                           Ready    <none>   ...   v...
```

---

# 5. Verify Secrets Store CSI Driver

Check the relevant Pods:

```bash
kubectl get pods \
  -n kube-system \
  -l 'app in (secrets-store-csi-driver,secrets-store-provider-azure)' \
  -o wide
```

You should see the CSI driver/provider Pods running.

You can also inspect all related Pods:

```bash
kubectl get pods -n kube-system | grep -E 'csi|secrets'
```

---

# 6. Create Azure Key Vault

Create the Key Vault with Azure RBAC authorization:

```bash
az keyvault create \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --location "$LOCATION" \
  --enable-rbac-authorization
```

Get the Key Vault resource ID:

```bash
export KEYVAULT_SCOPE=$(az keyvault show \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query id \
  -o tsv | tr -d '\r')
```

Verify:

```bash
echo "$KEYVAULT_SCOPE"
```

---

# 7. Grant User Permission to Create Secrets

Get the currently signed-in user's Object ID:

```bash
export CURRENT_USER_OID=$(az ad signed-in-user show \
  --query id \
  -o tsv | tr -d '\r')
```

Assign the Key Vault Secrets Officer role:

```bash
az role assignment create \
  --role "Key Vault Secrets Officer" \
  --assignee-object-id "$CURRENT_USER_OID" \
  --assignee-principal-type User \
  --scope "$KEYVAULT_SCOPE"
```

This allows the current user to create/manage secrets.

---

# 8. Create a Test Secret

Azure RBAC propagation may take some time.

```bash
sleep 30
```

Create the secret:

```bash
az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name "secret1" \
  --value "HelloFromKeyVault"
```

Verify that the secret exists:

```bash
az keyvault secret show \
  --vault-name "$KEYVAULT_NAME" \
  --name "secret1" \
  --query name
```

Expected:

```text
"secret1"
```

> Avoid displaying the secret value in production environments or CI/CD logs.

---

# 9. Create User Assigned Managed Identity

Create the identity:

```bash
az identity create \
  --name "$UAMI" \
  --resource-group "$RESOURCE_GROUP"
```

Retrieve the required identity information:

```bash
export USER_ASSIGNED_CLIENT_ID=$(az identity show \
  --resource-group "$RESOURCE_GROUP" \
  --name "$UAMI" \
  --query clientId \
  -o tsv | tr -d '\r')
```

```bash
export USER_ASSIGNED_OBJECT_ID=$(az identity show \
  --resource-group "$RESOURCE_GROUP" \
  --name "$UAMI" \
  --query principalId \
  -o tsv | tr -d '\r')
```

Retrieve the AKS tenant:

```bash
export IDENTITY_TENANT=$(az aks show \
  --name "$CLUSTER_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query identity.tenantId \
  -o tsv | tr -d '\r')
```

Retrieve the AKS OIDC issuer:

```bash
export AKS_OIDC_ISSUER=$(az aks show \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --query "oidcIssuerProfile.issuerUrl" \
  -o tsv | tr -d '\r')
```

Verify:

```bash
echo "$USER_ASSIGNED_CLIENT_ID"
echo "$USER_ASSIGNED_OBJECT_ID"
echo "$IDENTITY_TENANT"
echo "$AKS_OIDC_ISSUER"
```

---

# 10. Grant Key Vault Read Permission

The workload identity only needs to **read** secrets.

Assign:

```text
Key Vault Secrets User
```

to the User Assigned Managed Identity:

```bash
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee-object-id "$USER_ASSIGNED_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --scope "$KEYVAULT_SCOPE"
```

This is an important security principle:

```text
Developer/User
     │
     └── Key Vault Secrets Officer
             │
             └── Can manage secrets

AKS Workload Identity
     │
     └── Key Vault Secrets User
             │
             └── Can read secrets
```

The application does not receive permission to manage Key Vault secrets.

---

# 11. Create Federated Identity Credential

This is the key component that connects Kubernetes identity to Azure identity.

Create the federated credential:

```bash
az identity federated-credential create \
  --name "$FEDERATED_IDENTITY_NAME" \
  --identity-name "$UAMI" \
  --resource-group "$RESOURCE_GROUP" \
  --issuer "${AKS_OIDC_ISSUER}" \
  --subject "system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}"
```

The important relationship is:

```text
AKS OIDC Issuer
       +
Kubernetes ServiceAccount
       +
Federated Identity Credential
       ↓
User Assigned Managed Identity
```

The subject is:

```text
system:serviceaccount:default:workload-identity-sa
```

This means the Azure identity can be used by this specific Kubernetes ServiceAccount.

---

# 12. Create Kubernetes ServiceAccount

Create the ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workload-identity-sa
  namespace: default
  annotations:
    azure.workload.identity/client-id: <USER_ASSIGNED_CLIENT_ID>
```

The important annotation is:

```yaml
azure.workload.identity/client-id
```

It tells Azure Workload Identity which Azure User Assigned Managed Identity should be associated with this ServiceAccount.

---

# 13. Create SecretProviderClass

The `SecretProviderClass` defines which Azure Key Vault secrets should be retrieved.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-wi
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "<USER_ASSIGNED_CLIENT_ID>"
    keyvaultName: "<KEYVAULT_NAME>"
    tenantId: "<TENANT_ID>"
    objects: |
      array:
        - |
          objectName: secret1
          objectType: secret
          objectVersion: ""
```

Important parameters:

| Parameter        | Purpose                                          |
| ---------------- | ------------------------------------------------ |
| `provider`       | Specifies Azure Key Vault provider               |
| `usePodIdentity` | Disabled because Workload Identity is being used |
| `clientID`       | User Assigned Managed Identity client ID         |
| `keyvaultName`   | Azure Key Vault name                             |
| `tenantId`       | Microsoft Entra tenant ID                        |
| `objectName`     | Key Vault secret name                            |
| `objectType`     | Type of Key Vault object                         |

---

# 14. Deploy Test Pod

The Pod uses the ServiceAccount:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-secrets-store-inline-wi
  namespace: default
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: workload-identity-sa

  containers:
    - name: busybox
      image: registry.k8s.io/e2e-test-images/busybox:1.29-4

      command:
        - "/bin/sleep"
        - "10000"

      volumeMounts:
        - name: secrets-store01-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true

  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true

        volumeAttributes:
          secretProviderClass: "azure-kvname-wi"
```

The important pieces are:

```yaml
serviceAccountName: workload-identity-sa
```

and:

```yaml
labels:
  azure.workload.identity/use: "true"
```

and:

```yaml
driver: secrets-store.csi.k8s.io
```

---

# 15. Apply the Kubernetes Resources

The complete resources can be applied together:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  name: ${SERVICE_ACCOUNT_NAME}
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-wi
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "${USER_ASSIGNED_CLIENT_ID}"
    keyvaultName: "${KEYVAULT_NAME}"
    tenantId: "${IDENTITY_TENANT}"
    objects: |
      array:
        - |
          objectName: secret1
          objectType: secret
          objectVersion: ""
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-secrets-store-inline-wi
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: "${SERVICE_ACCOUNT_NAME}"
  containers:
    - name: busybox
      image: registry.k8s.io/e2e-test-images/busybox:1.29-4
      command:
        - "/bin/sleep"
        - "10000"
      volumeMounts:
        - name: secrets-store01-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kvname-wi"
EOF
```

---

# 16. Verify the Pod

Check the Pod:

```bash
kubectl get pod busybox-secrets-store-inline-wi
```

Wait for it to become Ready:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/busybox-secrets-store-inline-wi \
  --timeout=90s
```

Expected:

```text
pod/busybox-secrets-store-inline-wi condition met
```

---

# 17. Verify the Mounted Secret

The CSI driver should mount the Key Vault secret into:

```text
/mnt/secrets-store
```

Check the mounted files:

```bash
kubectl exec \
  busybox-secrets-store-inline-wi \
  -- ls -l /mnt/secrets-store
```

Expected:

```text
secret1
```

Read the secret:

```bash
kubectl exec \
  busybox-secrets-store-inline-wi \
  -- cat /mnt/secrets-store/secret1
```

Expected:

```text
HelloFromKeyVault
```

At this point the complete authentication chain is working:

```text
Pod
 │
 │ ServiceAccount
 ▼
AKS OIDC
 │
 │ Federated Credential
 ▼
User Assigned Managed Identity
 │
 │ Key Vault Secrets User
 ▼
Azure Key Vault
 │
 │ secret1
 ▼
Secrets Store CSI Driver
 │
 ▼
/mnt/secrets-store/secret1
```

---

# Understanding the Authentication Flow

This is the most important concept in the project.

## Step 1 — Pod starts

The Pod uses:

```yaml
serviceAccountName: workload-identity-sa
```

---

## Step 2 — Workload Identity detects the Pod

The Pod contains:

```yaml
azure.workload.identity/use: "true"
```

Azure Workload Identity injects the required environment/configuration into the workload.

---

## Step 3 — Kubernetes provides an OIDC token

The Kubernetes ServiceAccount can obtain a projected service-account token.

The token contains claims identifying:

* Kubernetes cluster issuer
* Namespace
* ServiceAccount
* Audience

The important subject is:

```text
system:serviceaccount:default:workload-identity-sa
```

---

## Step 4 — Microsoft Entra ID validates the federation

Azure checks:

```text
OIDC issuer
       +
Token subject
       +
Federated Identity Credential
```

The federated credential was configured with:

```text
system:serviceaccount:default:workload-identity-sa
```

Therefore Azure knows that this Kubernetes identity is allowed to use the User Assigned Managed Identity.

---

## Step 5 — Managed Identity gets Azure access

The User Assigned Managed Identity has:

```text
Key Vault Secrets User
```

on the Key Vault.

Therefore it can retrieve secrets.

---

## Step 6 — CSI Driver mounts the secret

The Secrets Store CSI Driver retrieves the secret through the Azure Key Vault provider and mounts it into:

```text
/mnt/secrets-store/secret1
```

---

# Why Use CSI Instead of Kubernetes Secrets?

Traditional Kubernetes secrets are stored as Kubernetes objects:

```bash
kubectl get secret
```

The Secrets Store CSI Driver allows workloads to retrieve secrets from an external secrets manager such as:

```text
Azure Key Vault
AWS Secrets Manager
Google Secret Manager
HashiCorp Vault
```

The source of truth remains outside Kubernetes.

This provides a cleaner separation:

```text
Application
     │
     ▼
Kubernetes
     │
     ▼
Secrets Store CSI Driver
     │
     ▼
External Secret Manager
```

---

# Important Security Considerations

## 1. Never commit secrets

Do not commit:

```text
Azure credentials
Client secrets
Private keys
Key Vault secret values
.env files
Kubeconfig files
```

Add sensitive files to `.gitignore`.

Example:

```gitignore
.env
*.key
*.pem
kubeconfig
credentials.json
```

---

## 2. Use least privilege

The workload only needs:

```text
Key Vault Secrets User
```

Do not unnecessarily assign:

```text
Owner
Contributor
Key Vault Administrator
```

to the workload identity.

---

## 3. Restrict the federated subject

The federated identity is tied to:

```text
system:serviceaccount:default:workload-identity-sa
```

This is preferable to creating a broad trust relationship.

---

# Troubleshooting

## Pod is not starting

Check:

```bash
kubectl describe pod busybox-secrets-store-inline-wi
```

Then inspect events:

```bash
kubectl get events \
  --sort-by=.lastTimestamp
```

---

## Check CSI Driver

```bash
kubectl get pods -n kube-system | grep secrets
```

Check logs:

```bash
kubectl logs \
  -n kube-system \
  -l app=secrets-store-csi-driver \
  --tail=100
```

---

## Secret is not mounted

Check the SecretProviderClass:

```bash
kubectl get secretproviderclass
```

Describe it:

```bash
kubectl describe secretproviderclass azure-kvname-wi
```

Check the Pod:

```bash
kubectl describe pod busybox-secrets-store-inline-wi
```

Look for CSI-related events.

---

## Verify Workload Identity

Check the ServiceAccount:

```bash
kubectl describe serviceaccount workload-identity-sa
```

Verify the annotation:

```text
azure.workload.identity/client-id
```

Check the Pod:

```bash
kubectl describe pod busybox-secrets-store-inline-wi
```

Verify:

```text
azure.workload.identity/use=true
```

---

## Check the Federated Credential

```bash
az identity federated-credential list \
  --identity-name "$UAMI" \
  --resource-group "$RESOURCE_GROUP"
```

Verify that the subject matches:

```text
system:serviceaccount:default:workload-identity-sa
```

Also verify the issuer:

```bash
echo "$AKS_OIDC_ISSUER"
```

It must correspond to the AKS OIDC issuer configured in the federated credential.

---

## Check Key Vault RBAC

List role assignments:

```bash
az role assignment list \
  --scope "$KEYVAULT_SCOPE" \
  --assignee "$USER_ASSIGNED_OBJECT_ID" \
  -o table
```

You should see:

```text
Key Vault Secrets User
```

If the role was recently created, allow time for Azure RBAC propagation.

---

## Secret Does Not Exist

Verify:

```bash
az keyvault secret show \
  --vault-name "$KEYVAULT_NAME" \
  --name secret1
```

If necessary:

```bash
az keyvault secret set \
  --vault-name "$KEYVAULT_NAME" \
  --name secret1 \
  --value "HelloFromKeyVault"
```

---

# Useful Verification Commands

### AKS

```bash
az aks show \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME"
```

### Nodes

```bash
kubectl get nodes -o wide
```

### Pods

```bash
kubectl get pods -A
```

### ServiceAccount

```bash
kubectl get serviceaccount "$SERVICE_ACCOUNT_NAME" -o yaml
```

### SecretProviderClass

```bash
kubectl get secretproviderclass -o yaml
```

### Pod

```bash
kubectl get pod busybox-secrets-store-inline-wi -o yaml
```

### Mounted secret

```bash
kubectl exec \
  busybox-secrets-store-inline-wi \
  -- ls -la /mnt/secrets-store
```

---

# Cleanup

Delete the Kubernetes resources:

```bash
kubectl delete pod busybox-secrets-store-inline-wi

kubectl delete secretproviderclass azure-kvname-wi

kubectl delete serviceaccount workload-identity-sa
```

Delete the federated credential:

```bash
az identity federated-credential delete \
  --name "$FEDERATED_IDENTITY_NAME" \
  --identity-name "$UAMI" \
  --resource-group "$RESOURCE_GROUP"
```

Delete the managed identity:

```bash
az identity delete \
  --name "$UAMI" \
  --resource-group "$RESOURCE_GROUP"
```

Delete the Key Vault:

```bash
az keyvault delete \
  --name "$KEYVAULT_NAME" \
  --resource-group "$RESOURCE_GROUP"
```

Delete the resource group:

```bash
az group delete \
  --name "$RESOURCE_GROUP" \
  --yes
```

> **Warning:** Deleting the resource group removes all resources contained within it.

---

# Key Concepts Demonstrated

This project demonstrates the following production-relevant concepts:

### Azure

* Azure Resource Groups
* Azure CLI
* Azure RBAC
* Microsoft Entra ID
* Azure Key Vault
* Managed Identities
* User Assigned Managed Identity
* OIDC
* Federated Identity Credentials
* AKS

### Kubernetes

* Pods
* ServiceAccounts
* CSI Drivers
* Volume mounts
* SecretProviderClass
* Kubernetes identity
* Workload Identity

### Security

* Least privilege
* Passwordless authentication
* Short-lived workload credentials
* External secret management
* Identity federation
* Avoiding static credentials

---

# Interview Explanation

A concise way to explain this project in an interview:

> "I implemented Azure Workload Identity on AKS to allow a Kubernetes workload to securely access secrets stored in Azure Key Vault without storing Azure credentials inside the Pod. I enabled the AKS OIDC issuer and Workload Identity, created a User Assigned Managed Identity, granted it the Key Vault Secrets User role, and created a federated identity credential mapping the Kubernetes ServiceAccount to the Azure identity. The Secrets Store CSI Driver with the Azure Key Vault provider then retrieves the secret and mounts it into the Pod as a filesystem volume."

### Interview flow

```text
Kubernetes ServiceAccount
          │
          ▼
      OIDC Token
          │
          ▼
Federated Identity Credential
          │
          ▼
User Assigned Managed Identity
          │
          ▼
Azure RBAC
          │
          ▼
Azure Key Vault
          │
          ▼
Secrets Store CSI Driver
          │
          ▼
      Pod Volume
```

---

# Common Interview Questions

### Q1. Why use Workload Identity instead of storing Azure credentials in Kubernetes?

Because Workload Identity avoids long-lived static credentials and allows Azure resources to authenticate workloads through federated identity.

---

### Q2. What is the purpose of the OIDC issuer?

The OIDC issuer provides a trusted identity token issuer for Kubernetes ServiceAccounts.

Azure uses this issuer when validating the federated credential.

---

### Q3. What is a Federated Identity Credential?

It establishes a trust relationship between an external identity provider—in this case the AKS OIDC issuer—and an Azure Managed Identity.

---

### Q4. What is the difference between Client ID and Object ID?

**Client ID**

Identifies the managed identity from an application/authentication perspective.

**Object ID / Principal ID**

Identifies the identity's service principal object in Microsoft Entra ID and is commonly used for Azure RBAC operations.

---

### Q5. Why do we need Azure RBAC if Workload Identity is already configured?

Workload Identity answers:

> "Who is this workload?"

Azure RBAC answers:

> "What is this workload allowed to do?"

Both are required.

```text
Workload Identity
       ↓
Authentication

Azure RBAC
       ↓
Authorization
```

---

### Q6. What does Secrets Store CSI Driver do?

It provides the Kubernetes CSI interface that allows secrets stored outside Kubernetes to be mounted into Pods.

In this project:

```text
CSI Driver
    ↓
Azure Key Vault Provider
    ↓
Azure Key Vault
```

---

### Q7. Is the secret stored as a Kubernetes Secret?

In this implementation, the secret is mounted directly as a file through the CSI volume.

The application accesses:

```text
/mnt/secrets-store/secret1
```

rather than retrieving the secret from a normal Kubernetes Secret object.

---

# Production Improvements

This project is intentionally designed as a learning/demo implementation. For production, consider:

* Use Terraform/Bicep instead of manually executing Azure CLI commands.
* Use separate Key Vaults per environment where appropriate.
* Use Azure RBAC with least privilege.
* Restrict Key Vault network access using private endpoints/firewall rules.
* Use separate managed identities for different workloads where appropriate.
* Avoid exposing secret values in logs.
* Use GitHub Actions/Azure DevOps with federated authentication instead of static CI/CD credentials.
* Enable Key Vault diagnostic logging.
* Enable Azure Monitor and audit logging.
* Define Kubernetes resources declaratively.
* Pin container image versions instead of using floating tags.
* Consider secret synchronization into Kubernetes only when an application specifically requires a Kubernetes Secret object.

---

# Project Outcome

After completing this project, the following authentication path is established:

```text
                    ┌───────────────────┐
                    │   Azure Key Vault  │
                    │                   │
                    │ secret1           │
                    └─────────▲─────────┘
                              │
                       Azure RBAC
                              │
                              │
                    ┌─────────┴─────────┐
                    │ User Assigned      │
                    │ Managed Identity   │
                    └─────────▲─────────┘
                              │
                     Federated Credential
                              │
                              │
                    ┌─────────┴─────────┐
                    │ AKS OIDC Issuer   │
                    └─────────▲─────────┘
                              │
                         OIDC Token
                              │
                    ┌─────────┴─────────┐
                    │ Kubernetes Pod    │
                    │                   │
                    │ ServiceAccount    │
                    │ workload-         │
                    │ identity-sa       │
                    └─────────┬─────────┘
                              │
                              │
                    ┌─────────▼─────────┐
                    │ Secrets Store CSI │
                    │ Driver            │
                    └─────────┬─────────┘
                              │
                              ▼
                    /mnt/secrets-store/
                              │
                              ▼
                         secret1
```

The key security principle demonstrated by this project is:

> **Authenticate workloads using federated identity rather than embedding long-lived cloud credentials inside applications.**
