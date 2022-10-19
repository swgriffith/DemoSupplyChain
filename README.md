# Demo Supply Chain 

## TODO:

* Lock for PR only
* Create infra setup steps
* Setup github template


## Infrastructure Setup

To create the cluster we'll use basic [Azure CLI]() commands. This will help better understand all the specific components involved. This cluster will also be locked down for authentication, but we won't add the complexity of egress lockdown at this time.

> **NOTE:**
> You will need the Azure Active Directory Object ID for the AAD Group which will serve as your administor group. If you dont have this, you can create the cluster without AAD integrated authentication enabled.

### Resource Group Creation

```bash
# Set some environment variables
RG=DemoSupplyChain
LOC=eastus
CLUSTER_NAME=aquademo

# Create the resource group
az group create -n $RG -l $LOC
```

### Network setup

```bash
VNET_NAME=demovnet

# Create the Virtual Network and cluster subnet
az network vnet create \
-g $RG \
-n $VNET_NAME \
--address-prefix 10.40.0.0/16 \
--subnet-name aks --subnet-prefix 10.40.0.0/24

# Get the Vnet and Subnet IDs for later
AKS_SUBNET_ID=$(az network vnet subnet show -g $RG --vnet-name demovnet -n aks -o tsv --query id)

# Create the ACR Subnet
az network vnet subnet create \
--name acr \
--vnet-name $VNET_NAME \
--resource-group $RG \
--address-prefixes 10.40.1.0/24 \
--disable-private-endpoint-network-policies
```

### Create the Azure Container Registry

The following steps will create a private Azure Container Registry. All steps are shown below to help you understand the component parts, but if you prefer the easy route, setting this up in the Azure portal is very quick and easy.

> **NOTE:**
> For the private networking features we're enabling, premium level is required. You can skip the public-network-enabled flag if you wish to use Standard tier.

```bash
# Set the ACR Name
ACR_NAME=demosupplychainacr

# Create the ACR
az acr create \
-n $ACR_NAME \
-g $RG \
--sku Premium \
--admin-enabled false \
--public-network-enabled false

# Create the private zone
az network private-dns zone create \
--resource-group $RG \
--name "privatelink.azurecr.io"

# Link the private zone to the Virtual Network
az network private-dns link vnet create \
--resource-group $RG \
--zone-name "privatelink.azurecr.io" \
--name ACRZone \
--virtual-network $VNET_NAME \
--registration-enabled false

# Get the ACR resource ID
REGISTRY_ID=$(az acr show --name $ACR_NAME --query 'id' --output tsv)

# Create the private endpoint
az network private-endpoint create \
--name myPrivateEndpoint \
--resource-group $RG \
--vnet-name $VNET_NAME \
--subnet acr \
--private-connection-resource-id $REGISTRY_ID \
--group-ids registry \
--connection-name acrprivateendpoint

# Get the private endpoint IPs and FQDNs to use to add the A records to the private zone
NETWORK_INTERFACE_ID=$(az network private-endpoint show \
--name myPrivateEndpoint \
--resource-group $RG \
--query 'networkInterfaces[0].id' \
--output tsv)

REGISTRY_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateIpAddress" \
  --output tsv)

DATA_ENDPOINT_PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_$LOC'].privateIpAddress" \
  --output tsv)

# An FQDN is associated with each IP address in the IP configurations
# REGISTRY_FQDN=$(az network nic show \
#   --ids $NETWORK_INTERFACE_ID \
#   --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry'].privateLinkConnectionProperties.fqdns" \
#   --output tsv)

# DATA_ENDPOINT_FQDN=$(az network nic show \
#   --ids $NETWORK_INTERFACE_ID \
#   --query "ipConfigurations[?privateLinkConnectionProperties.requiredMemberName=='registry_data_$LOC'].privateLinkConnectionProperties.fqdns" \
#   --output tsv)

# Create the A records
az network private-dns record-set a create \
--name $REGISTRY_NAME \
--zone-name privatelink.azurecr.io \
--resource-group $RESOURCE_GROUP

# Specify registry region in data endpoint name
az network private-dns record-set a create \
--name $ACR_NAME \
--zone-name privatelink.azurecr.io \
--resource-group $RG

az network private-dns record-set a create \
--name ${ACR_NAME}.${LOC}.data \
--zone-name privatelink.azurecr.io \
--resource-group $RG

az network private-dns record-set a add-record \
--record-set-name $ACR_NAME \
--zone-name privatelink.azurecr.io \
--resource-group $RG \
--ipv4-address $REGISTRY_PRIVATE_IP

# Specify registry region in data endpoint name
az network private-dns record-set a add-record \
--record-set-name ${ACR_NAME}.${LOC}.data \
--zone-name privatelink.azurecr.io \
--resource-group $RG \
--ipv4-address $DATA_ENDPOINT_PRIVATE_IP
```

### Create the AKS Cluster

In this cluster creation, we'll set the AKS cluster up with the following configuration:

* Public Kubernetes API Server Accesse
* Enforce allowed ips on the API server
* Enable Azure AD integrated authentication
* Provide an administrator Azure AD Group ID 
* Disable the default cluster local admin account
* Attach the ACR

```bash
CLUSTER_NAME=democluster
MY_IP=$(curl -4 icanhazip.com)
ADMIN_GROUP_ID=

# Create the AKS Cluster
az aks create \
-g $RG \
-n $CLUSTER_NAME \
--enable-aad \
--aad-admin-group-object-ids $ADMIN_GROUP_ID \
--disable-local-accounts \
--attach-acr $ACR_NAME \
--vnet-subnet-id $AKS_SUBNET_ID \
--api-server-authorized-ip-ranges "${MY_IP}"

az aks get-credentials -g $RG -n $CLUSTER_NAME

PAT=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
```