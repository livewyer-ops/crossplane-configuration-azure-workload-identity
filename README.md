# Crossplane Configuration for Azure Workload Identity

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A Crossplane Configuration package that simplifies the creation and management of Azure Workload Identity resources, enabling secure authentication between Kubernetes workloads and Azure services using OIDC federation.

## Overview

This configuration automates the setup of Azure Workload Identity by creating and managing:

- **Azure User Assigned Managed Identity** - The identity that will be used by your Kubernetes workloads
- **Federated Identity Credentials** - OIDC federation configuration linking the managed identity to Kubernetes service accounts
- **Azure RBAC Role Assignments** - Permissions granted to the managed identity
- **Custom Role Definitions** - Support for creating custom Azure roles when needed
- **Kubernetes Service Accounts** - Optionally creates service accounts with proper annotations

## Prerequisites

Before using this configuration, ensure you have:

1. **Kubernetes Cluster** (v1.24+) with:
   - Crossplane v2.0.2 or higher installed
   - OIDC issuer enabled (for AKS, EKS, or self-managed clusters)

2. **Azure Subscription** with:
   - Appropriate permissions to create managed identities and role assignments
   - Resource Group where resources will be created

3. **Required Crossplane Providers**:
   - `provider-family-azure` v2+
   - `provider-kubernetes` v1+

4. **Required Crossplane Functions**:
   - `function-go-templating` v0.10.0+
   - `function-auto-ready` v0.5.0+

## Installation

### Step 1: Install Crossplane

If you haven't already installed Crossplane:

```bash
kubectl create namespace crossplane-system
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane \
  --namespace crossplane-system \
  crossplane-stable/crossplane \
  --version 2.0.2
```

### Step 2: Install Required Providers and Functions

```bash
# Install providers
kubectl apply -f examples/providers.yaml

# Install functions
kubectl apply -f examples/functions.yaml

# Wait for providers to become healthy
kubectl wait --for=condition=Healthy \
  --timeout=300s \
  provider.pkg.crossplane.io --all
```

### Step 3: Configure Azure Provider Credentials

Create a secret with your Azure service principal credentials:

```bash
# Create a JSON file with your Azure credentials
cat > azure-credentials.json <<EOF
{
  "clientId": "YOUR_CLIENT_ID",
  "clientSecret": "YOUR_CLIENT_SECRET",
  "subscriptionId": "YOUR_SUBSCRIPTION_ID",
  "tenantId": "YOUR_TENANT_ID"
}
EOF

# Create the secret
kubectl create secret generic provider-azure \
  --from-file=credential=./azure-credentials.json \
  --namespace crossplane-system

# Clean up the credentials file
rm azure-credentials.json
```

### Step 4: Install the Configuration

Build and push the configuration package:

```bash
# Build the package
crossplane xpkg build

# Login to your registry (if using Upbound)
crossplane xpkg login

# Push the package
crossplane xpkg push -f *.xpkg livewyer-ops/crossplane-configuration-azure-workload-identity:v1.0.0
```

Or install directly from the registry:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: configuration-azure-workload-identity
spec:
  package: xpkg.upbound.io/livewyer-ops/crossplane-configuration-azure-workload-identity:v1.0.0
EOF
```

## Usage

### Basic Example

Create a WorkloadIdentity resource with basic role assignments:

```yaml
apiVersion: azure.livewyer.io/v1alpha1
kind: WorkloadIdentity
metadata:
  name: my-app-identity
  namespace: default
spec:
  forProvider:
    location: eastus
    resourceGroupName: my-resource-group
    serviceAccountName: my-app-sa
    oidcIssuerURL: https://eastus.oic.prod-aks.azure.com/00000000-0000-0000-0000-000000000000/11111111-1111-1111-1111-111111111111/
    roleAssignments:
      - roleDefinitionName: Reader
        scope: /subscriptions/YOUR_SUBSCRIPTION_ID
    tags:
      environment: production
      app: my-app
```

### Advanced Example with Custom Roles

```yaml
apiVersion: azure.livewyer.io/v1alpha1
kind: WorkloadIdentity
metadata:
  name: storage-operator
  namespace: storage-system
spec:
  forProvider:
    location: westeurope
    resourceGroupName: storage-rg
    serviceAccountName: storage-operator-sa
    serviceAccountTokenExpiration: 3600 # 1 hour
    oidcIssuerURL: https://oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE
    roleAssignments:
      # Built-in role
      - roleDefinitionName: "Storage Account Contributor"
        scope: /subscriptions/SUB_ID/resourceGroups/storage-rg
        description: "Manage storage accounts in storage-rg"

      # Custom role with specific permissions
      - roleDefinitionName: "CustomBlobReader"
        scope: /subscriptions/SUB_ID
        permissions:
          - actions:
              - "Microsoft.Storage/storageAccounts/listKeys/action"
              - "Microsoft.Storage/storageAccounts/read"
            dataActions:
              - "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read"
              - "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/write"
            notActions:
              - "Microsoft.Storage/storageAccounts/delete"
    tags:
      team: platform
      cost-center: engineering
```

### Using the Created Service Account in Your Pods

Once the WorkloadIdentity is created, you can use the service account in your pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
spec:
  serviceAccountName: my-app-sa # The service account created by WorkloadIdentity
  containers:
    - name: app
      image: myapp:latest
      env:
        - name: AZURE_CLIENT_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['azure.workload.identity/client-id']
```

## Configuration Reference

### WorkloadIdentity Spec

