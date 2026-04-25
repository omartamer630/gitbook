# Transit Gateway

## What is Transit Gateway



VPC Transit Gateway is a <mark style="color:red;">network transit hub</mark> used to <mark style="color:red;">interconnect virtual private clouds (VPCs) and on-premises networks.</mark> As your cloud infrastructure expands globally, inter-Region peering connects transit gateways together using the AWS Global Infrastructure. All network traffic between AWS data centers is automatically encrypted at the physical layer.

***

## Example Diagram Transit Gateway



<figure><img src="../.gitbook/assets/image (42).png" alt=""><figcaption><p>Architecture</p></figcaption></figure>

***

## What does Transit Gateway offer

* **Simplified Network Architecture**: Transit GW provides <mark style="color:red;">a centralized hub</mark> for connecting multiple VPCs, on-premises networks, and <mark style="color:red;">VPN connections</mark>. This simplification <mark style="color:red;">reduces the complexity</mark> of managing inter-VPC connectivity and routing configurations
* **Scalability**: Transit GW <mark style="color:red;">supports thousands of VPCs</mark> and handling large volumes of network traffic, This capability <mark style="color:red;">essential for  organizations with growing (Scalable) infrastructure needs.</mark>
* **Transitive Routing**: Transit GW supports transitive (Multi) routing, <mark style="color:red;">enabling traffic to flow between any connected VPCs, even if they are not directly peered with each other.</mark> This capability simplifies network connectivity and eliminates the need of complexity
* **Centralized Route management**: Transit GW <mark style="color:red;">uses a centralized route table, providing control over how traffic is routed</mark> between connected VPCs and on-premises networks. This centralized management simplifies routing configuration and ensures consistent routing policies across the network.
* **Integration with VPN and Direct Connect**: Transit GW seamlessly integrates with VPN and AWS Direct Connect, enabling organizations to extend their on-premises network connectivity to multiple VPCs via a single gateway. <mark style="color:red;">This integration streamlines hybrid cloud deployments and facilitates secure connectivity between on-premises infrastructure and AWS resources.</mark>

Overall, TGW offers <mark style="color:red;">significant value by simplifying network connectivity</mark>, improving scalability, enhancing <mark style="color:red;">visibility and control</mark>, and reducing costs for organizations deploying complex network architectures within AWS

***

## Transit gateway concepts <a href="#concepts" id="concepts"></a>

* **TGW Route table**: A transit gateway <mark style="color:red;">has a default route table and can optionally have additional route tables</mark>. A route table includes dynamic and static routes that decide the next hop based on the destination IP address of the packet. The target of these routes could be any transit gateway attachment. By default, transit gateway attachments are associated with the default transit gateway route table.
* **TGW Attachment**: You can attach the following:
  * One or more VPCs
  * A Connect SD-WAN/third-party network appliance
  * An AWS Direct Connect gateway
  * A peering connection with another transit gateway
  * A VPN connection to a transit gateway
* **TGW Associations**: Each attachment is associated with exactly one route table. Each route table can be associated with zero to many attachments.
* **TGW Route propagation**: Transit gateway route tables allows you to associate a table with a transit gateway attachment. VPC, VPN, Direct Connect gateway, Peering, and Connect attachments are all supported. When associated, <mark style="color:red;">routes for these attachments are propagated from the attachment to the target transit gateway route table</mark>. An attachment can be <mark style="color:red;">propagated to multiple route tables.</mark>

