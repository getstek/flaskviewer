# Inter-container communication over a TCP/IP network

This document elaborates on some of the network options available for Docker container today.

- Bridged network driver
- Overlay network driver
  - Calico
  - Weave
  - Flannel
-  Underlay networking
  - MacVLAN network driver
- Other
  - OpenvSwitch
  - OpenVPN

These are typically separated into _underlay_ and _overlay_ networks.

The default behavior of the Docker bridge and overlay driver is to provide an internal subnet for each bridge or overlay network.

- Bridge networks only have local significance on each host and they provide port mapping to allow containers to reach outside of the host.
- Overlay networks span across hosts and use VXLAN tunneling to keep its IP namespace separate from other networks.

In this context we call the physical network (comprised of the host network adapters and upstream switches & routers) the underlay network.
Between port mapping and overlay tunneling, containers only receive private IP addresses and are not part of the underlay network. This has many advantages:

1. Portability is increased because applications are not tied to the design of the physical network
2. Agility is improved because new networks can be created and destroyed without having to reconfigure physical infrastructure

There are scenarios where it may be more desirable to place containers directly on the underlay network so that they receive an IP address that is on the underlay subnet. These scenarios include:

1. Legacy Applications - Some legacy applications may use hard-coded IP addresses or ports. Applications that require themselves to be advertised on a certain port can cause difficulties with port-contention and may not be suitable to exist on a private bridge or overlay network.
2. Protocol & Application Incompatibilities - Some protocols and applications are incompatible with NAT and port mapping.
3. Security based on MAC filtering and Enterprise IT network management requirements.

# Detailed discussion

### Bridged networking

A Linux bridge provides a host internal network in which containers on the same host may communicate, but the IP addresses assigned to each container are not accessible from outside the host.
Bridge networking leverages iptables for NAT and port-mapping, which provide single-host networking.
Bridge networking is the default Docker network type (i.e., docker0), where one end of a virtual network interface pair is connected between the bridge and the container.

Here’s an example of the creation flow:

1. A bridge is provisioned on the host.
2. A namespace for each container is provisioned inside that bridge.
3. Containers’ ethX are mapped to private bridge interfaces.
4. iptables with NAT are used to map between each private container and the host’s public interface.

NAT is used to provide communication beyond the host.

![Bridge networking overview](/docs/bridge_networking.png?raw=true "Overview")

While bridged networks solve port-conflict problems and provide network isolation to containers running on one host, there’s a performance cost related to using NAT.
Additionally port conflicts can occur if multiple containers wish to use the same host port (for example 80), which can be worked around with service discovery and load balancing but this is not a default configuration and requires additional configuration.

### Overlay networking

#### Calico

Project Calico uses standard IP routing via the Border Gateway Protocol (BGP), as defined in RFC 1105 and networking tools to provide a layer 3 solution.
In contrast, most other networking solutions build an overlay network by encapsulating layer 2 traffic into a higher layer.
The primary operating mode requires no encapsulation and is designed for data centers where the organization has control over the physical network fabric.

![Calico overview](/docs/calico_overview.png?raw=true "Overview")

Key principles of Calico -

- Distribuite routes with BGP
  - Route reflectors provide scaling capability
- Perform L3 forwarding at each node
- Use Linux kernel for IP forwarding without additional vSwitch
- Separate policy and routing decisions
- Implement global policies on a distributed firewall on each host

#### Weave

Weaveworks’ WeaveNet creates a virtual network that connects Docker containers deployed across multiple hosts.
Applications use the network just as if the containers were all plugged into the same network switch, with no need to configure port mappings and links.
Services provided by application containers on the Weave network can be made accessible to the outside world, regardless of where those containers are running.

Similarly, existing internal systems can be exposed to application containers irrespective of their location. Weave has network policy support for Kubernetes.

Weave can traverse firewalls and operate in partially connected networks. Traffic can be encrypted, allowing hosts to be connected across an untrusted network. An additional feature of Weave is Fast DataPath which uses the built-in Linux VXLAN kernel modules to improve packet throughput without encapsulating data in a slower UDP tunnel.

Fast DP is not available across firewalls, as this functionality requires a tunnel in user space to traverse firewalls.

![Weave Overview](/docs/weave_overview.png?raw=true "Weave Overview")

#### Flannel

Flannel is a virtual network that assigns a subnet to each host for use with container runtimes. Each container—or pod, in the case of Kubernetes has a unique, routable IP inside the cluster.
Flannel supports a range of backends, such as VXLAN, and the default layer 2 UDP network. The advantage of flannel is that it reduces the complexity of doing port mapping.
Packets are forwarded using one of several backend mechanisms, which include the built-in defaults:

