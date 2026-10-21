/\*\*  
 \* MPLSForge Pro \- Web Studio Application Controller  
 \*/

document.addEventListener('DOMContentLoaded', () \=\> {  
  const PRESETS \= {  
    'telecom-tier1-isp-core': {  
      name: "Tier-1 ISP Carrier Core Backbone (Cisco / Juniper)",  
      badge: "Tier-1 Carrier",  
      description: "Hardened Tier-1 Global Carrier MPLS Core with 9216 Jumbo MTU, LDP-IGP Sync, TI-LFA Sub-50ms Fast Reroute, RFC 6790 Entropy Labels, and Pipe QoS mode.",  
      hostname: "NYC-CR-01",  
      routerId: "198.51.100.1",  
      coreInterface: "HundredGigE0/0/0/0",  
      coreIp: "10.100.1.1 255.255.255.252",  
      transportProtocol: "LDP",  
      interfaceMtu: 9216,  
      payloadMtu: 1500,  
      vlanTags: 1,  
      pwe3ControlWord: false,  
      ldpIgpSync: true,  
      ldpHoldTimerSeconds: 15,  
      phpMode: "EXPLICIT\_NULL\_V4",  
      hasIpv6: true,  
      qosModel: "PIPE",  
      ttlPropagate: false,  
      authType: "TCP\_AO",  
      frrMode: "TI\_LFA",  
      entropyLabel: true,  
      labelRangeMin: 16,  
      filterReservedLabels: true,  
      vpnService: true,  
      vpnIsolation: true,  
      perVrfLabel: true,  
      stack: \[  
        { role: "Transport / IGP", label: 16010, tc: 5, ttl: 255 },  
        { role: "Entropy Label Indicator (ELI)", label: 7, tc: 0, ttl: 255 },  
        { role: "Entropy Label (EL)", label: 84210, tc: 0, ttl: 255 },  
        { role: "VPN Service Label", label: 24001, tc: 5, ttl: 255 }  
      \]  
    },  
    'cloud-wan-evpn-spine-leaf': {  
      name: "Cloud WAN Multi-Tenant EVPN over SR-MPLS",  
      badge: "Cloud Backbone",  
      description: "Modern Cloud Backbone Interconnect with Segment Routing (SR-MPLS), Explicit Null 6PE, Short-Pipe DiffServ, and Per-VRF Label Multi-Tenancy.",  
      hostname: "SJC-WAN-SPINE-01",  
      routerId: "198.51.100.10",  
      coreInterface: "et-0/0/0",  
      coreIp: "10.100.2.1/30",  
      transportProtocol: "SR-MPLS",  
      interfaceMtu: 9192,  
      payloadMtu: 1500,  
      vlanTags: 0,  
      pwe3ControlWord: false,  
      ldpIgpSync: false,  
      ldpHoldTimerSeconds: 0,  
      phpMode: "EXPLICIT\_NULL\_V6",  
      hasIpv6: true,  
      qosModel: "SHORT\_PIPE",  
      ttlPropagate: false,  
      authType: "TCP\_AO",  
      igpAuth: true,  
      frrMode: "TI\_LFA",  
      entropyLabel: true,  
      labelRangeMin: 16000,  
      filterReservedLabels: true,  
      vpnService: true,  
      vpnIsolation: true,  
      perVrfLabel: true,  
      stack: \[  
        { role: "SRGB Prefix-SID", label: 16020, tc: 6, ttl: 255 },  
        { role: "EVPN Type-2 Mac/IP Label", label: 30005, tc: 6, ttl: 255 }  
      \]  
    },  
    'vulnerable-blackhole-mtu-desync': {  
      name: "Vulnerable Legacy Core (MTU Blackhole & Desync)",  
      badge: "High Risk Core",  
      description: "Hazardous un-engineered MPLS network with 1500 byte standard MTU (dropping packets with labels), LDP-IGP sync disabled, plaintext TCP, and open TTL propagation.",  
      hostname: "LEGACY-P-03",  
      routerId: "10.1.1.3",  
      coreInterface: "GigabitEthernet0/1",  
      coreIp: "10.1.1.2 255.255.255.252",  
      transportProtocol: "LDP",  
      interfaceMtu: 1500,  
      payloadMtu: 1500,  
      vlanTags: 1,  
      pwe3ControlWord: false,  
      ldpIgpSync: false,  
      ldpHoldTimerSeconds: 0,  
      phpMode: "IMPLICIT\_NULL",  
      hasIpv6: true,  
      qosModel: "UNIFORM",  
      ttlPropagate: true,  
      authType: "NONE",  
      frrMode: "NONE",  
      entropyLabel: false,  
      labelRangeMin: 0,  
      filterReservedLabels: false,  
      vpnService: false,  
      vpnIsolation: false,  
      perVrfLabel: false,  
      stack: \[  
        { role: "Outer LDP Label", label: 100, tc: 0, ttl: 64 },  
        { role: "Inner VPN Label", label: 200, tc: 0, ttl: 64 }  
      \]  
    }  
  };

  let currentStack \= JSON.parse(JSON.stringify(PRESETS\['telecom-tier1-isp-core'\].stack));  
  let activeTab \= 'cisco';

  // Elements  
  const presetSelect \= document.getElementById('presetSelect');  
  const presetBadge \= document.getElementById('presetBadge');  
  const presetDesc \= document.getElementById('presetDesc');  
  const cfgHostname \= document.getElementById('cfgHostname');  
  const cfgRouterId \= document.getElementById('cfgRouterId');  
  const cfgCoreInterface \= document.getElementById('cfgCoreInterface');  
  const cfgTransport \= document.getElementById('cfgTransport');  
  const cfgInterfaceMtu \= document.getElementById('cfgInterfaceMtu');  
  const cfgPayloadMtu \= document.getElementById('cfgPayloadMtu');  
  const cfgVlanTags \= document.getElementById('cfgVlanTags');  
  const cfgPwe3ControlWord \= document.getElementById('cfgPwe3ControlWord');  
  const cfgPhpMode \= document.getElementById('cfgPhpMode');  
  const cfgQosModel \= document.getElementById('cfgQosModel');  
  const cfgFrrMode \= document.getElementById('cfgFrrMode');  
  const cfgAuthType \= document.getElementById('cfgAuthType');  
  const cfgLdpIgpSync \= document.getElementById('cfgLdpIgpSync');  
  const cfgLdpHoldTimer \= document.getElementById('cfgLdpHoldTimer');  
  const cfgEntropyLabel \= document.getElementById('cfgEntropyLabel');  
  const cfgTtlPropagate \= document.getElementById('cfgTtlPropagate');  
  const cfgFilterReserved \= document.getElementById('cfgFilterReserved');  
  const cfgVpnIsolation \= document.getElementById('cfgVpnIsolation');

  const labelStackList \= document.getElementById('labelStackList');  
  const btnAddLabel \= document.getElementById('btnAddLabel');

  const mtuStatusBadge \= document.getElementById('mtuStatusBadge');  
  const valTotalMplsSize \= document.getElementById('valTotalMplsSize');  
  const valTotalL2Size \= document.getElementById('valTotalL2Size');  
  const valHeadroom \= document.getElementById('valHeadroom');  
  const valStackDepth \= document.getElementById('valStackDepth');  
  const mtuNotice \= document.getElementById('mtuNotice');

  const auditGradeCircle \= document.getElementById('auditGradeCircle');  
  const auditScoreText \= document.getElementById('auditScoreText');  
  const auditSummaryText \= document.getElementById('auditSummaryText');  
  const auditList \= document.getElementById('auditList');

  const hexViewer \= document.getElementById('hexViewer');  
  const btnCopyHex \= document.getElementById('btnCopyHex');

  const configViewer \= document.getElementById('configViewer');  
  const btnCopyConfig \= document.getElementById('btnCopyConfig');  
  const tabButtons \= document.querySelectorAll('.tab-btn');

  function getConfiguration() {  
    return {  
      hostname: cfgHostname.value.trim() || 'PE-01',  
      routerId: cfgRouterId.value.trim() || '10.0.0.1',  
      coreInterface: cfgCoreInterface.value.trim() || 'HundredGigE0/0/0/0',  
      transportProtocol: cfgTransport.value,  
      interfaceMtu: parseInt(cfgInterfaceMtu.value, 10\) || 1500,  
      payloadMtu: parseInt(cfgPayloadMtu.value, 10\) || 1500,  
      vlanTags: parseInt(cfgVlanTags.value, 10\) || 0,  
      pwe3ControlWord: cfgPwe3ControlWord.checked,  
      phpMode: cfgPhpMode.value,  
      qosModel: cfgQosModel.value,  
      frrMode: cfgFrrMode.value,  
      authType: cfgAuthType.value,  
      ldpIgpSync: cfgLdpIgpSync.checked,  
      ldpHoldTimerSeconds: parseInt(cfgLdpHoldTimer.value, 10\) || 0,  
      entropyLabel: cfgEntropyLabel.checked,  
      ttlPropagate: cfgTtlPropagate.checked,  
      filterReservedLabels: cfgFilterReserved.checked,  
      vpnIsolation: cfgVpnIsolation.checked,  
      perVrfLabel: cfgVpnIsolation.checked,  
      labelCount: currentStack.length,  
      stack: currentStack  
    };  
  }

  function renderStackList() {  
    labelStackList.innerHTML \= '';  
    currentStack.forEach((lse, idx) \=\> {  
      const isBottom \= idx \=== currentStack.length \- 1;  
      const card \= document.createElement('div');  
      card.className \= 'lse-card';  
      card.innerHTML \= \`  
        \<div class="lse-index"\>\${idx \+ 1}\</div\>  
        \<div\>  
          \<input type="text" class="lse-role" value="\${lse.role || ''}" placeholder="Role / Name" style="font-size: 11px; padding: 4px 6px;"\>  
        \</div\>  
        \<div\>  
          \<input type="number" class="lse-label" value="\${lse.label}" min="0" max="1048575" style="font-size: 11px; padding: 4px 6px;"\>  
        \</div\>  
        \<div\>  
          \<select class="lse-tc" style="font-size: 11px; padding: 4px 2px;"\>  
            \${\[0,1,2,3,4,5,6,7\].map(tc \=\> \`\<option value="\${tc}" \${tc \=== (lse.tc || 0\) ? 'selected' : ''}\>TC \${tc}\</option\>\`).join('')}  
          \</select\>  
        \</div\>  
        \<div\>  
          \<input type="number" class="lse-ttl" value="\${lse.ttl \!== undefined ? lse.ttl : 255}" min="1" max="255" style="font-size: 11px; padding: 4px 6px;"\>  
        \</div\>  
        \<div\>  
          \<button class="btn-remove-lse" title="Pop Label"\>\&times;\</button\>  
        \</div\>  
      \`;

      card.querySelector('.lse-role').addEventListener('input', (e) \=\> {  
        lse.role \= e.target.value;  
        update();  
      });  
      card.querySelector('.lse-label').addEventListener('input', (e) \=\> {  
        lse.label \= parseInt(e.target.value, 10\) || 0;  
        update();  
      });  
      card.querySelector('.lse-tc').addEventListener('change', (e) \=\> {  
        lse.tc \= parseInt(e.target.value, 10\) || 0;  
        update();  
      });  
      card.querySelector('.lse-ttl').addEventListener('input', (e) \=\> {  
        lse.ttl \= parseInt(e.target.value, 10\) || 255;  
        update();  
      });  
      card.querySelector('.btn-remove-lse').addEventListener('click', () \=\> {  
        if (currentStack.length \> 1\) {  
          currentStack.splice(idx, 1);  
          renderStackList();  
          update();  
        } else {  
          alert('Label stack must retain at least one entry.');  
        }  
      });

      labelStackList.appendChild(card);  
    });  
  }

  function update() {  
    const cfg \= getConfiguration();

    // 1\. MTU Budget Calculation  
    const mtuRes \= MPLSEngine.calculateMTUBudget({  
      labelCount: cfg.labelCount,  
      payloadMtu: cfg.payloadMtu,  
      vlanTags: cfg.vlanTags,  
      interfaceMtu: cfg.interfaceMtu,  
      pwe3ControlWord: cfg.pwe3ControlWord  
    });

    valTotalMplsSize.textContent \= \`\${mtuRes.totalMplsPacketSize}B\`;  
    valTotalL2Size.textContent \= \`\${mtuRes.totalL2FrameSize}B\`;  
    valHeadroom.textContent \= \`\${mtuRes.headroomL3}B\`;  
    valStackDepth.textContent \= cfg.labelCount;

    if (mtuRes.status \=== 'CRITICAL\_BLACKHOLE') {  
      mtuStatusBadge.className \= 'badge badge-red';  
      mtuStatusBadge.textContent \= 'CRITICAL BLACKHOLE';  
      valHeadroom.className \= 'metric-val alert-critical';  
    } else if (mtuRes.status \=== 'WARNING\_LOW\_HEADROOM') {  
      mtuStatusBadge.className \= 'badge badge-purple';  
      mtuStatusBadge.textContent \= 'LOW HEADROOM';  
      valHeadroom.className \= 'metric-val alert-warn';  
    } else {  
      mtuStatusBadge.className \= 'badge badge-green';  
      mtuStatusBadge.textContent \= 'HEALTHY HEADROOM';  
      valHeadroom.className \= 'metric-val alert-ok';  
    }  
    mtuNotice.textContent \= mtuRes.recommendations.join(' ');

    // 2\. 10-Point Zero-Trust Audit  
    const auditRes \= MPLSSecurityAuditor.auditMPLSNetwork(cfg);  
    auditGradeCircle.className \= \`grade-circle grade-\${auditRes.grade}\`;  
    auditGradeCircle.textContent \= auditRes.grade;  
    auditScoreText.textContent \= \`Score: \${auditRes.score} / 100 (Grade \${auditRes.grade})\`;  
    auditSummaryText.textContent \= auditRes.summary;

    auditList.innerHTML \= '';  
    auditRes.findings.forEach(f \=\> {  
      const item \= document.createElement('div');  
      item.className \= 'audit-item';  
      const icon \= f.status \=== 'PASS' ? '&\#9989;' : (f.status \=== 'WARN' ? '&\#9888;&\#65039;' : '&\#10060;');  
      const ptsColor \= f.status \=== 'PASS' ? '\#34d399' : (f.status \=== 'WARN' ? '\#fbbf24' : '\#f87171');  
      item.innerHTML \= \`  
        \<div class="audit-icon"\>\${icon}\</div\>  
        \<div class="audit-info"\>  
          \<h4\>\${f.name}\</h4\>  
          \<p\>\${f.description}\</p\>  
        \</div\>  
        \<div class="audit-pts" style="color: \${ptsColor};"\>\${f.score}/10 pts\</div\>  
      \`;  
      auditList.appendChild(item);  
    });

    // 3\. RFC 3032 Hex Frame Wireframe  
    const frameRes \= MPLSEngine.encodeMPLSFrame(cfg.stack, {  
      vlan: cfg.vlanTags \> 0 ? 100 : null,  
      vlanCount: cfg.vlanTags,  
      addDummyPayload: true,  
      payloadType: cfg.hasIpv6 ? 'ipv6' : 'ipv4'  
    });

    let hexText \= \`=== RFC 3032 MPLS Packet Frame Dump (Wireshark Hex Format) \===\\n\`;  
    hexText \+= frameRes.hexDump \+ '\\n\\n';  
    hexText \+= \`Total Frame Size: \${frameRes.totalLength} Bytes | L2 Header: \${frameRes.l2HeaderLength}B | MPLS Stack Depth: \${frameRes.labelCount} (\${frameRes.mplsShimLength}B)\\n\`;  
    hexText \+= \`------------------------------------------------------------------------------------\\n\`;  
    frameRes.breakdown.forEach(b \=\> {  
      const rsvd \= b.isReserved ? \` \[RESERVED: \${b.reservedName}\]\` : '';  
      hexText \+= \`  Level \${b.position} \[\${b.role}\]: Label=\${b.label}\${rsvd}, TC=\${b.tc}, S=\${b.s ? 1 : 0}, TTL=\${b.ttl} \-\> Hex: \${b.hex}\\n\`;  
    });  
    hexViewer.textContent \= hexText;

    // 4\. Multi-Vendor Configuration Synthesis  
    renderVendorConfig(cfg);  
  }

  function renderVendorConfig(cfg) {  
    let output \= '';  
    if (activeTab \=== 'cisco') {  
      output \= MPLSConfigSynthesizer.generateCiscoIOSXR(cfg);  
    } else if (activeTab \=== 'juniper') {  
      output \= MPLSConfigSynthesizer.generateJuniperJunos(cfg);  
    } else if (activeTab \=== 'nokia') {  
      output \= MPLSConfigSynthesizer.generateNokiaSROS(cfg);  
    } else if (activeTab \=== 'linux') {  
      output \= MPLSConfigSynthesizer.generateLinuxKernel(cfg);  
    } else if (activeTab \=== 'prometheus') {  
      output \= MPLSConfigSynthesizer.generatePrometheusExporter(cfg);  
    }  
    configViewer.textContent \= output;  
  }

  function loadPreset(key) {  
    const p \= PRESETS\[key\];  
    if (\!p) return;

    presetBadge.textContent \= p.badge;  
    presetDesc.textContent \= p.description;

    cfgHostname.value \= p.hostname;  
    cfgRouterId.value \= p.routerId;  
    cfgCoreInterface.value \= p.coreInterface;  
    cfgTransport.value \= p.transportProtocol;  
    cfgInterfaceMtu.value \= p.interfaceMtu;  
    cfgPayloadMtu.value \= p.payloadMtu;  
    cfgVlanTags.value \= p.vlanTags;  
    cfgPwe3ControlWord.checked \= p.pwe3ControlWord;  
    cfgPhpMode.value \= p.phpMode;  
    cfgQosModel.value \= p.qosModel;  
    cfgFrrMode.value \= p.frrMode;  
    cfgAuthType.value \= p.authType;  
    cfgLdpIgpSync.checked \= p.ldpIgpSync;  
    cfgLdpHoldTimer.value \= p.ldpHoldTimerSeconds;  
    cfgEntropyLabel.checked \= p.entropyLabel;  
    cfgTtlPropagate.checked \= p.ttlPropagate;  
    cfgFilterReserved.checked \= p.filterReservedLabels;  
    cfgVpnIsolation.checked \= p.vpnIsolation;

    currentStack \= JSON.parse(JSON.stringify(p.stack));  
    renderStackList();  
    update();  
  }

  // Event Listeners  
  presetSelect.addEventListener('change', (e) \=\> {  
    loadPreset(e.target.value);  
  });

  const inputs \= \[  
    cfgHostname, cfgRouterId, cfgCoreInterface, cfgTransport, cfgInterfaceMtu,  
    cfgPayloadMtu, cfgVlanTags, cfgPwe3ControlWord, cfgPhpMode, cfgQosModel,  
    cfgFrrMode, cfgAuthType, cfgLdpIgpSync, cfgLdpHoldTimer, cfgEntropyLabel,  
    cfgTtlPropagate, cfgFilterReserved, cfgVpnIsolation  
  \];

  inputs.forEach(input \=\> {  
    input.addEventListener('input', update);  
    input.addEventListener('change', update);  
  });

  btnAddLabel.addEventListener('click', () \=\> {  
    currentStack.push({  
      role: 'Inner Service / VPN',  
      label: 10000 \+ Math.floor(Math.random() \* 50000),  
      tc: 0,  
      ttl: 255  
    });  
    renderStackList();  
    update();  
  });

  tabButtons.forEach(btn \=\> {  
    btn.addEventListener('click', () \=\> {  
      tabButtons.forEach(b \=\> b.classList.remove('active'));  
      btn.classList.add('active');  
      activeTab \= btn.getAttribute('data-tab');  
      renderVendorConfig(getConfiguration());  
    });  
  });

  btnCopyHex.addEventListener('click', () \=\> {  
    navigator.clipboard.writeText(hexViewer.textContent).then(() \=\> {  
      btnCopyHex.textContent \= 'Copied\!';  
      setTimeout(() \=\> { btnCopyHex.textContent \= 'Copy Hex'; }, 1500);  
    });  
  });

  btnCopyConfig.addEventListener('click', () \=\> {  
    navigator.clipboard.writeText(configViewer.textContent).then(() \=\> {  
      btnCopyConfig.textContent \= 'Copied\!';  
      setTimeout(() \=\> { btnCopyConfig.textContent \= 'Copy Config'; }, 1500);  
    });  
  });

  // Initial load  
  loadPreset('telecom-tier1-isp-core');  
});  