| Field                                       | Type    | Required | Description                                  |
| ------------------------------------------- | ------- | -------- | -------------------------------------------- |
| `forProvider.location`                      | string  | Yes      | Azure region (e.g., eastus, westeurope)      |
| `forProvider.resourceGroupName`             | string  | Yes      | Name of the Azure Resource Group             |
| `forProvider.oidcIssuerURL`                 | string  | Yes      | OIDC issuer URL of your Kubernetes cluster   |
| `forProvider.roleAssignments`               | array   | Yes      | List of role assignments to grant            |
| `forProvider.serviceAccountName`            | string  | No       | Kubernetes service account name to create    |
| `forProvider.serviceAccountTokenExpiration` | integer | No       | Token expiration in seconds (default: 86400) |
| `forProvider.tags`                          | object  | No       | Azure resource tags                          |
| `providerConfigRef`                         | object  | No       | Reference to provider configuration          |

### Role Assignment Options

| Field                          | Type   | Description                                   |
| ------------------------------ | ------ | --------------------------------------------- |
| `roleDefinitionName`           | string | Name of built-in or custom role               |
| `scope`                        | string | Azure resource scope for the assignment       |
| `permissions`                  | array  | Permissions for custom role (optional)        |
| `permissions[].actions`        | array  | Allowed management operations                 |
| `permissions[].dataActions`    | array  | Allowed data operations                       |
| `permissions[].notActions`     | array  | Denied management operations                  |
| `permissions[].notDataActions` | array  | Denied data operations                        |
| `condition`                    | string | Condition expression for conditional access   |
| `conditionVersion`             | string | Version of condition syntax                   |
| `principalType`                | string | Type of principal (default: ServicePrincipal) |

## Testing

### Verify Installation

1. Check that the configuration is installed:

```bash
kubectl get configurations
kubectl get xrds | grep workloadidentity
kubectl get compositions | grep workloadidentity
```

2. Create a test WorkloadIdentity:

```bash
kubectl apply -f examples/workload-identity.yaml
```

3. Verify resources are created:

```bash
# Check the composite resource
kubectl get workloadidentity -n test

# Check the managed resources
kubectl get managed -n test

# Check Azure resources
kubectl get userassignedidentity -n test
kubectl get federatedidentitycredential -n test
kubectl get roleassignment -n test
```

### Validate Azure Resources

Using Azure CLI:

```bash
# List managed identities
az identity list --resource-group example-group

# Check role assignments
az role assignment list --assignee <PRINCIPAL_ID>

# Verify federated credentials
az identity federated-credential list \
  --resource-group example-group \
  --identity-name simple-example
```

### Test Workload Authentication

Deploy a test pod to verify authentication:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-workload-identity
  namespace: test
spec:
  serviceAccountName: simple-example
  containers:
    - name: azure-cli
      image: mcr.microsoft.com/azure-cli:latest
      command: ["/bin/bash", "-c", "--"]
      args:
        - |
          # Login using workload identity
          az login --federated-token $(cat $AZURE_FEDERATED_TOKEN_FILE) \
            --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID

          # Test access
          az account show
          az group list
```

## Observability

### Check Resource Status

```bash
# Get detailed status
kubectl describe workloadidentity -n test simple-example

# Check conditions
kubectl get workloadidentity -n test simple-example -o jsonpath='{.status.conditions}'
```

### View Logs

```bash
# Crossplane pod logs
kubectl logs -n crossplane-system deployment/crossplane -f

# Provider logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=provider-azure-managedidentity
```

## Troubleshooting

### Common Issues

1. **OIDC Issuer URL Incorrect**
   - Verify the OIDC issuer URL matches your cluster configuration
   - For AKS: `az aks show --name <cluster> --resource-group <rg> --query "oidcIssuerProfile.issuerUrl"`
   - For EKS: `aws eks describe-cluster --name <cluster> --query "cluster.identity.oidc.issuer"`

2. **Role Assignment Failures**
   - Ensure the service principal has permissions to create role assignments
   - Check that the scope exists and is correctly formatted
   - Verify custom role permissions are valid

3. **Service Account Not Created**
   - Check that the Kubernetes provider is healthy
   - Verify provider configuration references are correct
   - Check namespace exists

4. **Resources Stuck in Creating State**
   - Review provider pod logs for errors
   - Check Azure API quotas and limits
   - Verify network connectivity to Azure

### Debug Commands

```bash
# Get all events
kubectl get events -n test --sort-by='.lastTimestamp'

# Check composition pipeline
kubectl get composite -n test simple-example -o yaml | less

# View rendered resources
kubectl get managed -n test -o yaml | grep -A 5 "forProvider"
```

## Cleanup

To remove a WorkloadIdentity and all its resources:

```bash
kubectl delete workloadidentity -n test simple-example
```

To uninstall the configuration:

```bash
kubectl delete configuration configuration-azure-workload-identity
```

## Contributing

We welcome contributions! Please see our [Contributing Guide](CONTRIBUTING.md) for details.

### Development

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Update examples and documentation
5. Test your changes
6. Submit a pull request

### Testing Locally

```bash
# Build the package locally
crossplane xpkg build

# Install for testing
kubectl apply -f apis/definition.yaml
kubectl apply -f apis/composition.yaml

# Test with examples
kubectl apply -f examples/workload-identity.yaml
```

## Support

- **Issues**: [GitHub Issues](https://github.com/livewyer-ops/crossplane-configuration-azure-workload-identity/issues)
- **Discussions**: [GitHub Discussions](https://github.com/livewyer-ops/crossplane-configuration-azure-workload-identity/discussions)
- **Documentation**: [Crossplane Docs](https://docs.crossplane.io)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Maintainers

- Livewyer Team <bowen@livewyer.com>

## Acknowledgments

- [Crossplane](https://crossplane.io) for the amazing framework
- [Upbound](https://upbound.io) for provider implementations
- Azure Workload Identity documentation and examples