1. VXLAN
2. UDP

Flannel is responsible for providing a layer 3 IPv4 network between multiple nodes in a cluster. Flannel does not control how containers are networked to the host, only how the traffic is transported between hosts.
Furthermore, Flannel is focused on networking and not network policy, Canal (Calico + Flannel) should be considered if additional network policies are a requirement.

![Flannel Overview](/docs/flannel_overview.png?raw=true "Flannel Overview")

### Underlay networking

Underlay network drivers expose host interfaces (i.e., the physical network interface at eth0) directly to containers or VMs running on the host -
MacVLAN is such a technology.
The operation of and the behavior of MacVLAN may be very familiar to network engineers as it is conceptually simpler than bridge networking, and removes the need for port-mapping, as well as being more efficient at a packet switching level.

Given the restrictions — or lack of capabilities — in most public clouds, underlays are particularly useful when you have on-premises workloads, security concerns, traffic priorities or compliance to deal with, making them ideal for brownfield use.
Instead of needing one bridge per VLAN, underlay networking allows for one VLAN per sub-interface.

#### MacVLAN

MacVLAN allows the creation of multiple virtual network interfaces behind the host’s single physical interface. Each virtual interface has unique MAC and IP addresses assigned, with a restriction: the IP addresses need to be in the same broadcast domain as the physical interface. While many network engineers may be more familiar with the term subinterface (not to be confused with a secondary interface), the parlance used to describe MacVLAN virtual interfaces is typically upper or lower interface. MacVLAN networking is a way of eliminating the need for the Linux bridge, NAT and port-mapping, allowing you to connect directly to the physical interface.

MacVLAN uses a unique MAC address per container, and this may cause an issue with network switches that have security policies in place to prevent MAC spoofing, by allowing only one MAC address per physical switch interface.

Container traffic is filtered from being able to speak to the underlying host, which completely isolates the host from the containers it runs. The host cannot reach the containers. The container is isolated from the host. This is useful for service providers or multitenant scenarios and has more isolation than the bridge model.

![MacVLAN vs. Bridging](/docs/macv_vs_bridge.png?raw=true "MacVLAN vs. Bridging approach")

Promiscuous mode is required for MacVLAN; MacVLAN has four modes of operation, with only the bridge mode supported in Docker 1.12. MacVLAN bridge mode and IPvlan L2 mode are just about functionally equivalent. Both modes allow broadcast and multicast traffic ingress. These underlay protocols were designed with on-premises use cases in mind. Your public cloud mileage will vary as most do not support promiscuous mode on their VM interfaces.

**Side note:** _MacVLAN bridge mode assigning a unique MAC address per container can be a blessing in terms of tracing network traffic and end-to-end visibility;
however, with a typical network interface card (NIC), e.g., Broadcom, having a ceiling of 512 unique MAC addresses, this upper limit should be considered during platform design, sizing, and prior to purchasing of hardware._

### Other technologies

#### OpenvSwitch

Open vSwitch is a multilayer virtual switch designed to enable network automation through programmatic extension while supporting standard management interfaces and protocols, such as NetFlow, IPFIX, LACP, and 802.1ag. In addition, it is designed to support distribution across multiple physical servers and is used in Red Hat’s Kubernetes distro OpenShift, the default switch in Xen, KVM, Proxmox VE, and VirtualBox. It has also been integrated into many private cloud systems, such as OpenStack and oVirt.

#### OpenVPN

OpenVPN, another OSS project that has a commercial offering, allows you to create virtual private networks (VPNs) using TLS. These VPNs can also be used to securely connect containers to each other over the public internet.

Weave however has this same secure functionality, with additional container functionality.

#### Cilium

A BPF-based solution providing connectivity between containers, operating at layer 3/4 to provide networking and security services as well as layer 7 to protect modern protocols such as HTTP and gRPC.

Existing Linux network security mechanisms (e.g., iptables) only operate at the network and transport layers (i.e., IP addresses and ports) and lack visibility into the microservices layer.

Cilium brings API-aware network security filtering to Linux container frameworks. Using a new Linux kernel technology called BPF, Cilium provides a simple and efficient way to define and enforce both network-layer and application-layer security policies based on container/pod identity. Using Cilium policies you can for example restrict access to an endpoint:

- /admin only from authorized users only from authorized apps
- /healthz only to the health monitoring container

![Cilium overview](/docs/cilium_overview.png?raw=true "Overview")

# Comparions of container networking technologies

The document below shows a comparison of other container networking tools, not just those listed here.

- [Container Native Networking](https://docs.google.com/spreadsheets/d/1polIS2pvjOxCZ7hpXbra68CluwOZybsP1IYfr-HrAXc/edit#gid=0)
