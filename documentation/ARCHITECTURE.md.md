\# MPLSForge Pro \- Technical Architecture & Protocol Engineering Manual

\#\# 1\. Standards Foundation & Scope

MPLSForge Pro implements the core Internet Engineering Task Force (IETF) standards governing Multiprotocol Label Switching (MPLS):  
\- \*\*RFC 3031\*\*: Multiprotocol Label Switching Architecture  
\- \*\*RFC 3032\*\*: MPLS Label Stack Encoding  
\- \*\*RFC 3270\*\*: Multi-Protocol Label Switching (MPLS) Support of Differentiated Services  
\- \*\*RFC 3443\*\*: Time To Live (TTL) Processing in Multi-Protocol Label Switching (MPLS) Networks  
\- \*\*RFC 4364\*\*: BGP/MPLS IP Virtual Private Networks (VPNs)  
\- \*\*RFC 5036\*\*: LDP Specification  
\- \*\*RFC 5443\*\*: LDP IGP Synchronization  
\- \*\*RFC 6790\*\*: Use of Entropy Labels in MPLS Forwarding  
\- \*\*RFC 7432\*\*: BGP MPLS-Based Ethernet VPN (EVPN)

\---

\#\# 2\. RFC 3032 Label Stack Entry (LSE) Wireframe

Each MPLS shim header is exactly 32 bits (4 octets) long, organized in big-endian network byte order:

\`\`\`  
 0                   1                   2                   3  
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1  
\+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
|                Label (20 bits)        | Exp / TC  |S|   TTL   |  
\+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
\`\`\`

\#\#\# Bit-Field Specifications:  
1\. \*\*Label Value (Bits 0–19, 20 Bits)\*\*:  
   \- Dynamic Range: \`16\` to \`1,048,575\`.  
   \- Special / Reserved Labels (IANA):  
     \- \`Label 0\`: \*\*IPv4 Explicit Null\*\* — Instructs the penultimate router not to pop the label; preserves TC/EXP bits to egress PE.  
     \- \`Label 1\`: \*\*Router Alert\*\* — Directs packet to the local routing processor (used for MPLS OAM, RSVP-TE Hello).  
     \- \`Label 2\`: \*\*IPv6 Explicit Null\*\* — Required for 6PE/6VPE over IPv4 backbones. Informs egress PE that payload is IPv6.  
     \- \`Label 3\`: \*\*Implicit Null\*\* — Signaled by egress PE for Penultimate Hop Popping (PHP). Never transmitted over the wire\!  
     \- \`Label 7\`: \*\*Entropy Label Indicator (ELI)\*\* — RFC 6790 indicator specifying that the subsequent 32-bit word is an Entropy Label.  
     \- \`Label 14\`: \*\*OAM Alert Label\*\* — RFC 3429 Associated Channel Header (ACH).  
2\. \*\*Traffic Class (TC) / Experimental (EXP) (Bits 20–22, 3 Bits)\*\*:  
   \- Values: \`0\` to \`7\`. Used for Differentiated Services (DiffServ) QoS queue selection and ECN marking.  
3\. \*\*Bottom of Stack (S) (Bit 23, 1 Bit)\*\*:  
   \- \`0\`: There are additional Label Stack Entries deeper in the stack.  
   \- \`1\`: This is the innermost label. The byte immediately following this 4-byte LSE is the packet payload (IPv4, IPv6, or Ethernet MAC).  
4\. \*\*Time to Live (TTL) (Bits 24–31, 8 Bits)\*\*:  
   \- Values: \`1\` to \`255\`. Decremented by 1 at each Label Switching Router (LSR). Dropped when TTL \= 0 to prevent forwarding loops.

\---

\#\# 3\. Penultimate Hop Popping (PHP) vs Explicit Null

In classical MPLS forwarding:  
\- \*\*Penultimate Hop Popping (PHP)\*\*: The penultimate LSR pops the top transport label (signaled as Label 3 Implicit Null by the egress PE). The egress PE performs only one lookup (on the inner VPN label or IP destination address).  
  \- \*Failure Mode\*: If an IPv6 customer packet (6PE) traverses an IPv4-only core and the penultimate hop pops the label, the egress PE receives a raw IPv6 packet on an IPv4 core interface without protocol demarcation, dropping the packet.  
\- \*\*Explicit Null (Label 0 / Label 2)\*\*: The penultimate router swaps the top label with Label 0 (IPv4) or Label 2 (IPv6). The egress PE receives the Explicit Null label, preserving the 3-bit TC/EXP QoS bits and explicit payload protocol identification.

\---

\#\# 4\. QoS DiffServ Tunneling Models (RFC 3270\)

MPLSForge Pro supports the three standard RFC 3270 tunneling architectures:  
1\. \*\*Pipe Mode\*\*:  
   \- Ingress PE maps customer DSCP to an internal MPLS EXP value.  
   \- Core transit LSRs schedule packets based solely on MPLS EXP.  
   \- Egress PE pops the MPLS label and forwards the original, unmodified customer IP DSCP.  
   \- \*Advantage\*: Core backbone QoS is completely decoupled from customer markings.  
2\. \*\*Short-Pipe Mode\*\*:  
   \- Similar to Pipe Mode in transit.  
   \- At the egress PE, after label popping, packet scheduling to the CE is based on the customer's original IP DSCP rather than the popped MPLS EXP.  
   \- \*Advantage\*: Best suited for Service Provider L3VPN customer handoffs.  
3\. \*\*Uniform Mode\*\*:  
   \- Ingress PE copies customer IP DSCP into MPLS EXP.  
   \- If an LSR remarques EXP in the core, the egress PE rewrites the inner IP DSCP upon popping.  
   \- \*Risk\*: Malicious or misconfigured customer traffic can manipulate core priority queues.

\---

\#\# 5\. LDP-IGP Synchronization & High-Metric Protection (RFC 5443\)

When a failed link or router reboots:  
1\. Physical link goes UP and IGP (OSPF/IS-IS) establishes adjacency in milliseconds.  
2\. If LDP has not yet established its TCP session (port 646\) and exchanged label bindings, IGP immediately routes traffic over the link.  
3\. Because no MPLS label mappings exist, the router cannot forward label-switched traffic, resulting in \*\*100% packet blackholing\*\* for up to 30–60 seconds.  
4\. \*\*MPLSForge Pro Defense\*\*: Enforces RFC 5443 LDP-IGP Sync. IGP advertises maximum metric (e.g., OSPF \`65535\` or IS-IS wide metric \`16777214\`) until LDP session is active and label bindings converge, routing transit traffic along alternate paths.

\---

\#\# 6\. RFC 6790 Entropy Labels & ECMP Polarization

Modern spine-leaf fabrics utilize Equal-Cost Multi-Path (ECMP) hashing across dozens of parallel links:  
\- Transit LSRs only parse the top 1 or 2 MPLS labels. Because all packets destined to the same egress PE have the identical transport label, traditional routers hash all packets onto a single physical link, saturating it while parallel links remain idle (ECMP polarization).  
\- \*\*Entropy Label (EL)\*\*: Ingress PE hashes packet 5-tuples (Src IP, Dst IP, Protocol, Src Port, Dst Port) into a random 20-bit label value inserted near the bottom of the stack, preceded by Label 7 (ELI).  
\- Transit LSRs hash on the Entropy Label without requiring deep packet inspection (DPI), achieving perfect line-rate load distribution.  
