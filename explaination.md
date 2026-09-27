# To enable your pods in Azure Kubernetes Service (AKS) to access secrets stored in Azure Key Vault (AKV), the industry-standard and native approach is using the Azure Key Vault Secrets Store CSI Driver combined with Workload Identity.

**Core Architecture Elements**

The solution comprises five primary elements spanning Azure Identity, Kubernetes resources, and the storage provider:

```
+-----------------------------------------------------------------------------------+
| Kubernetes Pod                                                                   |
|   └── ServiceAccount (annotated with Azure Client ID)                             |
|         │                                                                         |
|         ▼ (Federated Credential Exchange)                                         |
|   [Azure Workload Identity / Microsoft Entra ID]                                  |
|         │                                                                         |
|         ▼ (RBAC: Key Vault Secrets User)                                          |
|   [Azure Key Vault]                                                              |
|         ▲                                                                         |
|         │ (Fetches & Mounts secret on pod startup)                                |
|   [Secrets Store CSI Driver (DaemonSet)] <─── reads [SecretProviderClass (CRD)]   |
+-----------------------------------------------------------------------------------+
```

## 1. Azure Infrastructure Elements

**Azure Key Vault (AKV):**

Houses your sensitive data: Secrets, Keys, or Certificates.

Must be configured with network access permitting AKS (either public with IP rules, or private endpoint inside the AKS VNet).

**Microsoft Entra Workload Identity (User-Assigned Managed Identity):**

Replaces legacy Pod Identity.

An Azure User-Assigned Managed Identity (UAMI) granted the required Azure RBAC role—typically Key Vault Secrets User—scoped to your Key Vault or individual secrets.

**Federated Identity Credential:**

Establishes a trust link in Microsoft Entra between your Managed Identity and the Kubernetes Service Account (using your cluster’s OpenID Connect / OIDC issuer URL).

## 2. Cluster Add-ons & Drivers

**Secrets Store CSI Driver (secrets-store.csi.k8s.io):**

Runs as a DaemonSet on every AKS node.

Intercepts pod volume mount requests and delegates communication to the Azure Key Vault provider.

Native AKS feature that can be enabled via flag: --enable-addons azure-keyvault-secrets-provider.

**Azure Key Vault CSI Provider:**

The cloud-specific plugin that interfaces directly with the Key Vault REST API to fetch objects.

## 3. Kubernetes Native & Custom Resources

**SecretProviderClass (Custom Resource Definition - CRD):**

**Defines the what and where:**

Key Vault name and tenant ID.

Specific secret, key, or cert names and versions to fetch.

Client ID of the Managed Identity used for authentication.

**Kubernetes ServiceAccount:**

Annotated with azure.workload.identity/client-id: <MANAGED_IDENTITY_CLIENT_ID>.

Tied to the pod via spec.serviceAccountName.

**Pod Specification (Deployment / Pod manifest):**

Volume: References the CSI driver (driver: secrets-store.csi.k8s.io) and links to the SecretProviderClass.

VolumeMount: Mounts the secrets directory inside the container (e.g., /mnt/secrets-store).

Optional Component: Secret Auto-Sync
By default, the CSI driver mounts secrets as read-only files on a volume. If your application expects native Kubernetes environment variables instead of file paths:

**secretObjects spec (inside SecretProviderClass):**

Tells the CSI driver to mirror fetched AKV secrets into a native v1/Secret inside the cluster.

Lifecycle Caveat: The native Kubernetes secret is only created after the pod mounts the volume. Once mounted, other pods or environment variables (env.valueFrom.secretKeyRef) can consume it.
