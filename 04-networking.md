# Deploy the hub-spoke network topology

The prerequisites for the [AKS baseline cluster](./) are now completed with [Azure AD group and user work](./03-aad.md) performed in the prior steps. Now we will start with our first Azure resource deployment, the network resources.

## Subscription and resource group topology

This reference implementation is split across several resource groups in a single subscription. This is to replicate the fact that many organizations will split certain responsibilities into specialized subscriptions (e.g. regional hubs/vwan in a _Connectivity_ subscription and workloads in landing zone subscriptions). We expect you to explore this reference implementation within a single subscription, but when you implement this cluster at your organization, you will need to take what you've learned here and apply it to your expected subscription and resource group topology (such as those [offered by the Cloud Adoption Framework](https://learn.microsoft.com/azure/cloud-adoption-framework/decision-guides/subscriptions/).) This single subscription, multiple resource group model is for simplicity of demonstration purposes only.

## Expected results

### Resource groups

The following two resource groups will be created and populated with networking resources in the steps below.

| Name                            | Purpose                                   |
|---------------------------------|-------------------------------------------|
| rg-enterprise-networking-hubs   | Contains all of your organization's regional hubs. A regional hubs include an egress firewall and Log Analytics for network logging. |
| rg-enterprise-networking-spokes | Contains all of your organization's regional spokes and related networking resources. All spokes will peer with their regional hub and subnets will egress through the regional firewall in the hub. |

### Resources

* Regional Azure Firewall in hub virtual network
* Network spoke for the cluster
* Network peering from the spoke to the hub
* Force tunnel UDR for cluster subnets to the hub
* Network Security Groups for all subnets that support them

## Steps

1. Login into the Azure subscription that you'll be deploying into.

   > :book: The networking team logins into the Azure subscription that will contain the regional hub. At Contoso Bicycle, all of their regional hubs are in the same, centrally-managed subscription.

   ```bash
   az login -t $TENANTID_AZURERBAC_AKS_BASELINE
   ```

1. Create the networking hubs resource group.

   > :book: The networking team has all their regional networking hubs in the following resource group. The group's default location does not matter, as it's not tied to the resource locations. (This resource group would have already existed.)

   ```bash
   # [This takes less than one minute to run.]
   az group create -n rg-enterprise-networking-hubs -l Australiaeast
   ```

1. Create the networking spokes resource group.

   > :book: The networking team also keeps all of their spokes in a centrally-managed resource group. As with the hubs resource group, the location of this group does not matter and will not factor into where our network will live. (This resource group would have already existed or would have been part of an Azure landing zone that contains the cluster.)

   ```bash
   # [This takes less than one minute to run.]
   az group create -n rg-enterprise-networking-spokes -l Australiaeast
   ```

1. Create the regional network hub.

   > :book: When the networking team created the regional hub for eastus2, it didn't have any spokes yet defined, yet the networking team always lays out a base hub following a standard pattern (defined in `hub-default.bicep`). A hub always contains an Azure Firewall (with some org-wide policies), Azure Bastion, a gateway subnet for VPN connectivity, and Azure Monitor for network observability. They follow Microsoft's recommended sizing for the subnets.
   >
   > The networking team has decided that `10.200.[0-9].0` will be where all regional hubs are homed on their organization's network space. The `eastus2` hub (created below) will be `10.200.0.0/24`.
   >
   > Note: The subnets for Azure Bastion and on-prem connectivity are deployed in this reference architecture, but the resources are not deployed. Since this reference implementation is expected to be deployed isolated from existing infrastructure; these IP addresses should not conflict with any existing networking you have, even if those IP addresses overlap. If you need to connect the reference implementation to existing networks, you will need to adjust the IP space as per your requirements as to not conflict in the reference ARM templates.

   ```bash
   # [This takes about six minutes to run.]
   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-default.bicep -p location=Australiaeast
   ```

   The hub creation will emit the following:

      * `hubVnetId` - which you'll will query in future steps when creating connected regional spokes. E.g. `/subscriptions/[id]/resourceGroups/rg-enterprise-networking-hubs/providers/Microsoft.Network/virtualNetworks/vnet-eastus2-hub`

