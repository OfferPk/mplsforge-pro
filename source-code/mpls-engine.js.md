/\*\*  
 \* MPLSForge Pro \- RFC 3031 / RFC 3032 Multiprotocol Label Switching (MPLS) Engine  
 \* Encodes 32-bit Label Stack Entries (LSE), handles multi-label push/swap/pop,  
 \* computes MTU/payload overhead budgets, and generates Wireshark-style hex dumps.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.MPLSEngine \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {

  // Reserved MPLS Labels per IANA & RFC 3032 / RFC 6790  
  const RESERVED\_LABELS \= {  
    0: { name: 'IPv4 Explicit Null', rfc: 'RFC 3032', purpose: 'Preserves QoS TC bits to egress PE; pops top label but router examines IPv4 header' },  
    1: { name: 'Router Alert', rfc: 'RFC 3032', purpose: 'Directs packet to local control plane for processing (OAM/RSVP)' },  
    2: { name: 'IPv6 Explicit Null', rfc: 'RFC 3032', purpose: 'Preserves QoS TC bits for 6PE/6VPE over IPv4 core' },  
    3: { name: 'Implicit Null', rfc: 'RFC 3032', purpose: 'PHP (Penultimate Hop Popping); signaled by egress PE, popped before egress, never on wire' },  
    7: { name: 'Entropy Label Indicator (ELI)', rfc: 'RFC 6790', purpose: 'Signifies that the next label in stack is an Entropy Label for ECMP load balancing' },  
    14: { name: 'OAM Alert Label', rfc: 'RFC 3429', purpose: 'Associated Channel Header (ACH) for MPLS-TP OAM frames' },  
    15: { name: 'Extension Label', rfc: 'RFC 7274', purpose: 'Allows future reserved label space extension' }  
  };

  /\*\*  
   \* Encodes a single 32-bit MPLS Label Stack Entry (LSE) per RFC 3032  
   \* Bit 0-19: Label (20 bits)  
   \* Bit 20-22: Traffic Class (TC / EXP) (3 bits)  
   \* Bit 23: Bottom of Stack (S) (1 bit)  
   \* Bit 24-31: Time to Live (TTL) (8 bits)  
   \*  
   \* @param {Object} entry \- { label: number, tc: number, s: boolean|number, ttl: number }  
   \* @returns {Uint8Array} 4-byte buffer  
   \*/  
  function encodeLSE(entry) {  
    const label \= Math.max(0, Math.min(1048575, parseInt(entry.label, 10\) || 0));  
    const tc \= Math.max(0, Math.min(7, parseInt(entry.tc, 10\) || 0));  
    const s \= entry.s ? 1 : 0;  
    const ttl \= Math.max(1, Math.min(255, parseInt(entry.ttl, 10\) || 255));

    const bytes \= new Uint8Array(4);  
    bytes\[0\] \= (label \>\> 12\) & 0xFF;  
    bytes\[1\] \= (label \>\> 4\) & 0xFF;  
    bytes\[2\] \= ((label & 0x0F) \<\< 4\) | ((tc & 0x07) \<\< 1\) | (s & 0x01);  
    bytes\[3\] \= ttl & 0xFF;

    return bytes;  
  }

  /\*\*  
   \* Decodes a 4-byte Uint8Array into an LSE object  
   \* @param {Uint8Array} bytes  
   \* @param {number} offset  
   \* @returns {Object}  
   \*/  
  function decodeLSE(bytes, offset \= 0\) {  
    if (bytes.length \< offset \+ 4\) {  
      throw new Error(\`Buffer underrun: needed 4 bytes, got \${bytes.length \- offset}\`);  
    }

    const b0 \= bytes\[offset\];  
    const b1 \= bytes\[offset \+ 1\];  
    const b2 \= bytes\[offset \+ 2\];  
    const b3 \= bytes\[offset \+ 3\];

    const label \= (b0 \<\< 12\) | (b1 \<\< 4\) | ((b2 \>\> 4\) & 0x0F);  
    const tc \= (b2 \>\> 1\) & 0x07;  
    const s \= b2 & 0x01;  
    const ttl \= b3;

    return {  
      label,  
      tc,  
      s: s \=== 1,  
      ttl,  
      isReserved: label in RESERVED\_LABELS,  
      reservedMeta: RESERVED\_LABELS\[label\] || null  
    };  
  }

  /\*\*  
   \* Encodes an entire MPLS Label Stack with optional sample IP payload  
   \* @param {Array\<Object\>} stack \- array of LSE descriptors from top (outermost) to bottom (innermost)  
   \* @param {Object} options \- { vlan: number, addDummyPayload: boolean, payloadType: 'ipv4'|'ipv6' }  
   \* @returns {Object} { bytes: Uint8Array, hexDump: string, breakdown: Array }  
   \*/  
  function encodeMPLSFrame(stack, options \= {}) {  
    if (\!Array.isArray(stack) || stack.length \=== 0\) {  
      throw new Error('Label stack must contain at least one Label Stack Entry');  
    }

    const vlanCount \= options.vlanCount || (options.vlan ? 1 : 0);  
    const l2HeaderLen \= 14 \+ (vlanCount \* 4); // DstMAC(6) \+ SrcMAC(6) \+ EtherType(2) \+ VLAN(4 each)  
    const mplsShimLen \= stack.length \* 4;  
    const dummyPayloadLen \= options.addDummyPayload ? (options.payloadType \=== 'ipv6' ? 40 : 20\) : 0;  
    const totalLen \= l2HeaderLen \+ mplsShimLen \+ dummyPayloadLen;

    const buffer \= new Uint8Array(totalLen);  
    let pos \= 0;

    // 1\. Layer 2 Ethernet Header  
    // Dst MAC: 00:00:5E:00:53:01 (VRRP / sample router MAC)  
    buffer.set(\[0x00, 0x00, 0x5E, 0x00, 0x53, 0x01\], pos);  
    pos \+= 6;  
    // Src MAC: 00:1A:2B:3C:4D:5E  
    buffer.set(\[0x00, 0x1A, 0x2B, 0x3C, 0x4D, 0x5E\], pos);  
    pos \+= 6;

    // Optional 802.1Q VLAN Tag (EtherType 0x8100)  
    if (vlanCount \> 0\) {  
      const vlanId \= options.vlan || 100;  
      buffer\[pos++\] \= 0x81;  
      buffer\[pos++\] \= 0x00;  
      buffer\[pos++\] \= (vlanId \>\> 8\) & 0x0F;  
      buffer\[pos++\] \= vlanId & 0xFF;  
    }

    // MPLS EtherType: 0x8847 (MPLS Unicast) or 0x8848 (MPLS Multicast)  
    const etherType \= options.isMulticast ? 0x8848 : 0x8847;  
    buffer\[pos++\] \= (etherType \>\> 8\) & 0xFF;  
    buffer\[pos++\] \= etherType & 0xFF;

    // 2\. MPLS Label Stack  
    const breakdown \= \[\];  
    for (let i \= 0; i \< stack.length; i++) {  
      const isBottom \= (i \=== stack.length \- 1);  
      const entry \= {  
        label: stack\[i\].label,  
        tc: stack\[i\].tc \!== undefined ? stack\[i\].tc : 0,  
        s: isBottom ? 1 : 0,  
        ttl: stack\[i\].ttl \!== undefined ? stack\[i\].ttl : 255,  
        role: stack\[i\].role || (i \=== 0 ? 'Transport / IGP' : (isBottom ? 'VPN / Service' : 'Inner / Entropy'))  
      };

      const lseBytes \= encodeLSE(entry);  
      buffer.set(lseBytes, pos);

      const hexWord \= '0x' \+ Array.from(lseBytes).map(b \=\> b.toString(16).padStart(2, '0')).join('');  
      breakdown.push({  
        position: i \+ 1,  
        depth: stack.length \- i,  
        role: entry.role,  
        label: entry.label,  
        isReserved: entry.label in RESERVED\_LABELS,  
        reservedName: RESERVED\_LABELS\[entry.label\]?.name || null,  
        tc: entry.tc,  
        s: entry.s,  
        ttl: entry.ttl,  
        hex: hexWord,  
        offset: pos  
      });

      pos \+= 4;  
    }

    // 3\. Optional Dummy Payload Header  
    if (options.addDummyPayload) {  
      if (options.payloadType \=== 'ipv6') {  
        // Sample IPv6 Header (40 bytes)  
        buffer\[pos\] \= 0x60; // Version 6, Traffic Class 0  
        buffer\[pos \+ 7\] \= 0x3B; // Next Header: None  
        buffer\[pos \+ 8\] \= 0x40; // Hop Limit: 64  
        // Src / Dst IPv6 addresses  
        pos \+= 40;  
      } else {  
        // Sample IPv4 Header (20 bytes)  
        buffer\[pos\] \= 0x45; // Version 4, IHL 5 (20 bytes)  
        buffer\[pos \+ 1\] \= 0x00; // DSCP 0  
        buffer\[pos \+ 2\] \= 0x00; // Total Length (e.g. 20\)  
        buffer\[pos \+ 3\] \= 0x14;  
        buffer\[pos \+ 8\] \= 0x40; // TTL: 64  
        buffer\[pos \+ 9\] \= 0x06; // Protocol: TCP  
        // Src IP: 192.0.2.1  
        buffer.set(\[0xC0, 0x00, 0x02, 0x01\], pos \+ 12);  
        // Dst IP: 198.51.100.1  
        buffer.set(\[0xC6, 0x33, 0x64, 0x01\], pos \+ 16);  
        pos \+= 20;  
      }  
    }

    const hexDump \= formatWiresharkHex(buffer);

    return {  
      bytes: buffer,  
      totalLength: buffer.length,  
      l2HeaderLength: l2HeaderLen,  
      mplsShimLength: mplsShimLen,  
      labelCount: stack.length,  
      breakdown,  
      hexDump  
    };  
  }

  /\*\*  
   \* Formats a Uint8Array into a Wireshark-compatible hex \+ ASCII dump  
   \* Format:  
   \* 0000   00 00 5e 00 53 01 00 1a  2b 3c 4d 5e 88 47 00 01   ..^.S... \+\<M^.G..  
   \* @param {Uint8Array} buffer  
   \* @returns {string}  
   \*/  
  function formatWiresharkHex(buffer) {  
    const lines \= \[\];  
    for (let i \= 0; i \< buffer.length; i \+= 16\) {  
      const offsetHex \= i.toString(16).padStart(4, '0');  
      const chunk \= buffer.slice(i, i \+ 16);

      const hexBytes \= \[\];  
      let ascii \= '';

      for (let j \= 0; j \< 16; j++) {  
        if (j \< chunk.length) {  
          const byte \= chunk\[j\];  
          hexBytes.push(byte.toString(16).padStart(2, '0'));  
          ascii \+= (byte \>= 32 && byte \<= 126\) ? String.fromCharCode(byte) : '.';  
        } else {  
          hexBytes.push('  ');  
          ascii \+= ' ';  
        }  
      }

      // Add space between 8-byte blocks  
      const leftBlock \= hexBytes.slice(0, 8).join(' ');  
      const rightBlock \= hexBytes.slice(8, 16).join(' ');  
      lines.push(\`\${offsetHex}   \${leftBlock}  \${rightBlock}   \${ascii}\`);  
    }  
    return lines.join('\\n');  
  }

  /\*\*  
   \* Calculates MTU overhead, path margins, and detects MTU blackhole vulnerabilities  
   \*  
   \* @param {Object} params  
   \*   \- labelCount: number (depth of stack, each 4 bytes)  
   \*   \- payloadMtu: number (e.g. 1500 or 9000\)  
   \*   \- vlanTags: number (0, 1=802.1Q 4B, 2=QinQ 8B)  
   \*   \- interfaceMtu: number (Physical L2 interface MTU configured on core routers)  
   \*   \- pwe3ControlWord: boolean (Pseudowire 4-byte Control Word RFC 4385\)  
   \* @returns {Object}  
   \*/  
  function calculateMTUBudget(params) {  
    const labelCount \= Math.max(1, parseInt(params.labelCount, 10\) || 1);  
    const payloadMtu \= Math.max(576, parseInt(params.payloadMtu, 10\) || 1500);  
    const vlanTags \= Math.max(0, parseInt(params.vlanTags, 10\) || 0);  
    const interfaceMtu \= Math.max(1500, parseInt(params.interfaceMtu, 10\) || 1500);  
    const controlWord \= \!\!params.pwe3ControlWord;

    const baseEthernetOverhead \= 14; // Dst(6)+Src(6)+Type(2)  
    const vlanOverhead \= vlanTags \* 4;  
    const mplsOverhead \= labelCount \* 4;  
    const controlWordOverhead \= controlWord ? 4 : 0;

    // Minimum L3 IP MTU traversing the MPLS LSP  
    const totalMplsPayloadMtu \= payloadMtu \+ controlWordOverhead;  
    // Total L3 MPLS packet size (labels \+ IP)  
    const totalMplsPacketSize \= totalMplsPayloadMtu \+ mplsOverhead;  
    // Total L2 frame size on the wire (excluding 4-byte FCS and 8-byte preamble)  
    const totalL2FrameSize \= totalMplsPacketSize \+ baseEthernetOverhead \+ vlanOverhead;

    // Margin between configured interface MTU and required frame size  
    // Note: Vendors differ in whether 'interface mtu' denotes IP MTU or L2 frame size.  
    // Standard Carrier definition: interface MTU must be \>= totalMplsPacketSize (or totalL2FrameSize).  
    const headroomL3 \= interfaceMtu \- totalMplsPacketSize;  
    const headroomL2 \= interfaceMtu \- totalL2FrameSize;

    const isBlackholeRisk \= headroomL3 \< 0;  
    const isMarginal \= headroomL3 \>= 0 && headroomL3 \< 20;

    let status \= 'HEALTHY';  
    let recommendations \= \[\];

    if (isBlackholeRisk) {  
      status \= 'CRITICAL\_BLACKHOLE';  
      recommendations.push(  
        \`Interface MTU (\${interfaceMtu}B) is smaller than required MPLS frame size (\${totalMplsPacketSize}B). Packets exceeding \${interfaceMtu \- mplsOverhead}B will be silently dropped in the core without ICMP notification.\`  
      );  
      recommendations.push(  
        \`Increase core physical interface MTU to at least \${totalL2FrameSize}B (recommended standard jumbo: 9192B or 9216B).\`  
      );  
    } else if (isMarginal) {  
      status \= 'WARNING\_LOW\_HEADROOM';  
      recommendations.push(  
        \`Headroom is only \${headroomL3} bytes. Adding additional labels (e.g. Fast Reroute bypass, Segment Routing SID list, or Entropy Label) will cause immediate packet drops.\`  
      );  
      recommendations.push(\`Standardize on 9216B Jumbo MTU across all PE and P core links.\`);  
    } else {  
      recommendations.push(\`Generous MTU headroom (\${headroomL3}B). Core links can absorb deep SR-MPLS segment lists and EVPN labels.\`);  
    }

    return {  
      labelCount,  
      payloadMtu,  
      vlanTags,  
      interfaceMtu,  
      controlWord,  
      baseEthernetOverhead,  
      vlanOverhead,  
      mplsOverhead,  
      controlWordOverhead,  
      totalMplsPacketSize,  
      totalL2FrameSize,  
      headroomL3,  
      headroomL2,  
      status,  
      recommendations  
    };  
  }

  return {  
    RESERVED\_LABELS,  
    encodeLSE,  
    decodeLSE,  
    encodeMPLSFrame,  
    formatWiresharkHex,  
    calculateMTUBudget  
  };  
});  
