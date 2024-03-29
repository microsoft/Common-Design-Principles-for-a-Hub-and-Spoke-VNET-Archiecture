
***
**DISCLAIMER**: This article has been deprecated and is no longer being updated. Please refer to the following article for the most up-to-date information: [ExpressRoute MSEE hairpin design considerations](https://techcommunity.microsoft.com/t5/azure-networking-blog/expressroute-msee-hairpin-design-considerations/ba-p/4101161)
***

# What is a VNet
 
An Azure Virtual Network (VNet) is like a virtual routing and forwarding (VRF) instance in a traditional network.  It provides a logical network building block to create your network infrastructure on Azure.  It is virtual network isolation on the Azure cloud dedicated to your subscription.  You use a VNet to build or create multiple subnets to support your CIDR design. IP Subnet (Think of a VLAN):  Provides full layer-3 semantics and partial layer-2 semantics (DHCP, ARP, no broadcast/multicast).  You can optionally peer or connect VNets with other VNets as long as address ranges do not overlap.  

![alt text](https://github.com/jgmitter/images/blob/master/vnet.jpg)


Use VNets to:
*   Link your on-premise IT infrastructure to create a hybrid connection to Azure via VPN (S2S and P2S) or ExpressRoute Private-Peering with a virtual network gateway deployed within a VNet. https://aka.ms/AA5ynvj and https://aka.ms/AA5yt4p
*   Create a dedicated private cloud-only VNet. Sometimes you don't require a hybrid connection to on-premise for your solution. When you create a VNet, your services and VMs within your VNet can communicate directly and securely with each other in the cloud. 
*   Create a dedicated DMZ for secure Internet access including Azure native security services such as Azure Firewall and Azure WAF or 3rd party NVAs.  https://aka.ms/AA5ygg9
*   Create a Hub and Spoke model. The Hub acts as a transitive routing domain, and the Spokes nest your applications dedicated to a business group’s subscription.  https://aka.ms/AA5yt60
*   Control routing behavior and creates layers of segmentation.   When you put a virtual machine on an Azure virtual network, the VM can connect to any other VM on the same virtual network, even if the other VMs are on different subnets. It is possible because a collection of system routes enabled by default allows this type of communication.  Although the default system routes are useful for many deployment scenarios, there are times when you want to customize the routing configuration for your deployments. You can configure the next-hop address to reach specific destinations.  Also, you can create additional layers of segmentation using Network security groups (NSGs).  NSGs are simple, stateful packet inspection devices that use the 5-tuple approach (source IP, source port, destination IP, destination port, and layer four protocol) to create allow/deny rules for network traffic.  But in some situations, you want or need to enable security at high levels of the stack. In such situations, we recommend that you deploy virtual network security appliances provided by Azure or by Azure partners.
*   Privatize the data plane of supported PAAS services such as SQL MI, ASE, APIM, etc. Examples are instead of the transfer of data over public networks transfer over a private network such as your VNet including hybrid connections such as ExpressRoute).  Examples of VNet Integration: (ASE: https://aka.ms/AA5ygph, App Service: https://aka.ms/AA5ygpk, and Private Link: https://aka.ms/AA62xe2
*   Create shared services that can be shared by multiple workloads, such as network virtual appliances (NVAs) and DNS servers, in a single location.  

# Hub and Spoke Architecture

There are plenty of reasons for connecting a single VNet to your ExpressRoute circuit. After all, a VNet can support up to 65,513 IP addresses, thousands of subnets, and thousands of network blocks.  For many, these limits are enough to create an entire virtual data center within a single VNet!  The single VNet model is ideal for customers who have a single subscription for their Azure footprint and prefer to use subnets as a way of creating segmentation for networking security and workload assignment.  
However, for others, and single VNet is not enough to address the complexity of their business model in the Cloud.  This complexity often stems from their need to support multiple subscriptions using a centralized IT model which will offer shared services across different tenants or application stacks.  The ability to assign each tenant to their own subscription allows for a high degree of granularity and segmentation for billing, role-based access, and network security. 

The most flexible, effective way of realizing this topology within Azure is by way of the Hub-and-Spoke model, where a Hub VNet provides shared network, security, and identity services to a variety of Spoke VNets that are mapped to different apps or tenants.  Spoke VNets, by default, only see the Hub VNet through their peering connection.  This “sequestered spoke” behavior sets up a strong DMZ foundation in the Hub to allow greater control between both Spoke-to-Spoke traffic and Spoke-to-on-prem traffic.  For a deeper dive into the benefits of Hub-and-Spoke, please read https://aka.ms/AA62lke and https://aka.ms/AA633dq.
The most flexible, effective way of realizing this topology within Azure is by way of the Hub-and-Spoke model, where a Hub VNet provides shared network, security, and identity services to a variety of spoke VNet that are mapped to different apps or tenants.  The components of this model include:  

*   Hub VNet:   An Azure VNet used as the Hub in the Hub-Spoke topology. The Hub is the central point of connectivity to your on-premises network, and a place to host shared services that can be consumed by the different workloads hosted in the Spoke VNets.  The Hub VNet is typically mapped directly to an IT subscription to provide network transit (ExpressRoute Gateway, VPN Gateways, NVAs), security, and identity services. Any virtual machines and workloads deployed in the Hub VNet will be invisible to tenant subscriptions that are mapped to Spoke VNets.  
*   Spoke VNets: One or more Azure VNets that are used as Spokes in the Hub-Spoke topology.   Spoke VNets are focused on tenant workloads. These Spoke VNets could be a representation of different application states, application stacks, departments, or lines of business. It is common for IT and security professionals to be granted role-based access (RBAC) to these Spokes to maintain infrastructure components such as VNets, Network Security Groups, Route Tables, and so forth.   Spokes can be used to isolate workloads in their VNets, managed separately from other Spokes. Each workload might include multiple tiers, with multiple subnets connected through Azure load balancers.   Due to the nature of VNets being non-transitive by default Spoke VNets are isolated from each other.
*   VNet peering: VNet peering is a non-transitive low latency connection between two VNets.  Once peered, the VNets exchange traffic by using the Azure backbone, without the need for a router. In a Hub-Spoke network topology, you use VNet peering to connect the Hub to each Spoke. You can peer virtual networks in the same region or different regions. You can also configure Spokes to use the Hub VNet’s ExpressRoute or Azure VPN gateway to communicate with on-prem networks.  When an ExpressRoute circuit is connected into the Hub-and-Spoke design, the BGP routes that are advertised from on-prem into the ExpressRoute virtual gateway will not natively transit into the Spoke VNets. Instead, these routes will populate in the Hub VNet only. Likewise, the ExpressRoute gateway will not natively advertise Spoke VNet routes out, and instead, will only advertise the network ranges that belong to the Hub VNet.  In order to allow the ExpressRoute virtual gateway to advertise routes from Hub to Spoke, you must select the “Allow Gateway Transit” option in the VNet peering panel of the Hub.  And then to allow the Spoke VNet routes to transit out of the ExpressRoute gateway, you must select the “Use Remote Gateway” option in the VNet peering panel of the Spoke.  Once complete, all on-prem BGP routes advertised through ExpressRoute will be visible in the Hub and all participating Spokes, and vise-versa. 
*   Resource Groups: The Hub VNet, and each Spoke VNet, can be implemented in different resource groups, and different subscriptions. When you peer virtual networks in different subscriptions, both subscriptions can be associated with the same or different Azure Active Directory tenant. RGs allows for decentralized management of each workload while sharing services maintained in the Hub VNet.
*   GatewaySubnet:  A Gateway subnet is required to build a virtual network gateway. Allocating 32 addresses to this subnet will help to prevent reaching gateway size limitations in the future.   For more information about setting up the gateway, see the following reference architectures, https://aka.ms/AA5z0l9.  For higher availability, you can use ExpressRoute plus a VPN for failover, https://aka.ms/AA5ytjc.   A Hub-Spoke topology can also be used without a gateway if you don't need connectivity with your on-premises network.
*   Network Security Groups:  Network security groups protect your workloads with distributed ACLs by providing stateful layer 3 and 4 segmentation.   NSGs can be enforced at the host and subnet layer.  NSGs can use service tags which are named monikers for Azure service IPs and removes the need for management of IP addresses.  NSGs also support flow logs for traffic monitoring and troubleshooting.  Optionally, NSGs can use Application Security Groups (ASG) which are named monikers for groups of VMs and allows the removal of management of NSGs via IP addresses.  As of this writing ASGS are only supported within a VNet.  
*   User Defined Routes:  You can create custom routes in your VNets to override Azure's default system routes, or to add additional routes to a subnet's route table. In Azure, you create a route table, then associate the route table to one or more virtual network subnets.  UDRs allow you to route traffic to virtual appliances (NVAs), VPN Gateways, Internet, etc.  https://aka.ms/AA5z1t3.  It's also important to understand how Aure selects a route: https://aka.ms/AA62lkm.
*   Please always reference Azure VNet Subscription Limits: https://aka.ms/AA62lkr

# The diagram below details a transitive Hub VNet and Spoke VNet topology.

![alt text](https://github.com/jgmitter/images/blob/master/hub%20and%20spoke.jpg)

# Intra-Region VNET Routing Options

In environments where spoke to spoke communication is required, there are three different options for allowing this connectivity. Each option has advantages and disadvantages, which will be laid out here for intra-region communications, and these options can be co-mingled for various workloads.  

# Option 1: Leveraging ExpressRoute

If ExpressRoute is in use to allow connectivity from on-prem locations, we can leverage the ExpressRoute circuit to provide native spoke to spoke communication. Either a default route (0.0.0.0/0) or a summary route comprising all the networks for the region VNETs can be injected via BGP across ExpressRoute. The figure below shows this configuration.

![alt text](https://github.com/jgmitter/images/blob/master/jg1.png)
 
By the VNET peerings established in the figure above, the Microsoft Edge routers (MSEEs) will receive the summary routes of the HUB and Spoke VNETs in the West region. Specifically, the MSEEs will know about the routes 10.0.0.0/16 (from West 2 HUB), 10.1.0.0/16 (from West 2 Spoke 1), and 10.4.0.0/16 (from West 2 Spoke 2). At this point, however, the MSEEs are the only capable routing device that knows about these individual routes. The spokes do not have routes to each other, and although the HUB is aware of both spokes, it is not transitive by default and therefore cannot route traffic between them. With each spoke having (in most cases) a default route the internet, when spoke one tries to talk to spoke 2, this traffic will try and route through the internet using its default route and then fail.

If we advertise a default route (0.0.0.0/0) or a summary route of all the VNETs in the region (10.0.0.0/8), we can change this behavior and allow spoke to spoke communication. As the default or summary route gets advertised from on-prem, the hub will receive this route and propagate this route down into the spokes. As a result, each spoke now has a new route instead of the default to the internet. For instance, if we were to advertise a summary route of 10.0.0.0/8 from on-prem, both spokes would receive this route. When spoke one tried to talk to spoke 2, the new summary route would be the closest match and traffic would be sent towards ExpressRoute. Since the MSEEs are aware of the individual routes, we are now able to route back up ExpressRoute to the other spoke. These default or summary routes merely allow us a mechanism to pull spoke to spoke traffic down to the MSEEs which is the only device which knows the specific spoke’s address space.

The advantage to this approach is that by advertising a summary or default route, we provide the ability for spoke to spoke routing natively within the Azure backbone via the MSEEs. Also, this traffic is specifically identified within Azure and does not trigger any VNET peering costs, so it is completely free to the customer. The downside to this approach is that traffic is limited to the bandwidth of the ExpressRoute gateway SKU size and latency of hair-pinning off of the MSEEs which are in the peering location of your ExpressRoute circuit. For example, if I am using the Standard Express Route Gateway SKU, I have a bandwidth limit of 1G. This throughput limit would also apply to spoke to spoke communications using this method.  If the latency is X ms, it is doubled due to hair-pinning at the peering location.  

 ![alt text](https://github.com/jgmitter/images/blob/master/jg2.png)
 
# Option 2: Leveraging a HUB NVA

A secondary option for allowing spoke to spoke communication would be to leverage an NVA in a HUB VNET to route traffic between the spokes. In this scenario, we deploy an NVA of our choosing into the HUB and define static UDRs within each spoke to route to the NVA to get to the other spoke.
 
![alt text](https://github.com/jgmitter/images/blob/master/jg3.png) 

This approach has some advantages over Option 1. However, it also comes with its own set of disadvantages. As opposed to Option 1 in which we are routing all the way down to the MSEEs, we are manually doing this at the HUB level. As a result, we no longer need to worry about advertising in a default or summary route which may relieve some administrative overhead but might also result in some lower latency as we are communicating through a HUB rather than through an MSEE. In all reality, the improvement here is fairly low as the Azure backbone doesn’t add too much latency. However, this may be beneficial for workloads looking to shave some additional milliseconds.

Additionally, we do not have the bandwidth limitation of the ExpressRoute gateway as this is no longer in the path. One final advantage of leveraging an NVA for this functionality would be the granular control and inspection capabilities this NVA comes with. Spoke to spoke traffic can now be fully inspected and subject to the NVAs granular policy set using this method.

The disadvantages of this approach come first with the cost of deploying an NVA. Regardless of NVA chosen, there will be an additional cost for running the NVA and, typically, a throughput cost for the traffic traversing the NVA. While we lose the bandwidth limitation of the ExpressRoute gateway in this scenario, NVAs have bandwidth limitations of their own, which could be a bottleneck for this traffic. Understanding the specific throughput limitations of the NVA chosen would be critical when leveraging this method. Finally, when leveraging Option 1, the Azure fabric recognizes this data path and does not apply charges for the traffic when it traverses the VNET peerings. If using an NVA, all traffic which traverses VNET peerings to reach the NVA will incur VNET peering costs.

# Option 3: VNET Peering

The 3rd option for allowing spoke to spoke communication would be to directly peer the spokes requiring communication. Similar to how each spoke is VNET peered to the HUB, and additional VNET peer would be created between the two spokes which require communication.
 
![alt text](https://github.com/jgmitter/images/blob/master/jg4.png)

The advantage of this option is spokes are now directly connected via the Azure backbone and have the lowest latency path possible. Additionally, no bandwidth restrictions exist along the path, so hosts are only limited by the amount of data they can push.

The disadvantage of this approach is the additional cost associated with VNET peering as well as the scale limitations should spokes continue to grow. As multiple spokes are introduced behind a hub and require connectivity, a full mesh of VNET peers would be required to provide this connectivity.

As the Hub and Spoke model is replicated across regions, spoke to spoke communication options change a little bit. In environments where spoke to spoke communication is required between regions, there are three different options for allowing this connectivity. Each option has advantages and disadvantages, which will be discussed below for inter-region communications, and these options can be co-mingled for various workloads.

# Inter-Region VNET Routing Options

# Option 1: Leveraging Bow-Tied ExpressRoute Connections

A best practice for providing redundancy and connectivity between regions and on-premise is to bow-tie connect ExpressRoute Gateways with adjacent ExpressRoute circuits in different regions, as shown in  below. Not only does this provide connectivity to on-premise, but you also establish connectivity between the ExpressRoute circuits to the cross-regional VNets with no transit charges. The bow-tie cross-regional connectivity between VNets to the ExpressRoute circuits would enable communications between the US West 2 VNET Hub and Spokes to communicate with the US East 2 VNet Hub and Spokes. 

This option allows inter-regional Azure communication for the private deployment to ride over the Microsoft backbone. The Hub VNET attached to an ExpressRoute gateway will advertise its VNET address space as well as connected VNET Spoke address space(s) (by selecting ‘Use Remote Gateway’) towards the ExpressRoute Circuit. Since both regions are bow-tied, each Hub VNET will learn about their neighboring region VNETS and provide native communication between any inter-region Hub to Hub, Hub to Spoke or Spoke to Spoke communication.

![alt text](https://github.com/jgmitter/images/blob/master/jg5.png)

The VNET peerings established in the figure above, the Microsoft Edge routers (MSEEs) will receive the summary routes of the HUB and Spoke VNETs in the US West 2 region. Specifically, the MSEEs will know about the routes 10.0.0.0/16 (from West 2 HUB), 10.1.0.0/16 (from West 2 Spoke 1), and 10.4.0.0/16 (from West 2 Spoke 2). At this point, however, the MSEEs are the only capable routing device that knows about these individual routes. As the routes get advertised from the MSEEs, the HUB in each region will receive the neighboring region routes and propagate this route down into the spokes. As a result, each spoke now has a new route to reach networks in the other region. Since the MSEEs are aware of the individual routes, we are now able to route back up ExpressRoute to the other spoke.

The advantage to this approach is by simply bow-tie connecting the ExpressRoute Gateways in each region to each regions ExpressRoute circuits you gain the ability for the inter-region hub to hub, hub to spoke and spoke to spoke routing natively within the Azure backbone with no transit fee.  The downside to this approach is that traffic is limited to the bandwidth of the ExpressRoute gateway SKU size and latency of hair-pinning off of the MSEEs which are in the peering location of your ExpressRoute circuit. For example, if I am using the Standard Express Route Gateway SKU, I have a bandwidth limit of 1G. This throughput limit would also apply to spoke to spoke communications using this method.  If the latency is X ms, it is doubled due to hair-pinning at the peering location.  The ExpressRoute Gateway is in the path for ingress traffic only.  

# Option 2: Leveraging a HUB NVA

A secondary option for allowing Inter Region spoke to spoke communication would be to leverage an NVA in each HUB VNET to route traffic between the spokes. In this scenario, we deploy an NVA of our choosing into the HUB and define UDRs within each spoke to route to the NVA to get to the other spoke.

![alt text](https://github.com/jgmitter/images/blob/master/jg6.png)
 
As opposed to Option 1 in which we are routing all down to the MSEEs, we are manually doing this at the HUB level. As a result, some lower latency as we are communicating through a HUB via Global VNET Peering rather than through an MSEE. We do not have the bandwidth limitation of the ExpressRoute gateway on ingress traffic patterns as this is no longer in the path. One final advantage of leveraging an NVA for this functionality would be the granular control and inspection capabilities this NVA comes with. Spoke to spoke traffic can now be fully inspected and subject to the NVAs granular policy set using this method. 

The disadvantages of this approach come first with the cost of deploying an NVA. Regardless of NVA chosen, there will be an additional cost for running the NVA and, typically, a throughput cost for the traffic traversing the NVA. While we lose the bandwidth limitation of the ExpressRoute gateway in this scenario, NVAs have bandwidth limitations of their own, which could be a bottleneck for this traffic. Understanding the specific throughput limitations of the NVA chosen would be critical when leveraging this method. Finally, when leveraging Option 1, the Azure fabric recognizes this data path and does not apply charges for the traffic when it traverses the VNET peerings. If using an NVA, all traffic which traverses VNET peerings to reach the NVA will incur vnet peering costs.

# Option 3: Global VNET Peering

The 3rd option for allowing spoke to spoke communication would be to directly Global VNET peer the spokes requiring communication. Similar to how each spoke is VNET peered to the HUB, and a Global VNET peer would be created between the two spokes which require communication.

![alt text](https://github.com/jgmitter/images/blob/master/jg7.png)

The advantage of this option is spokes are now directly connected via the Azure backbone and have the lowest latency and highest bandwidth path possible.

The disadvantage of this approach is the additional cost associated with Global VNET peering as well as the scale limitations should spokes continue to grow. As multiple spokes are introduced behind a hub and require connectivity, a full mesh of VNET peers would be required to provide this connectivity.

![alt text](https://github.com/jgmitter/images/blob/master/jg8.png)
 
Reference: https://azure.microsoft.com/en-us/pricing/details/virtual-network/
Pricing Calculator to explore the options defined above: https://azure.microsoft.com/en-us/pricing/calculator/




# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

# Legal Notices

Microsoft and any contributors grant you a license to the Microsoft documentation and other content
in this repository under the [Creative Commons Attribution 4.0 International Public License](https://creativecommons.org/licenses/by/4.0/legalcode),
see the [LICENSE](LICENSE) file, and grant you a license to any code in the repository under the [MIT License](https://opensource.org/licenses/MIT), see the
[LICENSE-CODE](LICENSE-CODE) file.

Microsoft, Windows, Microsoft Azure and/or other Microsoft products and services referenced in the documentation
may be either trademarks or registered trademarks of Microsoft in the United States and/or other countries.
The licenses for this project do not grant you rights to use any Microsoft names, logos, or trademarks.
Microsoft's general trademark guidelines can be found at http://go.microsoft.com/fwlink/?LinkID=254653.

Privacy information can be found at https://privacy.microsoft.com/en-us/

Microsoft and any contributors reserve all other rights, whether under their respective copyrights, patents,
or trademarks, whether by implication, estoppel or otherwise.
