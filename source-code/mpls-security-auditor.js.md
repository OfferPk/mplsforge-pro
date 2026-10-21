/\*\*  
 \* MPLSForge Pro \- 10-Point Zero-Trust MPLS Backbone Security & Resilience Auditor  
 \* Evaluates MTU blackholing, LDP-IGP sync, PHP vs Explicit Null, QoS Pipe mode,  
 \* TTL topology hiding, control plane authentication, FRR/TI-LFA, and Entropy labels.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.MPLSSecurityAuditor \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {

  /\*\*  
   \* Evaluates an MPLS network design configuration against the 10-point Zero-Trust rubric  
   \* @param {Object} cfg \- Network parameters  
   \* @returns {Object} { score, maxScore: 100, percentage, grade, summary, findings }  
   \*/  
  function auditMPLSNetwork(cfg) {  
    const findings \= \[\];  
    let totalScore \= 0;

    // 1\. MPLS MTU Sizing & Headroom Margin  
    const ifaceMtu \= parseInt(cfg.interfaceMtu, 10\) || 1500;  
    const payloadMtu \= parseInt(cfg.payloadMtu, 10\) || 1500;  
    const labelCount \= Math.max(1, parseInt(cfg.labelCount, 10\) || 2);  
    const vlanCount \= parseInt(cfg.vlanTags, 10\) || 0;  
    const requiredMplsSize \= payloadMtu \+ (labelCount \* 4\) \+ (cfg.pwe3ControlWord ? 4 : 0);  
    const requiredL2Size \= requiredMplsSize \+ 14 \+ (vlanCount \* 4);  
    const headroom \= ifaceMtu \- requiredMplsSize;

    let mtuScore \= 0;  
    let mtuStatus \= 'FAIL';  
    let mtuMsg \= '';

    if (headroom \< 0\) {  
      mtuScore \= 0;  
      mtuStatus \= 'CRITICAL';  
      mtuMsg \= \`CRITICAL BLACKHOLE: Interface MTU (\${ifaceMtu}B) is smaller than required MPLS packet size (\${requiredMplsSize}B). Customer packets with payload \${payloadMtu}B will be silently dropped with zero ICMP unreachable generation.\`;  
    } else if (ifaceMtu \>= 9000\) {  
      mtuScore \= 10;  
      mtuStatus \= 'PASS';  
      mtuMsg \= \`EXCELLENT: Jumbo MTU (\${ifaceMtu}B) configured. Generous \${headroom}B headroom safely accommodates deep SR-MPLS segment lists, EVPN, and TI-LFA bypass labels.\`;  
    } else if (headroom \>= 20\) {  
      mtuScore \= 7;  
      mtuStatus \= 'WARN';  
      mtuMsg \= \`ACCEPTABLE: Standard MTU (\${ifaceMtu}B) provides \${headroom}B headroom for current \${labelCount} labels, but cannot support future SR-MPLS segment lists or deep encapsulation.\`;  
    } else {  
      mtuScore \= 3;  
      mtuStatus \= 'WARN';  
      mtuMsg \= \`MARGINAL: Headroom is only \${headroom}B. Adding just \${Math.floor(headroom / 4\) \+ 1} more label(s) will trigger immediate silent packet drops.\`;  
    }

    findings.push({  
      id: 'MTU\_SIZING',  
      name: 'MPLS MTU Sizing & Blackhole Prevention',  
      score: mtuScore,  
      maxScore: 10,  
      status: mtuStatus,  
      description: mtuMsg,  
      remediation: mtuScore \< 10 ? 'Configure Jumbo MTU (\>= 9192 or 9216 bytes) across all P and PE core links.' : null  
    });  
    totalScore \+= mtuScore;

    // 2\. LDP-IGP Synchronization & Max-Metric Protection (RFC 5443 / RFC 6138\)  
    const ldpIgpSync \= \!\!cfg.ldpIgpSync;  
    const ldpHoldTimer \= parseInt(cfg.ldpHoldTimerSeconds, 10\) || 0;  
    let ldpScore \= 0;  
    let ldpStatus \= 'FAIL';  
    let ldpMsg \= '';

    if (cfg.transportProtocol \=== 'SR-MPLS') {  
      ldpScore \= 10;  
      ldpStatus \= 'PASS';  
      ldpMsg \= 'Segment Routing (SR-MPLS) is in use; labels are advertised natively in IGP link-state packets, eliminating LDP-IGP desynchronization race conditions.';  
    } else if (ldpIgpSync && ldpHoldTimer \>= 10\) {  
      ldpScore \= 10;  
      ldpStatus \= 'PASS';  
      ldpMsg \= \`LDP-IGP synchronization is active with high-metric hold timer (\${ldpHoldTimer}s). IGP delays routing traffic over newly restored links until LDP adjacencies and labels converge.\`;  
    } else if (ldpIgpSync) {  
      ldpScore \= 6;  
      ldpStatus \= 'WARN';  
      ldpMsg \= 'LDP-IGP synchronization is enabled, but hold timer is unconfigured or low. Slow-converging LDP sessions may still suffer temporary micro-blackholing.';  
    } else {  
      ldpScore \= 0;  
      ldpStatus \= 'CRITICAL';  
      ldpMsg \= 'LDP-IGP sync is DISABLED. When physical links recover, IGP will route transit traffic immediately before LDP labels are exchanged, blackholing traffic for up to minutes.';  
    }

    findings.push({  
      id: 'LDP\_IGP\_SYNC',  
      name: 'LDP-IGP Synchronization (RFC 5443)',  
      score: ldpScore,  
      maxScore: 10,  
      status: ldpStatus,  
      description: ldpMsg,  
      remediation: ldpScore \< 10 ? 'Enable LDP-IGP sync on OSPF/IS-IS core interfaces with a hold-down timer of at least 15 seconds.' : null  
    });  
    totalScore \+= ldpScore;

    // 3\. Penultimate Hop Popping (PHP) & Explicit Null Hygiene  
    const phpMode \= cfg.phpMode || 'IMPLICIT\_NULL'; // IMPLICIT\_NULL, EXPLICIT\_NULL\_V4, EXPLICIT\_NULL\_V6  
    const hasIpv6Payload \= cfg.hasIpv6 || cfg.payloadType \=== 'ipv6';  
    let phpScore \= 0;  
    let phpStatus \= 'PASS';  
    let phpMsg \= '';

    if (hasIpv6Payload && phpMode \=== 'IMPLICIT\_NULL' && \!cfg.vpnService) {  
      phpScore \= 3;  
      phpStatus \= 'FAIL';  
      phpMsg \= 'RISK: Egress PE signals Implicit Null (Label 3\) for pure 6PE over IPv4 core. When penultimate router pops the label, egress PE receives a naked IPv6 frame on an IPv4-only interface, dropping traffic.';  
    } else if (hasIpv6Payload && (phpMode \=== 'EXPLICIT\_NULL\_V6' || cfg.vpnService)) {  
      phpScore \= 10;  
      phpStatus \= 'PASS';  
      phpMsg \= 'Explicit Null (Label 2\) or inner VPN label in use for IPv6. Guarantees protocol differentiation and preserves QoS TC bits to egress PE.';  
    } else if (phpMode \=== 'EXPLICIT\_NULL\_V4') {  
      phpScore \= 10;  
      phpStatus \= 'PASS';  
      phpMsg \= 'IPv4 Explicit Null (Label 0\) enabled. Preserves QoS EXP/TC bits across penultimate hop for end-to-end SLA verification.';  
    } else {  
      phpScore \= 8;  
      phpStatus \= 'PASS';  
      phpMsg \= 'Standard Penultimate Hop Popping (Implicit Null Label 3\) active. Egress PE saves one lookup cycle; acceptable for standard IPv4/VPN deployments.';  
    }

    findings.push({  
      id: 'PHP\_HYGIENE',  
      name: 'Penultimate Hop Popping (PHP) & Null Label Hygiene',  
      score: phpScore,  
      maxScore: 10,  
      status: phpStatus,  
      description: phpMsg,  
      remediation: phpScore \< 8 ? 'Configure "mpls explicit-null" or ensure VPN service label is signaled for IPv6/6PE core transit.' : null  
    });  
    totalScore \+= phpScore;

    // 4\. QoS DiffServ Tunneling Model (RFC 3270\)  
    const qosModel \= cfg.qosModel || 'UNIFORM'; // PIPE, SHORT\_PIPE, UNIFORM  
    let qosScore \= 0;  
    let qosStatus \= 'PASS';  
    let qosMsg \= '';

    if (qosModel \=== 'PIPE') {  
      qosScore \= 10;  
      qosStatus \= 'PASS';  
      qosMsg \= 'PIPE MODE: Transit provider preserves customer IP DSCP untouched while applying independent core EXP markings. Prevents customer QoS remarking from corrupting core queue priority.';  
    } else if (qosModel \=== 'SHORT\_PIPE') {  
      qosScore \= 10;  
      qosStatus \= 'PASS';  
      qosMsg \= 'SHORT-PIPE MODE: Core uses independent EXP markings; egress PE forwards traffic based on customer original DSCP. Ideal for carrier VPN service delivery.';  
    } else {  
      qosScore \= 5;  
      qosStatus \= 'WARN';  
      qosMsg \= 'UNIFORM MODE: Customer DSCP is mapped directly to MPLS EXP on ingress and copied back on egress. Untrusted customer markings can overwrite core QoS tiers and vice-versa.';  
    }

    findings.push({  
      id: 'QOS\_DIFFSERV\_MODEL',  
      name: 'QoS DiffServ Tunneling Model (RFC 3270)',  
      score: qosScore,  
      maxScore: 10,  
      status: qosStatus,  
      description: qosMsg,  
      remediation: qosScore \< 10 ? 'Transition to Pipe or Short-Pipe mode to decouple internal backbone EXP queues from customer DSCP markings.' : null  
    });  
    totalScore \+= qosScore;

    // 5\. TTL Propagation & Topology Hiding (RFC 3443\)  
    const ttlPropagate \= cfg.ttlPropagate \!== undefined ? \!\!cfg.ttlPropagate : true;  
    let ttlScore \= 0;  
    let ttlStatus \= 'PASS';  
    let ttlMsg \= '';

    if (\!ttlPropagate) {  
      ttlScore \= 10;  
      ttlStatus \= 'PASS';  
      ttlMsg \= 'TTL PROPAGATION DISABLED (Topology Hiding): Ingress PE sets outer MPLS TTL to 255\. Customer traceroute sees the entire MPLS core as a single invisible hop, concealing internal router IPs and topology.';  
    } else if (cfg.edgeAclRestricted) {  
      ttlScore \= 7;  
      ttlStatus \= 'WARN';  
      ttlMsg \= 'TTL propagation enabled with edge ICMP filtering. Internal hops are partially shielded from unauthorized public reconnaissance.';  
    } else {  
      ttlScore \= 4;  
      ttlStatus \= 'WARN';  
      ttlMsg \= 'TTL propagation ENABLED without topology hiding. External users can traceroute and map all internal P-router loopback IP addresses, exposing core nodes to targeted DDoS and port scans.';  
    }

    findings.push({  
      id: 'TTL\_PROPAGATION',  
      name: 'TTL Propagation & Core Topology Hiding (RFC 3443)',  
      score: ttlScore,  
      maxScore: 10,  
      status: ttlStatus,  
      description: ttlMsg,  
      remediation: ttlScore \< 10 ? 'Disable MPLS IP TTL propagation ("no mpls ip-ttl-propagate" or "set protocols mpls no-propagate-ttl") on border PE routers.' : null  
    });  
    totalScore \+= ttlScore;

    // 6\. Control Plane Security & Cryptographic Authentication  
    const authType \= cfg.authType || 'NONE'; // TCP\_AO, MD5, NONE  
    let authScore \= 0;  
    let authStatus \= 'FAIL';  
    let authMsg \= '';

    if (authType \=== 'TCP\_AO') {  
      authScore \= 10;  
      authStatus \= 'PASS';  
      authMsg \= 'TCP Authentication Option (TCP-AO RFC 5925\) enabled with cryptographic SHA-1/AES keys. Immune to replay attacks and MD5 collision vulnerabilities.';  
    } else if (authType \=== 'MD5') {  
      authScore \= 8;  
      authStatus \= 'PASS';  
      authMsg \= 'TCP MD5 signature option (RFC 2385\) enabled for LDP sessions. Protects against casual session hijacking and forged reset injection.';  
    } else if (cfg.transportProtocol \=== 'SR-MPLS' && cfg.igpAuth) {  
      authScore \= 10;  
      authStatus \= 'PASS';  
      authMsg \= 'SR-MPLS with cryptographic HMAC-SHA256 IGP authentication enabled. Control plane sessions are securely authenticated at the link-state layer.';  
    } else {  
      authScore \= 0;  
      authStatus \= 'CRITICAL';  
      authMsg \= 'NO CONTROL PLANE AUTHENTICATION: LDP TCP port 646 sessions are unauthenticated. An attacker on the local link or via spoofed TCP RST can tear down the core LSP or inject rogue label bindings.';  
    }

    findings.push({  
      id: 'CONTROL\_PLANE\_AUTH',  
      name: 'Control Plane Signaling Authentication (RFC 5925 / RFC 2385)',  
      score: authScore,  
      maxScore: 10,  
      status: authStatus,  
      description: authMsg,  
      remediation: authScore \< 8 ? 'Configure LDP TCP MD5 or TCP-AO password on all neighbor peering sessions.' : null  
    });  
    totalScore \+= authScore;

    // 7\. Fast Reroute (FRR) & Path Protection  
    const frrMode \= cfg.frrMode || 'NONE'; // TI\_LFA, RSVP\_FACILITY, RSVP\_1\_TO\_1, NONE  
    let frrScore \= 0;  
    let frrStatus \= 'FAIL';  
    let frrMsg \= '';

    if (frrMode \=== 'TI\_LFA') {  
      frrScore \= 10;  
      frrStatus \= 'PASS';  
      frrMsg \= 'Topology-Independent LFA (TI-LFA) enabled. Guarantees 100% link and node protection coverage with sub-50ms deterministic failover without RSVP signaling overhead.';  
    } else if (frrMode \=== 'RSVP\_FACILITY' || frrMode \=== 'RSVP\_1\_TO\_1') {  
      frrScore \= 8;  
      frrStatus \= 'PASS';  
      frrMsg \= 'RSVP-TE Fast Reroute enabled (sub-50ms local repair). Protects core transit LSPs against fiber cuts and transit node crashes.';  
    } else {  
      frrScore \= 0;  
      frrStatus \= 'CRITICAL';  
      frrMsg \= 'NO FAST REROUTE: Core link or router failures require full IGP reconvergence (hundreds of milliseconds to seconds), causing severe voice/video packet loss and micro-outages.';  
    }

    findings.push({  
      id: 'FAST\_REROUTE',  
      name: 'Fast Reroute (FRR) & Protection Path (RFC 4090 / TI-LFA)',  
      score: frrScore,  
      maxScore: 10,  
      status: frrStatus,  
      description: frrMsg,  
      remediation: frrScore \< 8 ? 'Enable TI-LFA (Topology-Independent Loop-Free Alternate) or RSVP-TE Fast Reroute across core backbone paths.' : null  
    });  
    totalScore \+= frrScore;

    // 8\. Entropy Label (RFC 6790\) & ECMP Polarization Mitigation  
    const entropyLabel \= \!\!cfg.entropyLabel;  
    let entropyScore \= 0;  
    let entropyStatus \= 'PASS';  
    let entropyMsg \= '';

    if (entropyLabel) {  
      entropyScore \= 10;  
      entropyStatus \= 'PASS';  
      entropyMsg \= 'RFC 6790 Entropy Label (EL) and Entropy Label Indicator (ELI Label 7\) enabled. Ingress PE calculates flow hash and inserts EL, enabling all transit P routers to perform perfect ECMP hashing without deep packet inspection.';  
    } else if (cfg.fatPw) {  
      entropyScore \= 8;  
      entropyStatus \= 'PASS';  
      entropyMsg \= 'FAT PW (Flow-Aware Transport RFC 6391\) enabled for pseudowires. Mitigates ECMP polarization on Layer 2 circuits.';  
    } else {  
      entropyScore \= 2;  
      entropyStatus \= 'WARN';  
      entropyMsg \= 'Entropy Labels DISABLED. Multi-label packets will suffer from ECMP hash polarization across ECMP core links, leading to link congestion while adjacent links remain idle.';  
    }

    findings.push({  
      id: 'ENTROPY\_LABEL',  
      name: 'Entropy Label & ECMP Hash Load Balancing (RFC 6790)',  
      score: entropyScore,  
      maxScore: 10,  
      status: entropyStatus,  
      description: entropyMsg,  
      remediation: entropyScore \< 8 ? 'Enable Entropy Label (EL/ELI) capability on ingress PE routers to maximize ECMP link utilization across spine-leaf backbones.' : null  
    });  
    totalScore \+= entropyScore;

    // 9\. Reserved Label Space Isolation & Rogue Binding Filtering  
    const labelRangeMin \= parseInt(cfg.labelRangeMin, 10\) || 16;  
    const labelSecurity \= \!\!cfg.filterReservedLabels;  
    let rsvdScore \= 0;  
    let rsvdStatus \= 'PASS';  
    let rsvdMsg \= '';

    if (labelRangeMin \>= 16 && labelSecurity) {  
      rsvdScore \= 10;  
      rsvdStatus \= 'PASS';  
      rsvdMsg \= \`Dynamic label range strictly bounded (min \${labelRangeMin}). Egress ACLs explicitly filter customer packets containing reserved labels 0–15.\`;  
    } else if (labelRangeMin \>= 16\) {  
      rsvdScore \= 7;  
      rsvdStatus \= 'WARN';  
      rsvdMsg \= 'Dynamic label space starts at 16, but edge ACL does not explicitly drop incoming customer packets with reserved shim labels.';  
    } else {  
      rsvdScore \= 2;  
      rsvdStatus \= 'FAIL';  
      rsvdMsg \= 'DANGEROUS: Label range configured allows dynamic allocation into reserved label space (\< 16), creating protocol collision vulnerabilities.';  
    }

    findings.push({  
      id: 'RESERVED\_LABEL\_HYGIENE',  
      name: 'Reserved Label Space Isolation (Labels 0–15)',  
      score: rsvdScore,  
      maxScore: 10,  
      status: rsvdStatus,  
      description: rsvdMsg,  
      remediation: rsvdScore \< 10 ? 'Ensure dynamic label allocations start at label \>= 16 and filter customer-injected reserved labels at boundary interfaces.' : null  
    });  
    totalScore \+= rsvdScore;

    // 10\. Multi-Tenancy VPN & EVPN Label Separation (RFC 4364 / RFC 7432\)  
    const vpnIsolation \= \!\!cfg.vpnIsolation;  
    const perVrfLabel \= \!\!cfg.perVrfLabel || cfg.vpnLabelAllocation \=== 'per-ce' || cfg.vpnLabelAllocation \=== 'per-vrf';  
    let vpnScore \= 0;  
    let vpnStatus \= 'PASS';  
    let vpnMsg \= '';

    if (vpnIsolation && perVrfLabel) {  
      vpnScore \= 10;  
      vpnStatus \= 'PASS';  
      vpnMsg \= 'Strong Multi-Tenancy: Route Targets strictly isolated with per-VRF/per-CE label allocation, preventing cross-VRF route leaks and customer-to-customer packet spoofing.';  
    } else if (vpnIsolation) {  
      vpnScore \= 7;  
      vpnStatus \= 'WARN';  
      vpnMsg \= 'Per-prefix label allocation in use. Scalable, but shared label tables should be complemented with strict RT import/export sanity policies.';  
    } else {  
      vpnScore \= 3;  
      vpnStatus \= 'FAIL';  
      vpnMsg \= 'Cross-VRF leak risks detected: Weak Route Target partitioning or unvalidated BGP community filtering.';  
    }

    findings.push({  
      id: 'VPN\_MULTI\_TENANCY',  
      name: 'BGP/MPLS L3VPN & EVPN Multi-Tenant Label Isolation (RFC 4364)',  
      score: vpnScore,  
      maxScore: 10,  
      status: vpnStatus,  
      description: vpnMsg,  
      remediation: vpnScore \< 10 ? 'Audit Route Target (RT) community filters and verify per-VRF or per-CE label allocation rules.' : null  
    });  
    totalScore \+= vpnScore;

    // Calculate Grade  
    let grade \= 'F';  
    if (totalScore \>= 95\) grade \= 'A+';  
    else if (totalScore \>= 90\) grade \= 'A';  
    else if (totalScore \>= 80\) grade \= 'B';  
    else if (totalScore \>= 70\) grade \= 'C';  
    else if (totalScore \>= 60\) grade \= 'D';  
    else grade \= 'F';

    let summary \= '';  
    if (grade \=== 'A+' || grade \=== 'A') {  
      summary \= 'Production Zero-Trust Carrier Grade: Backbone is fully resilient against MTU blackholes, LDP desync, topology leakage, and ECMP polarization.';  
    } else if (grade \=== 'B') {  
      summary \= 'Acceptable Carrier Backbone: Core operates reliably, but hardening is recommended for topology hiding, entropy labels, or jumbo MTU margins.';  
    } else if (grade \=== 'C' || grade \=== 'D') {  
      summary \= 'Elevated Risk: Vulnerable to LDP-IGP desynchronization blackholing, unauthenticated control plane sessions, or marginal MTU headroom.';  
    } else {  
      summary \= 'CRITICAL FAILURE: Network is actively at risk of silent MTU packet loss, LDP desync outages, and rogue label injection attacks.';  
    }

    return {  
      score: totalScore,  
      maxScore: 100,  
      percentage: totalScore,  
      grade,  
      summary,  
      findings  
    };  
  }

  return {  
    auditMPLSNetwork  
  };  
});  
