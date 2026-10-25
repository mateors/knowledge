
What is FRRouting?

FRRouting (FRR) is an open-source, high-performance, internet-scale routing software suite for Linux and Unix platforms. It is a direct fork of the historical Quagga routing engine.
FRR turns a standard Linux server into an enterprise-grade, bare-metal hardware router. Instead of buying expensive proprietary routing hardware from vendors like Cisco or Juniper, companies like Amazon, Slack, and cloud providers run FRR directly on commodity white-box servers to drive their massive data center networks.

Core Architecture: The Multi-Daemon Design

Unlike traditional routing software that runs as a single, bloated process, FRR implements a highly resilient multi-daemon architecture. Every routing protocol runs inside its own isolated operating system process.
* zebra (The Core Engine): The master coordinator daemon. It does not calculate routes itself; instead, it manages the underlying Linux kernel routing tables, handles interface states, and acts as the broker between all protocol-specific daemons.
* Protocol Daemons: Separate processes dedicated entirely to specific math and state tracking for individual protocols:
	* bgpd: Handles Border Gateway Protocol (BGP-4 and BGP-4+).
	* ospfd / ospf6d: Handles Open Shortest Path First (IPv4 & IPv6).
	* isisd: Handles Intermediate System to Intermediate System routing.
	* pbrd: Handles Policy-Based Routing.
	* staticd: Manages local static routes.
* The Resilience Benefit: If the ospfd process encounters a memory bug and crashes, your BGP sessions (bgpd) keep running entirely uninterrupted while the OS restarts the OSPF daemon.

Essential Protocols to Master inside FRR

When building a custom cloud provider network (like your LxRoot infrastructure), you do not need to learn every feature. Focus intensely on these three pillars:

1. BGP (Border Gateway Protocol) via bgpd

* What it does: Connects separate networks (Autonomous Systems) together. It is the core routing engine of the global internet.
* Cloud Context: You will use this to peer your cluster with your upstream ISPs, announcing your custom APNIC ASN and your public IPv4/IPv6 address blocks so the global internet knows how to send traffic to your servers.

2. EVPN-VXLAN (Ethernet VPN)

* What it does: An extension of BGP that allows you to stretch an isolated Layer 2 network (a virtual switch/VLAN) across a physical Layer 3 routed underlay network.
* Cloud Context: This is how you allow a customer's VM on Hypervisor Node 1 to talk directly (via private MAC addresses) to their other VM on Hypervisor Node 5 without passing through an external router.

3. OSPF / IS-IS via ospfd / isisd

* What it does: Interior Gateway Protocols (IGPs). They find the fastest path between routers inside your own internal data center data plane.
* Cloud Context: Used to dynamically map out paths between your Top-of-Rack physical switches and your internal bare-metal hypervisor nodes.

Interactive Configuration Environment (vtysh)

FRR provides a unified shell interface called vtysh. Instead of configuring individual Linux text configuration files manually, vtysh gives your engineers a single Command Line Interface (CLI) that looks, feels, and acts exactly like Cisco IOS or Arista EOS network operating systems.

## Configuration Workflow Example:

To configure FRR, you open the `terminal shell` and execute standardized Cisco-style commands:  `text`

# Enter the unified shell
`sudo vtysh`

# Switch to Global Configuration Mode
router-node# `configure terminal`

# Access the BGP Routing engine configuration
```
router-node(config)# router bgp 65530
router-node(config-router)# neighbor 192.168.1.1 remote-as 65531
router-node(config-router)# address-family ipv4 unicast
router-node(config-router-af)# network 203.0.113.0/24
router-node(config-router-af)# exit
router-node(config)# exit
```

# Write changes permanently to the system config files
router-node# `write memory`
Use code with caution.

## Step-by-Step Learning Roadmap

To go from absolute beginner to deploying FRR in your public cloud stack, execute these learning phases:
[Phase 1: Local Linux Sandbox] ➔ [Phase 2: BGP Peering Theory] ➔ [Phase 3: The Data Center Underlay] ➔ [Phase 4: Go Control Plane Integration]

## Phase 1: The Local Linux Sandbox

* Task: Install FRR on a standard Ubuntu or Debian server (sudo apt install frr).
* Goal: Learn how to edit /etc/frr/daemons to toggle individual protocols (bgpd=yes, ospfd=yes), start the system services, and log into vtysh to inspect interfaces using commands like show interface brief or show ip route.

## Phase 2: BGP Peering Theory & Lab Execution

* Task: Use a tool called Containerlab or GNS3 on your local machine to spin up a virtual topology containing three FRR routers connected together in a loop.
* Goal: Establish BGP neighbors between them. Practice writing Route Maps and Prefix Lists to permit or deny specific subnets from being advertised between the nodes.

## Phase 3: The Data Center Underlay (EVPN-VXLAN)

* Task: Step away from simple routing and configure BGP EVPN.
* Goal: Map a virtual Linux Bridge interface inside the OS to a BGP EVPN instance. Confirm that MAC address tables are dynamically syncing over the network between two isolated Linux nodes using FRR as the control engine.

## Phase 4: Go Control Plane Integration

* Task: Programmatically interface your custom cloud management daemon with FRR.
* Goal: Instead of manually typing commands into vtysh, write a Go service that acts as an orchestration engine. Your Go code will dynamically write configuration templates or interact with FRR's underlying APIs (like gRPC or Netlink structures) to create network channels automatically whenever a new customer requests a public IP address block.

