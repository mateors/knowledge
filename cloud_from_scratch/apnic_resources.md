# Infrastructure

Operating at the infrastructure tier with your own `Autonomous System Number` (`ASN`) and `APNIC-allocated` IP blocks shifts your project into building an enterprise-grade cloud provider architecture (akin to AWS EC2 or DigitalOcean).
Instead of hiding your infrastructure behind standard Consumer NAT routers, your multi-node cluster must be engineered to participate directly in internet-wide core routing using `BGP` (Border Gateway Protocol). Your Go control plane will be responsible for orchestrating physical top-of-rack switches, Linux host hypervisors, and routing daemons dynamically.

Here is your comprehensive, enterprise-level engineering blueprint:

This document serves as your operational blueprint for constructing a carrier-grade bare-metal cloud platform using Go (Golang).

## Core Architectural Layers Detailed in the Guide

1. APNIC Resource Management & IPAM Architecture:
	• Designing an IP Asset Management (IPAM) microservice in Go capable of handling prefix segmentation down to /32 (IPv4) and /64 (IPv6) single allocations.
	• Integrating data stores like Couchbase to hold atomic leases, ensuring zero subnet overlaps across tenants.
2. Infrastructure Edge Routing & BGP Anycast:
	• Orchestrating FRRouting (FRR) or GoBGP instances on your hypervisor nodes.
	• Advertising tenant-allocated APNIC IP blocks directly to your upstream Tier-1 ISP transits via BGP.
	• Implementing EVPN-VXLAN structures to safely tunnel private L2 layers across L3 physical spine-leaf data center configurations.
3. Go-Driven Compute & Network Orchestration Code:
	• A full, production-ready Go codebase demonstrating how to programmatically attach a KVM virtual machine (libvirt) to a dynamically provisioned VXLAN interface and an internal Linux Bridge.
4. Resilient Control Plane Blueprint:
	• Structuring your centralized API scheduler to talk to remote hypervisor worker daemons securely over gRPC/mTLS.

## Step-by-Step Implementation Strategy for Your Team

To build this safely without accidentally misconfiguring your public APNIC prefixes or causing internet routing disruptions, follow this phased progression:

• Step 1: The Closed Lab Topology (Simulated Environment)
Do not announce your real APNIC ASN or public IPs to your upstream ISP yet. Set up a local test lab using GNS3 or Containerlab. Use private ASNs (e.g., 64512 to 65534) and private subnets (10.0.0.0/8 and fd00::/8) to test your Go automation code against virtual FRR routers.
• Step 2: Hypervisor Local Bridging
Compile the provided Go orchestration code on a single test server. Confirm that your script successfully creates a VXLAN interface, assigns an experimental IP address, spins up a KVM instance via libvirt, and attaches the VM to that bridge automatically.
• Step 3: Upstream Transit BGP Peer Validation
Coordinate with your data center colocator or upstream network transit providers. Configure your physical hardware switches or edge servers to peer with them over BGP, using tight prefix-list filters to ensure you only announce the exact IP block allocated to you by APNIC.
As you begin laying down the physical and network code for this architecture, let me know:
• What underlying database architecture will back your IPAM and scheduling state? (We can extend this to use the Couchbase atomic KV patterns we discussed earlier to guarantee sub-millisecond, collision-free IP address leasing).
• Which upstream network hardware (e.g., Cisco Nexus, Arista, Juniper) are you using at the Top-of-Rack (ToR) layer, and do you want your Go control plane to interact with them via Netconf/OpenConfig APIs?