1. Create the spoke that will be home to the AKS cluster and its adjacent resources.

   > :book: The networking team receives a request from an app team in business unit (BU) 0001 for a network spoke to house their new AKS-based application (Internally know as Application ID: A0008). The network team talks with the app team to understand their requirements and aligns those needs with Microsoft's best practices for a general-purpose AKS cluster deployment. They capture those specific requirements and deploy the spoke, aligning to those specs, and connecting it to the matching regional hub.

   ```bash
   RESOURCEID_VNET_HUB=$(az deployment group show -g rg-enterprise-networking-hubs -n hub-default --query properties.outputs.hubVnetId.value -o tsv)
   echo RESOURCEID_VNET_HUB: $RESOURCEID_VNET_HUB

   # [This takes about four minutes to run.]
   az deployment group create -g rg-enterprise-networking-spokes -f networking/spoke-BU0001A0008.bicep -p location=australiaeast hubVnetResourceId="${RESOURCEID_VNET_HUB}"
   ```

   The spoke creation will emit the following:

     * `appGwPublicIpAddress` - The Public IP address of the Azure Application Gateway (WAF) that will receive traffic for your workload.
     * `clusterVnetResourceId` - The resource ID of the Virtual network where the cluster, App Gateway, and related resources will be deployed. E.g. `/subscriptions/[id]/resourceGroups/rg-enterprise-networking-spokes/providers/Microsoft.Network/virtualNetworks/vnet-spoke-BU0001A0008-00`
     * `nodepoolSubnetResourceIds` - An array containing the subnet resource IDs of the AKS node pools in the spoke. E.g. `[ "/subscriptions/[id]/resourceGroups/rg-enterprise-networking-spokes/providers/Microsoft.Network/virtualNetworks/vnet-hub-spoke-BU0001A0008-00/subnets/snet-clusternodes" ]`

1. Update the shared, regional hub deployment to account for the requirements of the spoke.

   > :book: Now that their regional hub has its first spoke, the hub can no longer run off of the generic hub template. The networking team creates a named hub template (e.g. `hub-eastus2.bicep`) to forever represent this specific hub and the features this specific hub needs in order to support its spokes' requirements. As new spokes are attached and new requirements arise for the regional hub, they will be added to this template file.

   ```bash
   RESOURCEID_SUBNET_NODEPOOLS=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0008 --query properties.outputs.nodepoolSubnetResourceIds.value -o json)
   echo RESOURCEID_SUBNET_NODEPOOLS: $RESOURCEID_SUBNET_NODEPOOLS

   # [This takes about ten minutes to run.]
   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-regionA.bicep -p location=eastus2 nodepoolSubnetResourceIds="${RESOURCEID_SUBNET_NODEPOOLS}"
   ```

   > :book: At this point the networking team has delivered a spoke in which BU 0001's app team can lay down their AKS cluster (ID: A0008). The networking team provides the necessary information to the app team for them to reference in their infrastructure-as-code artifacts.
   >
   > Hubs and spokes are controlled by the networking team's GitHub Actions workflows. This automation is not included in this reference implementation as this body of work is focused on the AKS baseline and not the networking team's CI/CD practices.

## Private DNS Zones

Private DNS zones in this reference implementation are implemented directly at the spoke level, meaning the workload team creates the private link DNS zones & records for the resources needed; furthermore, the workload is directly using Azure DNS for resolution. Your networking topology might support this decentralized model or instead DNS & DNS zones for Private Link might be handed at the regional hub or in a [VWAN virtual hub extension](https://learn.microsoft.com/azure/architecture/guide/networking/private-link-virtual-wan-dns-virtual-hub-extension-pattern) by your networking team.

If your organization operate a centeralized DNS model, you'll need to adapt how DNS zones records are managed this implementation into your existing enteprise networking DNS zone strategy. Since this reference implementation is expected to be deployed isolated from existing infrastructure; this is not something you need to address now; but will be something to understand and address when taking your solution to production.

### Next step

:arrow_forward: [Prep for cluster bootstrapping](./05-bootstrap-prep.md)
