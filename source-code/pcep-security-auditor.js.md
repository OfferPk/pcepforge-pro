/\*\*  
 \* PCEPForge Pro \- 10-Point Zero-Trust PCEP Security & Resilience Auditor  
 \* Standards: RFC 5440, RFC 8231, RFC 8281, RFC 8664, RFC 8253 (TLS)  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.PCEPSecurityAuditor \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {

  /\*\*  
   \* Audits a PCEP / Stateful PCE design configuration against the 10-point Zero-Trust rubric  
   \* @param {Object} cfg \- Configuration parameters  
   \* @returns {Object} { score, maxScore: 100, percentage, grade, summary, findings }  
   \*/  
  function auditPCEPConfiguration(cfg) {  
    const findings \= \[\];  
    let totalScore \= 0;

    const authType \= (cfg.authType || 'TLS').toUpperCase();  
    const tlsEnabled \= authType \=== 'TLS' || Boolean(cfg.tlsEnabled);  
    const md5Key \= (cfg.md5Key || '').trim();  
    const isDelegated \= cfg.delegated \!== undefined ? Boolean(cfg.delegated) : true;  
    const pceRedundancy \= Boolean(cfg.secondaryPceIp);  
    const keepaliveSec \= parseInt(cfg.keepaliveSec, 10\) || 30;  
    const deadTimerSec \= parseInt(cfg.deadTimerSec, 10\) || 120;  
    const srlgShared \= parseInt(cfg.srlgSharedCount, 10\) || 0;  
    const nodeShared \= parseInt(cfg.nodeSharedCount, 10\) || 0;  
    const maxSidDepth \= parseInt(cfg.maxSidDepth, 10\) || 5;  
    const localFallback \= Boolean(cfg.localFallbackEnabled);  
    const dbVersioning \= Boolean(cfg.dbVersioningEnabled);  
    const latencyConstrained \= Boolean(cfg.latencyConstrained);  
    const telemetryAlerting \= Boolean(cfg.telemetryAlerting);

    // 1\. Transport Security & Authentication (RFC 8253 TLS / MD5)  
    let trScore \= 0;  
    let trStatus \= 'PASS';  
    let trMsg \= '';  
    if (tlsEnabled) {  
      trScore \= 10;  
      trMsg \= 'TOP TIER: RFC 8253 TLS (PCEPS on TCP 4189\) active with mutual X.509 certificate authentication and encryption.';  
    } else if (authType \=== 'MD5' && md5Key.length \>= 8\) {  
      trScore \= 8;  
      trMsg \= 'TCP MD5 authentication active (RFC 5440 Sec 10.2). Protects against spoofed TCP resets, though payload is unencrypted.';  
    } else {  
      trScore \= 0;  
      trStatus \= 'CRITICAL';  
      trMsg \= 'CRITICAL VULNERABILITY: Cleartext unauthenticated TCP 4189\! An attacker can inject rogue PCUpd/PCInit frames or tear down core backbones\!';  
    }  
    findings.push({  
      id: 'TRANSPORT\_SECURITY\_AND\_AUTH',  
      name: 'Transport Security & Mutual Authentication (RFC 8253 TLS / MD5)',  
      score: trScore,  
      maxScore: 10,  
      status: trStatus,  
      description: trMsg,  
      remediation: trScore \< 10 ? 'Enable RFC 8253 TLS (PCEPS) or configure strong TCP-AO/MD5 keys.' : null  
    });  
    totalScore \+= trScore;

    // 2\. Stateful LSP Delegation Control  
    let delScore \= 0;  
    let delStatus \= 'PASS';  
    let delMsg \= '';  
    if (isDelegated && cfg.primaryPceIp) {  
      delScore \= 10;  
      delMsg \= \`Stateful LSP delegation active (D-bit=1) bounded to verified primary controller (\${cfg.primaryPceIp}).\`;  
    } else if (\!isDelegated) {  
      delScore \= 6;  
      delStatus \= 'WARN';  
      delMsg \= 'Stateless mode: LSPs are not delegated. Centralized SDN controller cannot dynamically steer traffic around congestion.';  
    } else {  
      delScore \= 0;  
      delStatus \= 'CRITICAL';  
      delMsg \= 'Unvalidated delegation configuration.';  
    }  
    findings.push({  
      id: 'STATEFUL\_DELEGATION\_CONTROL',  
      name: 'Stateful LSP Delegation & Authority Policy',  
      score: delScore,  
      maxScore: 10,  
      status: delStatus,  
      description: delMsg,  
      remediation: delScore \< 10 ? 'Configure stateful delegation with explicit PCE address binding.' : null  
    });  
    totalScore \+= delScore;

    // 3\. Disjoint Path Diversity & SRLG Isolation  
    let disjScore \= 0;  
    let disjStatus \= 'PASS';  
    let disjMsg \= '';  
    if (nodeShared \=== 0 && srlgShared \=== 0\) {  
      disjScore \= 10;  
      disjMsg \= 'STRICT ZERO-TRUST DISJOINTNESS: Primary and Secondary LSPs share zero nodes and zero optical SRLG conduits.';  
    } else if (nodeShared \=== 0 && srlgShared \> 0\) {  
      disjScore \= 4;  
      disjStatus \= 'WARN';  
      disjMsg \= \`SRLG COLLISION: \${srlgShared} shared optical conduits\! A single backhoe fiber cut will collapse both primary and secondary LSPs\!\`;  
    } else {  
      disjScore \= 0;  
      disjStatus \= 'CRITICAL';  
      disjMsg \= \`CRITICAL NODE COLLISION: \${nodeShared} shared transit routers\! A hardware crash takes down all paths simultaneously\!\`;  
    }  
    findings.push({  
      id: 'DISJOINT\_PATH\_DIVERSITY\_AND\_SRLG',  
      name: 'Disjoint Path Diversity & SRLG Conduit Isolation',  
      score: disjScore,  
      maxScore: 10,  
      status: disjStatus,  
      description: disjMsg,  
      remediation: disjScore \< 10 ? 'Enforce PCEP Disjoint Association Object (RFC 8800\) with strict SRLG and Node exclusion.' : null  
    });  
    totalScore \+= disjScore;

    // 4\. SR-ERO Validation & Maximum SID Depth (MSD)  
    let msdScore \= 0;  
    let msdStatus \= 'PASS';  
    let msdMsg \= '';  
    if (maxSidDepth \<= 6\) {  
      msdScore \= 10;  
      msdMsg \= \`Optimal Maximum SID Depth (MSD=\${maxSidDepth}). Guaranteed full line-rate imposition across all commercial router ASICs.\`;  
    } else if (maxSidDepth \<= 10\) {  
      msdScore \= 8;  
      msdMsg \= \`Moderate SID Depth (MSD=\${maxSidDepth}). Supported on modern Jericho2/Tomahawk3, but verify legacy line cards.\`;  
    } else {  
      msdScore \= 2;  
      msdStatus \= 'CRITICAL';  
      msdMsg \= \`EXCESSIVE SID DEPTH (MSD=\${maxSidDepth}): Exceeds line card hardware label imposition limit\! Traffic will experience silent drops or CPU recirculation.\`;  
    }  
    findings.push({  
      id: 'SR\_ERO\_VALIDATION\_AND\_SID\_DEPTH',  
      name: 'Segment Routing ERO Validation & MSD Imposition Bounds',  
      score: msdScore,  
      maxScore: 10,  
      status: msdStatus,  
      description: msdMsg,  
      remediation: msdScore \< 10 ? 'Limit Segment Routing ERO hops to MSD \<= 6 or enable Node SID compression.' : null  
    });  
    totalScore \+= msdScore;

    // 5\. Timer Stability & DeadTimer Ratio  
    let tmScore \= 0;  
    let tmStatus \= 'PASS';  
    let tmMsg \= '';  
    const ratio \= deadTimerSec / keepaliveSec;  
    if (keepaliveSec \>= 15 && ratio \>= 3.5 && ratio \<= 4.5) {  
      tmScore \= 10;  
      tmMsg \= \`STABLE TIMERS: Keepalive \${keepaliveSec}s, DeadTimer \${deadTimerSec}s (4:1 ratio). Fully resilient against control plane scheduling jitter.\`;  
    } else if (keepaliveSec \< 5\) {  
      tmScore \= 0;  
      tmStatus \= 'CRITICAL';  
      tmMsg \= \`CRITICAL FLAP RISK: Keepalive is only \${keepaliveSec}s\! Normal BGP/OSPF table calculation spikes will cause false PCEP session drops\!\`;  
    } else {  
      tmScore \= 5;  
      tmStatus \= 'WARN';  
      tmMsg \= \`Suboptimal timer ratio (\${keepaliveSec}s / \${deadTimerSec}s). Standard RFC 5440 recommends DeadTimer \= 4 \* Keepalive.\`;  
    }  
    findings.push({  
      id: 'TIMER\_STABILITY\_AND\_DEADTIMER\_RATIO',  
      name: 'PCEP Session Keepalive & DeadTimer Stability',  
      score: tmScore,  
      maxScore: 10,  
      status: tmStatus,  
      description: tmMsg,  
      remediation: tmScore \< 10 ? 'Set Keepalive to 30s and DeadTimer to 120s (4:1 ratio).' : null  
    });  
    totalScore \+= tmScore;

    // 6\. Fail-Closed Local Fallback Protection  
    let fbScore \= 0;  
    let fbStatus \= 'PASS';  
    let fbMsg \= '';  
    if (localFallback) {  
      fbScore \= 10;  
      fbMsg \= 'RESILIENT FAILOVER: Local TI-LFA / pre-computed fallback LSP active. If PCE disconnects, traffic continues flowing over local IGP paths.';  
    } else {  
      fbScore \= 0;  
      fbStatus \= 'CRITICAL';  
      fbMsg \= 'TRAFFIC BLACKHOLE HAZARD: No local fallback configured\! If the centralized PCE controller drops, all delegated LSPs are immediately torn down\!';  
    }  
    findings.push({  
      id: 'FAIL\_CLOSED\_LOCAL\_PROTECTION',  
      name: 'Fail-Closed Local TI-LFA & Fallback LSP Protection',  
      score: fbScore,  
      maxScore: 10,  
      status: fbStatus,  
      description: fbMsg,  
      remediation: fbScore \< 10 ? 'Enable local fallback LSP / TI-LFA retention on PCE disconnection.' : null  
    });  
    totalScore \+= fbScore;

    // 7\. PCE Redundancy & High Availability  
    let haScore \= 0;  
    let haStatus \= 'PASS';  
    let haMsg \= '';  
    if (pceRedundancy) {  
      haScore \= 10;  
      haMsg \= \`High Availability configured: Primary PCE (\${cfg.primaryPceIp}) and Secondary Standby PCE (\${cfg.secondaryPceIp}).\`;  
    } else {  
      haScore \= 2;  
      haStatus \= 'CRITICAL';  
      haMsg \= 'SINGLE POINT OF FAILURE: Only a single PCE controller configured. A controller reboot or maintenance event freezes traffic engineering.';  
    }  
    findings.push({  
      id: 'PCE\_REDUNDANCY\_AND\_HIGH\_AVAILABILITY',  
      name: 'PCE Controller Redundancy & Active/Standby HA',  
      score: haScore,  
      maxScore: 10,  
      status: haStatus,  
      description: haMsg,  
      remediation: haScore \< 10 ? 'Configure dual geo-redundant PCE controllers.' : null  
    });  
    totalScore \+= haScore;

    // 8\. State Synchronization & DB Versioning (RFC 8232\)  
    let syncScore \= 0;  
    let syncStatus \= 'PASS';  
    let syncMsg \= '';  
    if (dbVersioning) {  
      syncScore \= 10;  
      syncMsg \= 'Incremental synchronization enabled (DB-VERSION-TLV active). Prevents catastrophic full resynchronization floods upon reconnection.';  
    } else {  
      syncScore \= 5;  
      syncStatus \= 'WARN';  
      syncMsg \= 'Incremental sync disabled. Router performs full LSP-DB dump on every reconnect, spiking control plane CPU on core routers.';  
    }  
    findings.push({  
      id: 'STATE\_SYNC\_AND\_DB\_VERSIONING',  
      name: 'State Synchronization & Incremental DB Versioning (RFC 8232)',  
      score: syncScore,  
      maxScore: 10,  
      status: syncStatus,  
      description: syncMsg,  
      remediation: syncScore \< 10 ? 'Enable incremental state synchronization and DB versioning.' : null  
    });  
    totalScore \+= syncScore;

    // 9\. Latency & Bandwidth Constraint Enforcement  
    let conScore \= 0;  
    let conStatus \= 'PASS';  
    let conMsg \= '';  
    if (latencyConstrained) {  
      conScore \= 10;  
      conMsg \= 'Metric Type 11 (Accumulated Delay / Latency Bound) and Bandwidth admission control strictly enforced.';  
    } else {  
      conScore \= 5;  
      conStatus \= 'WARN';  
      conMsg \= 'Unconstrained path computation: Latency SLA bounds not enforced in PCEP METRIC object.';  
    }  
    findings.push({  
      id: 'BANDWIDTH\_AND\_LATENCY\_CONSTRAINTS',  
      name: 'Bandwidth & Microsecond Latency Constraint Enforcement',  
      score: conScore,  
      maxScore: 10,  
      status: conStatus,  
      description: conMsg,  
      remediation: conScore \< 10 ? 'Enforce Metric Type 11 (Latency) and Bandwidth admission controls.' : null  
    });  
    totalScore \+= conScore;

    // 10\. Telemetry & Prometheus Observability  
    let telScore \= 0;  
    let telStatus \= 'PASS';  
    let telMsg \= '';  
    if (telemetryAlerting) {  
      telScore \= 10;  
      telMsg \= 'PCEP session telemetry active: PCReq/PCRep latency, PCErr counters, and delegation state exported to Prometheus/SIEM.';  
    } else {  
      telScore \= 4;  
      telStatus \= 'WARN';  
      telMsg \= 'PCEP telemetry disabled: Controller synchronization failures and path computation rejections will go unnoticed.';  
    }  
    findings.push({  
      id: 'TELEMETRY\_AND\_PROMETHEUS\_OBSERVABILITY',  
      name: 'PCEP Telemetry & Prometheus Anomaly Alerting',  
      score: telScore,  
      maxScore: 10,  
      status: telStatus,  
      description: telMsg,  
      remediation: telScore \< 10 ? 'Export PCEP telemetry metrics (session state, computation latency, error codes) to Prometheus.' : null  
    });  
    totalScore \+= telScore;

    const percentage \= Math.round((totalScore / 100\) \* 100);  
    let grade \= 'F';  
    let summary \= '';

    if (percentage \>= 95\) {  
      grade \= 'A+';  
      summary \= 'Tier-1 Carrier Zero-Trust PCEP Architecture: Full TLS mutual auth, geo-redundancy, strict SRLG diversity, and sub-50ms TI-LFA.';  
    } else if (percentage \>= 85\) {  
      grade \= 'A';  
      summary \= 'Hardened Production PCEP: Robust stateful traffic engineering with minor recommended tuning.';  
    } else if (percentage \>= 70\) {  
      grade \= 'B';  
      summary \= 'Standard Enterprise PCEP: Functional centralized path steering, but exposed to SRLG or redundancy risks.';  
    } else if (percentage \>= 60\) {  
      grade \= 'C';  
      summary \= 'Suboptimal PCEP Deployment: Significant resilience or timer risks detected.';  
    } else if (percentage \>= 50\) {  
      grade \= 'D';  
      summary \= 'Weak PCEP Security: High flap risk and inadequate transport protection.';  
    } else {  
      grade \= 'F';  
      summary \= 'CRITICAL PCEP VULNERABILITY: Severe operational hazards (unauthenticated plaintext, SRLG/node collision, or traffic blackhole).';  
    }

    return {  
      score: totalScore,  
      maxScore: 100,  
      percentage: percentage,  
      grade: grade,  
      summary: summary,  
      findings: findings  
    };  
  }

  return {  
    auditPCEPConfiguration: auditPCEPConfiguration  
  };  
});  
