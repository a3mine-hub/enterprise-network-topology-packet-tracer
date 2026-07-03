# enterprise-network-topology-packet-tracer
A self-built simulation of a small enterprise network with two departments (Sales, IT), dynamic routing, NAT to a simulated ISP, and full DHCP-to-internet connectivity — built to apply CCNA concepts in a realistic, end-to-end functional environment.

Overview


4 routers: Router0, Router1, Router2 (internal, Cisco 1841) + Router3 (simulated ISP, Cisco CGR1240, hidden inside the Cloud0 object)
2 switches: Switch0 (Sales), Switch1 (IT), trunked together
4 PCs: 2 per department, each pulling a real DHCP lease
Routing: OSPF area 0 across the internal triangle, with a default route injected toward the simulated internet
NAT: Overload (PAT) on Router2, translating both internal VLANs to a single public-style IP
Result: all 4 PCs successfully DHCP and ping 8.8.8.8 (Router3's loopback) across the full OSPF → NAT path


Topology

                    Cloud0 → Router3 (ISP, CGR1240)
                          |
                       Router2 (edge / NAT)
                       /         \
                 Router0 ------- Router1
                    |                |
                 Switch0          Switch1
                /        \       /        \
             PC0/PC1    (trunk)      PC2/PC3
           Sales VLAN10          IT VLAN20

Addressing

NetworkRangeNotesVLAN 10 (Sales)192.168.10.0/24Gateway 192.168.10.1 on Router0VLAN 20 (IT)192.168.20.0/24Gateway 192.168.20.1 on Router1Router0–Router110.0.1.0/30Transit linkRouter0–Router210.0.2.0/30Transit linkRouter1–Router210.0.3.0/30Transit linkRouter2–Router3203.0.113.0/30NAT outsideRouter3 loopback8.8.8.8/32Simulated internet ping target

Interface layout (confirmed via CDP — not symmetric, each router ended up different):


Router0: LAN on Fa0/1
Router1: LAN on Fa0/0
Router2: LAN on Fa0/1 (faces Cloud0)
Router3 (CGR1240): no routable Ethernet port — routed via its Vlan1 SVI instead


Configuration Summary


Routing: OSPF area 0 across Router0/Router1/Router2, advertising both LAN subnets and all three transit links. Router2 injects a default route via default-information originate so the whole domain can reach the internet.
DHCP: Router0 serves the SALES pool (192.168.10.0/24), Router1 serves the IT pool (192.168.20.0/24), each excluding the first 10 addresses.
NAT: Router2 runs ip nat inside source list 1 interface FastEthernet0/1 overload, translating both VLANs out to 203.0.113.1, with a static default route to Router3.
VLANs / trunking: VLAN 10 on Switch0, VLAN 20 on Switch1, trunked together over Fa0/2 on both switches, native VLAN 1 on both ends.


Troubleshooting Log

Eight real faults were hit and resolved during the build, in the order encountered:


OSPF network statement typos — Router0 and Router1 each advertised the other router's LAN subnet instead of their own (copy-paste error). No adjacency issue on the surface, but zero LAN routes propagated correctly until fixed.
Router2's serial interfaces had swapped IPs — assigned based on assumed port order, not actual cabling. Interfaces showed up/up (cabling fine), but OSPF formed zero neighbors because each link's two ends sat on mismatched subnets. Diagnosed by fixing hostnames first (all three routers still said "Router," making CDP output useless), then reading show cdp neighbors.
Cloud0 wasn't actually empty — it was a collapsed cluster hiding a real router (Router3, a CGR1240) that needed its own IP configuration. That device also has no routable Ethernet port — routing had to go through its Vlan1 SVI instead of a normal interface.
A placeholder address got pasted literally into the ip route command, silently failing to parse. The rest of the NAT config went through fine, so the missing default route wasn't obvious until show run | include ip route came back empty.
Switch-to-switch trunk port was configured as access mode on both switches instead of trunk, triggering a native VLAN mismatch (flagged directly by CDP). Both ends had to be converted to trunk mode with matching native VLAN.
VLANs existed on the trunk but not in each switch's local VLAN database — Switch0 never had VLAN 20 defined, Switch1 never had VLAN 10. show interfaces trunk showed the VLAN "allowed" but not "active" until the missing vlan entries were added on each switch.
VLAN access commands were applied to the wrong physical ports — the original config assumed Fa0/1–2 were PC ports, but those were actually the router uplink and switch-to-switch trunk. The real PC ports (Fa0/3–4) were still sitting in default VLAN 1, so DHCP broadcasts never reached the switch's forwarding path toward the router. Found by hovering over the actual cables in Packet Tracer's topology view.
DHCP still failed even after every path issue was fixed — Packet Tracer's DHCP radio button only sends a new DHCPDISCOVER on the transition into DHCP mode; it doesn't retry automatically. Toggling to Static and back to DHCP was needed to force a fresh request once the underlying network was actually correct.


The pattern worth naming

Almost every one of these looked like a routing or DHCP problem on the surface, but the root cause was almost always one layer down — L2 (VLAN/trunk) issues masquerading as L3 symptoms. Treating "no connectivity" as a Layer 2 question first, before assuming routing is broken, is the instinct this project reinforced.

Verification

All 4 PCs successfully obtained DHCP leases and pinged 8.8.8.8 (Router3's loopback) end-to-end through the full OSPF → NAT path.

Files


Enterprise_Network_Topology.pkt — Packet Tracer source file
topology.png — topology screenshot
