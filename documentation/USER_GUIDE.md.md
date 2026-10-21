\# MPLSForge Pro \- User Guide & DevSecOps Manual

\#\# 1\. Quickstart: Interactive Web Studio

1\. Open \`source-code/index.html\` in any modern web browser (Google Chrome, Firefox, Safari, Edge).  
2\. Select an industry preset from the \*\*Industry Presets\*\* dropdown:  
   \- \*\*Tier-1 ISP Carrier Core Backbone\*\*: Cisco IOS-XR / Juniper Junos with 9216 Jumbo MTU, LDP-IGP sync, TI-LFA Fast Reroute, and Pipe QoS mode (Grade A+).  
   \- \*\*Cloud WAN Multi-Tenant EVPN over SR-MPLS\*\*: Modern Segment Routing with Explicit Null 6PE and Per-VRF isolation (Grade A+).  
   \- \*\*Vulnerable Legacy Core\*\*: Demonstrates the MTU blackhole vulnerability, missing sync, and unauthenticated control plane (Grade F).  
3\. Review the \*\*Path MTU & Frame Overhead Monitor\*\* to verify headroom margins before silent packet drops occur.  
4\. Inspect the \*\*10-Point Zero-Trust Backbone Audit\*\* to view concrete remediation steps for any failing criteria.  
5\. Review the live \*\*RFC 3032 Packet Wireframe (Wireshark Hex Dump)\*\* with bit-exact LSE breakdown.  
6\. Click any tab under \*\*Multi-Vendor Backbone Synthesizer\*\* (Cisco IOS-XR, Juniper Junos, Nokia SR-OS, Linux Kernel, Prometheus Exporter) to copy validated configuration syntax.

\---

\#\# 2\. DevSecOps CLI Tool (\`mplsforge-cli.py\`)

MPLSForge Pro includes a standalone, zero-dependency Python 3 CLI tool located at \`build/mplsforge-cli.py\`.

\#\#\# 2.1 Encoding MPLS Shim Headers into Wireshark Hex Format  
\`\`\`bash  
python3 build/mplsforge-cli.py encode-shim \-p presets/telecom-tier1-isp-core.json  
\`\`\`  
Output:  
\`\`\`text  
\=================================================================  
RFC 3032 MPLS Packet Frame Dump (Wireshark Format):  
\=================================================================  
0000   00 00 5e 00 53 01 00 1a  2b 3c 4d 5e 81 00 00 64   ..^.S...+\<M^...d  
0010   88 47 03 e8 aa ff 00 00  70 ff 14 8f 20 ff 05 dc   .G......p... ...  
0020   1b ff                                              ..                
\=================================================================  
Total Frame Size: 34 Bytes | Label Stack Depth: 4  
  Level 1 \[Transport / IGP\]: Label=16010 (Standard Dynamic), TC=5, S=0, TTL=255 \-\> 0x03e8aaff  
  Level 2 \[Entropy Label Indicator (ELI)\]: Label=7 (Entropy Label Indicator (ELI)), TC=0, S=0, TTL=255 \-\> 0x000070ff  
  Level 3 \[Entropy Label (EL)\]: Label=84210 (Standard Dynamic), TC=0, S=0, TTL=255 \-\> 0x148f20ff  
  Level 4 \[VPN Service Label\]: Label=24001 (Standard Dynamic), TC=5, S=1, TTL=255 \-\> 0x05dc1bff  
\`\`\`

\#\#\# 2.2 Calculating MTU Overhead & Blackhole Vulnerabilities  
\`\`\`bash  
\# Calculate headroom on a standard 1500B interface with 2 labels (causes failure)  
python3 build/mplsforge-cli.py calc-mtu \--iface-mtu 1500 \--payload-mtu 1500 \--labels 2

\# Calculate headroom on a 9216B Jumbo interface with 4 labels  
python3 build/mplsforge-cli.py calc-mtu \--iface-mtu 9216 \--payload-mtu 1500 \--labels 4  
\`\`\`

\#\#\# 2.3 Running 10-Point Zero-Trust Audits in CI/CD Pipelines  
\`\`\`bash  
\# Regular audit  
python3 build/mplsforge-cli.py audit \-p presets/telecom-tier1-isp-core.json

\# Strict CI/CD gate: exits with code 1 if score \< 90 or grade is not A/A+  
python3 build/mplsforge-cli.py audit \-p presets/telecom-tier1-isp-core.json \--strict  
\`\`\`

\#\#\# 2.4 Synthesizing Multi-Vendor Router Configurations  
\`\`\`bash  
\# Cisco IOS-XR  
python3 build/mplsforge-cli.py synthesize \-p presets/telecom-tier1-isp-core.json \-f cisco

\# Juniper Junos  
python3 build/mplsforge-cli.py synthesize \-p presets/telecom-tier1-isp-core.json \-f juniper

\# Nokia SR-OS  
python3 build/mplsforge-cli.py synthesize \-p presets/telecom-tier1-isp-core.json \-f nokia

\# Linux Kernel Data Plane (iproute2)  
python3 build/mplsforge-cli.py synthesize \-p presets/telecom-tier1-isp-core.json \-f linux

\# Prometheus Metrics Exporter  
python3 build/mplsforge-cli.py synthesize \-p presets/telecom-tier1-isp-core.json \-f prometheus  
\`\`\`

\---

\#\# 3\. GitHub Actions CI/CD Integration

Embed MPLS network configuration auditing directly into infrastructure repositories:

\`\`\`yaml  
name: MPLS Backbone Audit Gate  
on: \[push, pull\_request\]

jobs:  
  audit-mpls-configs:  
    runs-on: ubuntu-latest  
    steps:  
      \- uses: actions/checkout@v4  
      \- name: Set up Python  
        uses: actions/setup-python@v5  
        with:  
          python-version: '3.11'  
      \- name: Audit Core Presets  
        run: |  
          python3 build/mplsforge-cli.py audit \-p presets/telecom-tier1-isp-core.json \--strict  
          python3 build/mplsforge-cli.py audit \-p presets/cloud-wan-evpn-spine-leaf.json \--strict  
\`\`\`  
