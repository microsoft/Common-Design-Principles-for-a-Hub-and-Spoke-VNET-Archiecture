# What is a VNet
 
An Azure Virtual Network (VNet) is like a virtual routing and forwarding (VRF) instance in a traditional network.  It provides a logical network building block to create your network infrastructure on Azure.  It is virtual network isolation on the Azure cloud dedicated to your subscription.  You use a VNet to build or create multiple subnets to support your CIDR design. IP Subnet (Think of a VLAN):  Provides full layer-3 semantics and partial layer-2 semantics (DHCP, ARP, no broadcast/multicast).  You can optionally peer or connect VNets with other VNets as long as address ranges do not overlap.  

![alt text](https://github.com/jgmitter/images/blob/master/vnet.jpg)


Use VNets to:
*   Link your on-premise IT infrastructure to create a hybrid connection to Azure via VPN (S2S and P2S) or ExpressRoute Private-Peering with a virtual network gateway deployed within a VNet. https://aka.ms/AA5ynvj and https://aka.ms/AA5yt4p
*   Create a dedicated private cloud-only VNet. Sometimes you don't require a hybrid connection to on-premise for your solution. When you create a VNet, your services and VMs within your VNet can communicate directly and securely with each other in the cloud. 
*   Create a dedicated DMZ for secure Internet access including Azure native security services such as Azure Firewall and Azure WAF or 3rd party NVAs.  https://aka.ms/AA5ygg9
*   Create a Hub and Spoke model. The Hub acts as a transitive routing domain, and the spokes nest your applications dedicated to a business group’s subscription.  https://aka.ms/AA5yt60
*   Control routing behavior and creates layers of segmentation.   When you put a virtual machine on an Azure virtual network, the VM can connect to any other VM on the same virtual network, even if the other VMs are on different subnets. It is possible because a collection of system routes enabled by default allows this type of communication.  Although the default system routes are useful for many deployment scenarios, there are times when you want to customize the routing configuration for your deployments. You can configure the next-hop address to reach specific destinations.  Also, you can create additional layers of segmentation using Network security groups (NSGs).  NSGs are simple, stateful packet inspection devices that use the 5-tuple approach (source IP, source port, destination IP, destination port, and layer four protocol) to create allow/deny rules for network traffic.  But in some situations, you want or need to enable security at high levels of the stack. In such situations, we recommend that you deploy virtual network security appliances provided by Azure or by Azure partners.
*   Privatize the data plane of supported PAAS services such as SQL MI, ASE, APIM, etc. Examples are instead of the transfer of data over public networks transfer over a private network such as your VNet including hybrid connections such as ExpressRoute).  Examples of VNet Integration: (ASE: https://aka.ms/AA5ygph, App Service: https://aka.ms/AA5ygpk, and Private Link: https://aka.ms/AA62xe2
*   Create shared services that can be shared by multiple workloads, such as network virtual appliances (NVAs) and DNS servers, in a single location.  

# Hub and Spoke Architecture

There are plenty of reasons for connecting a single VNet to your ExpressRoute circuit. After all, a VNet can support up to 65,513 IP addresses, thousands of subnets, and thousands of network blocks.  For many, these limits are enough to create an entire virtual data center within a single VNet!  The single VNet model is ideal for customers who have a single subscription for their Azure footprint and prefer to use subnets as a way of creating segmentation for networking security and workload assignment.  
However, for others, and single VNet is not enough to address the complexity of their business model in the Cloud.  This complexity often stems from their need to support multiple subscriptions using a centralized IT model which will offer shared services across different tenants or application stacks.  The ability to assign each tenant to their own subscription allows for a high degree of granularity and segmentation for billing, role-based access, and network security. 

The most flexible, effective way of realizing this topology within Azure is by way of the hub-and-spoke model, where a hub VNet provides shared network, security, and identity services to a variety of spoke VNets that are mapped to different apps or tenants.  Spoke VNets, by default, only see the hub VNet through their peering connection.  This “sequestered spoke” behavior sets up a strong DMZ foundation in the hub to allow greater control between both spoke-to-spoke traffic and spoke-to-on-prem traffic.  For a deeper dive into the benefits of hub-and-spoke, please read https://aka.ms/AA62lke and https://aka.ms/AA633dq.
The most flexible, effective way of realizing this topology within Azure is by way of the hub-and-spoke model, where a hub VNet provides shared network, security, and identity services to a variety of spoke VNet that are mapped to different apps or tenants.  The components of this model include:  

*   Hub VNet:   An Azure VNet used as the hub in the hub-spoke topology. The hub is the central point of connectivity to your on-premises network, and a place to host shared services that can be consumed by the different workloads hosted in the spoke VNets.  The hub vnet is typically mapped directly to an IT subscription to provide network transit (ExpressRoute Gateway, VPN Gateways, NVAs), security, and identity services. Any virtual machines and workloads deployed in the hub VNet will be invisible to tenant subscriptions that are mapped to spoke VNets.  
*   Spoke VNets: One or more Azure VNets that are used as spokes in the hub-spoke topology.   Spoke VNets are focused on tenant workloads. These spoke VNets could be a representation of different application states, application stacks, departments, or lines of business. It is common for IT and security professionals to be granted role-based access (RBAC) to these spokes to maintain infrastructure components such as VNets, Network Security Groups, Route Tables, and so forth.   Spokes can be used to isolate workloads in their VNets, managed separately from other spokes. Each workload might include multiple tiers, with multiple subnets connected through Azure load balancers.   Due to the nature of VNets being non-transitive by default spoke VNets are isolated from each other.
*   VNet peering: VNet peering is a non-transitive low latency connection between two VNets.  Once peered, the VNets exchange traffic by using the Azure backbone, without the need for a router. In a hub-spoke network topology, you use VNet peering to connect the hub to each spoke. You can peer virtual networks in the same region or different regions. You can also configure spokes to use the hub VNet’s ExpressRoute or Azure VPN gateway to communicate with on-prem networks.  When an ExpressRoute circuit is connected into the hub-and-spoke design, the BGP routes that are advertised from on-prem into the ExpressRoute virtual gateway will not natively transit into the spoke VNets. Instead, these routes will populate in the hub VNet only. Likewise, the ExpressRoute gateway will not natively advertise spoke VNet routes out, and instead, will only advertise the network ranges that belong to the hub VNet.  In order to allow the ExpressRoute virtual gateway to advertise routes from hub to spoke, you must select the “Allow Gateway Transit” option in the VNet peering panel of the hub.  And then to allow the spoke VNet routes to transit out of the ExpressRoute gateway, you must select the “Use Remote Gateway” option in the VNet peering panel of the spoke.  Once complete, all on-prem BGP routes advertised through ExpressRoute will be visible in the hub and all participating spokes, and vise-versa. 
*   Resource Groups: The hub VNet, and each spoke VNet, can be implemented in different resource groups, and different subscriptions. When you peer virtual networks in different subscriptions, both subscriptions can be associated with the same or different Azure Active Directory tenant. RGs allows for decentralized management of each workload while sharing services maintained in the hub VNet.
*   GatewaySubnet:  A Gateway subnet is required to build a virtual network gateway. Allocating 32 addresses to this subnet will help to prevent reaching gateway size limitations in the future.   For more information about setting up the gateway, see the following reference architectures, https://aka.ms/AA5z0l9.  For higher availability, you can use ExpressRoute plus a VPN for failover, https://aka.ms/AA5ytjc.   A hub-spoke topology can also be used without a gateway if you don't need connectivity with your on-premises network.
*   Network Security Groups:  Network security groups protect your workloads with distributed ACLs by providing stateful layer 3 and 4 segmentation.   NSGs can be enforced at the host and subnet layer.  NSGs can use service tags which are named monikers for Azure service IPs and removes the need for management of IP addresses.  NSGs also support flow logs for traffic monitoring and troubleshooting.  Optionally, NSGs can use Application Security Groups (ASG) which are named monikers for groups of VMs and allows the removal of management of NSGs via IP addresses.  As of this writing ASGS are only supported within a VNet.  
*   User Defined Routes:  You can create custom routes in your VNets to override Azure's default system routes, or to add additional routes to a subnet's route table. In Azure, you create a route table, then associate the route table to one or more virtual network subnets.  UDRs allow you to route traffic to virtual appliances (NVAs), VPN Gateways, Internet, etc.  https://aka.ms/AA5z1t3.  It's also important to understand how Aure selects a route: https://aka.ms/AA62lkm.
*   Please always reference Azure VNet Subscription Limits: https://aka.ms/AA62lkr

# The diagram below details a transitive Hub VNet and Spoke VNet topology.

![alt text](https://github.com/jgmitter/images/blob/master/hub%20and%20spoke.jpg)





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
