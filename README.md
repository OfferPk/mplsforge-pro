\# MPLSForge Pro (v1.0.0)

\> \*\*Zero-Trust Multiprotocol Label Switching (RFC 3031 / RFC 3032\) & LDP / RSVP-TE Signaling Studio, Label Switched Path (LSP) Shim Header (20-bit Label, 3-bit Traffic Class/EXP, 1-bit S bottom-of-stack, 8-bit TTL) Hex Wireframe Generator, Penultimate Hop Popping (PHP) & LDP-IGP Synchronization Auditor & Multi-Vendor Router Synthesizer.\*\*

\[\!\[Standards\](https\://img.shields.io/badge/Standards-RFC%203031%20%7C%20RFC%203032%20%7C%20RFC%205443%20%7C%20RFC%206790-cyan)\](\#)  
\[\!\[Telemetry\](https\://img.shields.io/badge/Telemetry-MPLS%20%7C%20EVPN%20%7C%20SR--MPLS%20%7C%20L3VPN-purple)\](\#)  
\[\!\[Tests\](https\://img.shields.io/badge/Tests-8%2F8%20Passed-green)\](\#)  
\[\!\[License\](https\://img.shields.io/badge/License-Commercial%20%2F%20Developer-blue)\](\#)

\---

\#\# Overview

Multiprotocol Label Switching (MPLS, RFC 3031 / RFC 3032\) forms the backbone of global telecommunications, Tier-1 ISP transit networks, 5G mobile backhaul, and multi-tenant cloud data centers (BGP/MPLS L3VPN RFC 4364, EVPN RFC 7432).

However, real-world MPLS deployments face severe operational failures:  
1\. \*\*Label Stack Misalignment & MTU Blackholing\*\*: Each 32-bit (4-byte) MPLS shim label added to an IP payload increases MTU requirements. If core P (provider) routers have standard 1500B MTU or lack path MTU discovery (PMTUD) for label-encapsulated frames, packets exceeding MTU are silently dropped with zero ICMP unreachable messages returned (the "MPLS black hole").  
2\. \*\*Penultimate Hop Popping (PHP) vs Explicit Null Hazards\*\*: When P routers pop the top label using Implicit Null (Label 3), egress PE routers only receive inner VPN labels or naked IP packets. For 6PE/6VPE over IPv4 core, Explicit Null (Label 2\) is required to maintain protocol identification and prevent packet drops.  
3\. \*\*LDP-IGP Synchronization Failures\*\*: In mixed OSPF/IS-IS and LDP networks, when an IGP link comes up faster than the LDP adjacency forms, IGP begins routing traffic over the link before label mappings are exchanged, causing total traffic drop until LDP converges (up to several minutes of blackholing). LDP-IGP sync with high IGP metric holds (RFC 5443\) is required.  
4\. \*\*Reserved Label Collisions & TTL Exposure\*\*: Misconfiguration of reserved labels (Label 0: IPv4 Explicit Null, Label 1: Router Alert, Label 2: IPv6 Explicit Null, Label 3: Implicit Null, Labels 4–15: Reserved) and non-compliant TTL propagation exposes internal core IP addresses to public traceroutes.  
5\. \*\*Entropy Label & ECMP Hash Imbalance\*\*: Under modern spine-leaf fabrics, static MPLS labels defeat hardware ECMP hashing, causing polarization and link saturation unless RFC 6790 Entropy Labels (EL / ELI Label 7\) are engineered.

\*\*MPLSForge Pro\*\* provides an interactive client-side web studio and standalone Python DevSecOps CLI tool to encode valid RFC 3032 message frames, calculate path MTU overhead and headroom budgets, audit backbone configurations against a 10-point Zero-Trust rubric (0–100%, Grades A+ to F), and synthesize production configurations for \*\*Cisco IOS-XR\*\*, \*\*Juniper Junos\*\*, \*\*Nokia SR-OS\*\*, and \*\*Linux Kernel MPLS\*\*.

\---

\#\# Key Capabilities

1\. \*\*Interactive RFC 3032 Label Stack & 32-Bit Shim Header Wireframe Generator\*\*:  
   \- Encodes complete 32-bit MPLS Label Stack Entries (LSE): 20-bit Label (0 to 1,048,575), 3-bit Traffic Class (TC / EXP, 0–7), 1-bit Bottom of Stack (S, 0 or 1), 8-bit Time to Live (TTL, 1–255).  
   \- Generates formatted Wireshark-style hex dumps (\`0000: 00 00 5e 00 53 01 ...\`) with byte offset inspection and ASCII decoding.  
2\. \*\*Path MTU & Frame Overhead Budget Calculator\*\*:  
   \- Calculates Total MPLS Packet Size, Total L2 Wire Frame Size, and Headroom Margin against configured interface MTU.  
   \- Detects MTU undersizing across PE-P-P-PE links with automated alert thresholds.  
3\. \*\*10-Point Zero-Trust MPLS Backbone Security & Resilience Scorecard (0–100%, Grades A+ to F)\*\*:  
   \- Evaluates: MPLS MTU Sizing & Headroom, LDP-IGP Synchronization (RFC 5443), Penultimate Hop Popping & Null Label Hygiene, QoS DiffServ Tunneling Model (Pipe vs Short-Pipe vs Uniform), TTL Propagation & Core Topology Hiding (RFC 3443), Control Plane Signaling Authentication (TCP-AO / MD5), Fast Reroute (TI-LFA / RSVP-TE), RFC 6790 Entropy Labels, Reserved Label Space Isolation (0–15), and BGP/MPLS L3VPN Multi-Tenancy.  
4\. \*\*Multi-Vendor Configuration Synthesizer\*\*:  
   \- \*\*Cisco IOS-XR\*\*: \`interface ... mpls mtu 9216\`, \`mpls ldp\`, \`igp sync delay on-session-up 15\`, \`mpls ip-ttl-propagate disable\`, \`mpls ipv6 explicit-null\`.  
   \- \*\*Juniper Junos\*\*: \`set interfaces ... family mpls mtu 9216\`, \`set protocols mpls no-propagate-ttl\`, \`set protocols ldp ldp-synchronization hold-time 15\`.  
   \- \*\*Nokia SR-OS\*\*: \`/configure router interface ... mpls-mtu 9216\`, \`/configure router ldp ldp-sync-timers hold-down 15\`, \`/configure router mpls ttl-propagate local false\`.  
   \- \*\*Linux Kernel MPLS\*\*: \`modprobe mpls\_router\`, \`sysctl \-w net.mpls.platform\_labels=1048575\`, \`ip link set dev eth0 mtu 9216\`, \`ip \-f mpls route add 100 as to 200 via inet ...\`.  
   \- \*\*Prometheus MPLS Exporter\*\*: Standalone Python telemetry script exposing \`mpls\_interface\_mtu\_bytes\`, \`mpls\_mtu\_headroom\_bytes\`, \`mpls\_ldp\_igp\_sync\_status\`, and \`mpls\_fast\_reroute\_active\` on \`:9129/metrics\`.  
5\. \*\*Standalone DevSecOps CLI Tool (\`mplsforge-cli.py\`)\*\*:  
   \- Pure Python 3 CLI supporting \`encode-shim\`, \`calc-mtu\`, \`audit\` (with \`--strict\` exit code 1 gating for CI/CD), and \`synthesize\`.  
6\. \*\*3 Industry Presets\*\*:  
   \- \`telecom-tier1-isp-core.json\`: Tier-1 ISP Core Backbone with Cisco IOS-XR / Juniper Junos, 9216 MTU, LDP-IGP sync, Pipe QoS mode, TI-LFA Fast Reroute, RFC 6790 Entropy Labels, Grade A+ (100/100).  
   \- \`cloud-wan-evpn-spine-leaf.json\`: Multi-tenant Cloud WAN / Data Center Interconnect (DCI) with EVPN over SR-MPLS, Nokia SR-OS / Arista, 9192 Jumbo MTU, Explicit Null 6PE, Short-Pipe QoS, Grade A+ (95/100).  
   \- \`vulnerable-blackhole-mtu-desync.json\`: Hazardous legacy backbone with 1500 byte standard MTU (causing silent MPLS drops), LDP-IGP sync disabled, unauthenticated LDP TCP sessions, missing FRR, Grade F (19/100).

\---

\#\# Directory Structure

\`\`\`  
mpls-fabric-forge/  
├── source-code/  
│   ├── index.html                    \# Web studio UI  
│   ├── mpls-forge.css                \# Carrier Backbone / NOC Operations Dark UI theme  
│   ├── mpls-engine.js                \# RFC 3032 32-bit shim encoder & MTU overhead math  
│   ├── mpls-security-auditor.js      \# 10-point Zero-Trust MPLS auditor  
│   ├── mpls-config-synthesizer.js    \# Multi-vendor configuration synthesizer  
│   └── app.js                        \# UI state orchestrator  
├── build/  
│   ├── mplsforge-pro-v1.0.0.zip      \# Distributable package  
│   └── mplsforge-cli.py              \# Standalone Python CLI & CI/CD tool  
├── presets/  
│   ├── telecom-tier1-isp-core.json  
│   ├── cloud-wan-evpn-spine-leaf.json  
│   └── vulnerable-blackhole-mtu-desync.json  
├── documentation/  
│   ├── ARCHITECTURE.md               \# Technical wireframe & byte layout specs  
│   ├── USER\_GUIDE.md                 \# User manual & CI/CD integration guide  
│   └── MONETIZATION\_AND\_MARKETING\_PLAYBOOK.md  
├── screenshots/  
│   ├── mpls-studio-preview.svg  
│   └── mpls-architecture.svg  
└── tests/  
    └── test\_mpls\_suite.py            \# Automated test suite (8/8 passed)  
\`\`\`

\---

\#\# Quickstart

\#\#\# Running the Web Studio  
Open \`source-code/index.html\` directly in any modern web browser.

\#\#\# Running the Automated Tests  
\`\`\`bash  
python3 tests/test\_mpls\_suite.py  
\`\`\`

\#\#\# Running the Standalone CLI  
\`\`\`bash  
\# Encode RFC 3032 MPLS Packet Frame Dump  
python3 build/mplsforge-cli.py encode-shim \-p presets/telecom-tier1-isp-core.json

\# Calculate MTU Headroom & Blackhole Risk  
python3 build/mplsforge-cli.py calc-mtu \--iface-mtu 9216 \--payload-mtu 1500 \--labels 3

\# Audit MPLS configuration with strict CI/CD gate  
python3 build/mplsforge-cli.py audit \-p presets/telecom-tier1-isp-core.json \--strict

\# Synthesize Cisco IOS-XR configuration  
python3 build/mplsforge-cli.py synthesize \-p presets/telecom-tier1-isp-core.json \-f cisco  
\`\`\`  
