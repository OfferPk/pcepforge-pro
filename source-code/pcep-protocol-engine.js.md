/\*\*  
 \* PCEPForge Pro \- Path Computation Element Communication Protocol (PCEP) Engine  
 \* Standards: RFC 5440, RFC 8231 (Stateful PCE), RFC 8281 (PCE-Initiated LSPs), RFC 8664 (Segment Routing)  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.PCEPProtocolEngine \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {

  const MESSAGE\_TYPES \= {  
    1: 'PCReq (Path Computation Request)',  
    2: 'PCRep (Path Computation Reply)',  
    3: 'PCNtf (Notification)',  
    4: 'PCErr (Error)',  
    5: 'Close',  
    7: 'Open',  
    8: 'Keepalive',  
    10: 'PCRpt (Path Computation Report)',  
    11: 'PCUpd (Path Computation Update)',  
    12: 'PCInit (Path Computation Initiate)'  
  };

  const OBJECT\_CLASSES \= {  
    1: 'OPEN',  
    2: 'RP',  
    3: 'NO-PATH',  
    4: 'END-POINTS',  
    5: 'BANDWIDTH',  
    6: 'METRIC',  
    7: 'ERO',  
    8: 'RRO',  
    9: 'LSPA',  
    32: 'LSP',  
    33: 'SRP',  
    40: 'ASSOCIATION'  
  };

  /\*\*  
   \* Encodes a PCEP Common Message Header (4 octets)  
   \*/  
  function encodeMessageHeader(msgType, totalLength) {  
    const buf \= new Uint8Array(4);  
    buf\[0\] \= 0x20; // Version 1 (bits 0-2: 001, flags: 00000\)  
    buf\[1\] \= msgType & 0xFF;  
    buf\[2\] \= (totalLength \>\> 8\) & 0xFF;  
    buf\[3\] \= totalLength & 0xFF;  
    return buf;  
  }

  /\*\*  
   \* Encodes an SRP (Stateful Request Parameters) Object (RFC 8231, 12 octets)  
   \*/  
  function encodeSRPObject(srpId, isRemove) {  
    const buf \= new Uint8Array(12);  
    // Common Object Header: Class 33 (SRP), Type 1, Length 12  
    buf\[0\] \= 33;  
    buf\[1\] \= (1 \<\< 4); // OT=1, P=0, I=0  
    buf\[2\] \= 0;  
    buf\[3\] \= 12; // Length

    // Flags: bit 31 is R (Remove)  
    buf\[4\] \= 0;  
    buf\[5\] \= 0;  
    buf\[6\] \= 0;  
    buf\[7\] \= isRemove ? 0x01 : 0x00;

    // SRP-ID Number (32-bit big endian)  
    buf\[8\] \= (srpId \>\>\> 24\) & 0xFF;  
    buf\[9\] \= (srpId \>\>\> 16\) & 0xFF;  
    buf\[10\] \= (srpId \>\>\> 8\) & 0xFF;  
    buf\[11\] \= srpId & 0xFF;

    return buf;  
  }

  /\*\*  
   \* Encodes an LSP Object (RFC 8231 / RFC 8281, 8 octets)  
   \* Flags: D (Delegate), S (Sync), R (Remove), A (Admin), O (Operational: 0-7), C (Create)  
   \*/  
  function encodeLSPObject(plspId, delegate, sync, operationalState, pceCreated) {  
    const buf \= new Uint8Array(8);  
    // Class 32 (LSP), Type 1, Length 8  
    buf\[0\] \= 32;  
    buf\[1\] \= (1 \<\< 4);  
    buf\[2\] \= 0;  
    buf\[3\] \= 8;

    // PLSP-ID (20 bits) \+ Flags (12 bits)  
    const pid \= (plspId || 1\) & 0x0FFFFF;  
    buf\[4\] \= (pid \>\>\> 12\) & 0xFF;  
    buf\[5\] \= (pid \>\>\> 4\) & 0xFF;

    // Remaining 4 bits of PLSP-ID \+ top 4 bits of Flags (D, S, R, A)  
    const dBit \= delegate ? 1 : 0;  
    const sBit \= sync ? 1 : 0;  
    const rBit \= 0;  
    const aBit \= 1; // Admin UP  
    buf\[6\] \= ((pid & 0x0F) \<\< 4\) | (dBit \<\< 3\) | (sBit \<\< 2\) | (rBit \<\< 1\) | aBit;

    // Operational (3 bits: 1=UP, 0=DOWN) \+ C-bit (1=PCE created)  
    const op \= (operationalState \!== undefined ? operationalState : 1\) & 0x07;  
    const cBit \= pceCreated ? 1 : 0;  
    buf\[7\] \= (op \<\< 5\) | (cBit \<\< 4);

    return buf;  
  }

  /\*\*  
   \* Encodes an SR-ERO Object (RFC 8664\) with Segment Routing Hops  
   \*/  
  function encodeSREROObject(hops) {  
    const hopList \= hops || \[  
      { sid: 16001, loose: false, naiType: 1, ip: '198.51.100.1' },  
      { sid: 16002, loose: false, naiType: 1, ip: '198.51.100.2' },  
      { sid: 16003, loose: false, naiType: 1, ip: '198.51.100.3' }  
    \];

    // Each SR-ERO Subobject is 12 octets:  
    // L (1b) \+ Type 36 (7b) \= 1B, Length 12 \= 1B, NT (4b) \+ Flags (12b: M=1, C=0, S=1, F=0) \= 2B  
    // SID (4B) \= 4B, IPv4 Node NAI (4B) \= 4B. Total \= 12 bytes per hop.  
    const subobjLen \= hopList.length \* 12;  
    const totalLen \= 4 \+ subobjLen;  
    const buf \= new Uint8Array(totalLen);

    // ERO Object Header: Class 7, Type 1  
    buf\[0\] \= 7;  
    buf\[1\] \= (1 \<\< 4);  
    buf\[2\] \= (totalLen \>\> 8\) & 0xFF;  
    buf\[3\] \= totalLen & 0xFF;

    let offset \= 4;  
    hopList.forEach(hop \=\> {  
      // Byte 0: L-bit (0x80) \+ Type 36 (0x24)  
      buf\[offset\] \= (hop.loose ? 0x80 : 0x00) | 36;  
      buf\[offset \+ 1\] \= 12; // Length

      // Byte 2-3: NAI Type (NT: 4 bits) \+ Flags (12 bits: M=1 for MPLS, S=1 for SID specified)  
      const nt \= (hop.naiType || 1\) & 0x0F;  
      buf\[offset \+ 2\] \= (nt \<\< 4\) | 0x01; // M-bit set  
      buf\[offset \+ 3\] \= 0x08; // S-bit set

      // Bytes 4-7: 32-bit SID / MPLS Label (e.g. 16001\)  
      const sid \= (hop.sid || 16001\) \>\>\> 0;  
      buf\[offset \+ 4\] \= (sid \>\>\> 24\) & 0xFF;  
      buf\[offset \+ 5\] \= (sid \>\>\> 16\) & 0xFF;  
      buf\[offset \+ 6\] \= (sid \>\>\> 8\) & 0xFF;  
      buf\[offset \+ 7\] \= sid & 0xFF;

      // Bytes 8-11: NAI IPv4 address  
      const ipParts \= (hop.ip || '10.0.0.1').split('.').map(p \=\> parseInt(p, 10\) || 0);  
      buf\[offset \+ 8\] \= ipParts\[0\];  
      buf\[offset \+ 9\] \= ipParts\[1\];  
      buf\[offset \+ 10\] \= ipParts\[2\];  
      buf\[offset \+ 11\] \= ipParts\[3\];

      offset \+= 12;  
    });

    return buf;  
  }

  /\*\*  
   \* Encodes a complete PCEP PCRpt or PCUpd Message (RFC 8231\)  
   \*/  
  function encodeStatefulPCEPMessage(msgType, plspId, srpId, delegate, sync, opState, pceCreated, hops) {  
    const srpBuf \= encodeSRPObject(srpId || 1001, false);  
    const lspBuf \= encodeLSPObject(plspId || 1, delegate, sync, opState, pceCreated);  
    const eroBuf \= encodeSREROObject(hops);

    const totalLen \= 4 \+ srpBuf.length \+ lspBuf.length \+ eroBuf.length;  
    const msgBuf \= new Uint8Array(totalLen);

    // Message Header  
    const hdr \= encodeMessageHeader(msgType || 10, totalLen); // 10 \= PCRpt, 11 \= PCUpd  
    msgBuf.set(hdr, 0);

    // Objects  
    msgBuf.set(srpBuf, 4);  
    msgBuf.set(lspBuf, 4 \+ srpBuf.length);  
    msgBuf.set(eroBuf, 4 \+ srpBuf.length \+ lspBuf.length);

    return msgBuf;  
  }

  /\*\*  
   \* Calculates Path Latency, TE Metrics & SRLG Disjointness  
   \*/  
  function calculatePathMetrics(options) {  
    const opts \= options || {};  
    const primaryDistanceKm \= parseFloat(opts.primaryDistanceKm) || 1200;  
    const secondaryDistanceKm \= parseFloat(opts.secondaryDistanceKm) || 1450;  
    const bandwidthGbps \= parseFloat(opts.bandwidthGbps) || 100;  
    const srlgSharedCount \= parseInt(opts.srlgSharedCount, 10\) || 0;  
    const nodeSharedCount \= parseInt(opts.nodeSharedCount, 10\) || 0;

    // Optical propagation delay rule of thumb: \~4.9 microseconds per km (in single-mode fiber c \= 204,000 km/s)  
    const fiberLatencyUsPerKm \= 4.90;  
    const primaryLatencyUs \= primaryDistanceKm \* fiberLatencyUsPerKm;  
    const secondaryLatencyUs \= secondaryDistanceKm \* fiberLatencyUsPerKm;  
    const latencyDeltaUs \= Math.abs(secondaryLatencyUs \- primaryLatencyUs);

    let disjointStatus \= 'STRICT\_DISJOINT';  
    let riskLevel \= 'SAFE';  
    let summary \= 'Full SRLG and Node diversity between Primary and Secondary paths.';

    if (nodeSharedCount \> 0\) {  
      disjointStatus \= 'SHARED\_NODE';  
      riskLevel \= 'CRITICAL';  
      summary \= \`CRITICAL DIVERSITY HAZARD: \${nodeSharedCount} transit nodes shared between primary and secondary LSPs\! A router crash takes down both paths.\`;  
    } else if (srlgSharedCount \> 0\) {  
      disjointStatus \= 'SHARED\_SRLG';  
      riskLevel \= 'HIGH';  
      summary \= \`SRLG CONDUIT COLLISION: \${srlgSharedCount} shared optical duct/conduit segments. A single backhoe fiber cut will sever both paths\!\`;  
    }

    return {  
      primaryDistanceKm: primaryDistanceKm,  
      secondaryDistanceKm: secondaryDistanceKm,  
      primaryLatencyUs: Math.round(primaryLatencyUs),  
      primaryLatencyMs: (primaryLatencyUs / 1000).toFixed(2),  
      secondaryLatencyUs: Math.round(secondaryLatencyUs),  
      secondaryLatencyMs: (secondaryLatencyUs / 1000).toFixed(2),  
      latencyDeltaMs: (latencyDeltaUs / 1000).toFixed(2),  
      bandwidthGbps: bandwidthGbps,  
      srlgSharedCount: srlgSharedCount,  
      nodeSharedCount: nodeSharedCount,  
      disjointStatus: disjointStatus,  
      riskLevel: riskLevel,  
      summary: summary  
    };  
  }

  /\*\*  
   \* Formats binary buffer into Wireshark / hexdump format  
   \*/  
  function formatHexDump(buf) {  
    const lines \= \[\];  
    for (let i \= 0; i \< buf.length; i \+= 16\) {  
      const chunk \= buf.slice(i, i \+ 16);  
      const hexParts \= \[\];  
      const asciiParts \= \[\];  
      for (let j \= 0; j \< 16; j++) {  
        if (j \< chunk.length) {  
          const b \= chunk\[j\];  
          hexParts.push(b.toString(16).padStart(2, '0'));  
          asciiParts.push((b \>= 32 && b \<= 126\) ? String.fromCharCode(b) : '.');  
        } else {  
          hexParts.push('  ');  
          asciiParts.push(' ');  
        }  
      }  
      const left \= hexParts.slice(0, 8).join(' ');  
      const right \= hexParts.slice(8, 16).join(' ');  
      const offset \= i.toString(16).padStart(4, '0');  
      lines.push(\`\${offset}  \${left}  \${right}  |\${asciiParts.join('')}|\`);  
    }  
    return lines.join('\\n');  
  }

  return {  
    MESSAGE\_TYPES: MESSAGE\_TYPES,  
    OBJECT\_CLASSES: OBJECT\_CLASSES,  
    encodeMessageHeader: encodeMessageHeader,  
    encodeSRPObject: encodeSRPObject,  
    encodeLSPObject: encodeLSPObject,  
    encodeSREROObject: encodeSREROObject,  
    encodeStatefulPCEPMessage: encodeStatefulPCEPMessage,  
    calculatePathMetrics: calculatePathMetrics,  
    formatHexDump: formatHexDump  
  };  
});  
