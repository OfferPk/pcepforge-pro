/\*\*  
 \* PCEPForge Pro \- Web Studio Application Controller  
 \*/

document.addEventListener('DOMContentLoaded', () \=\> {  
  const PRESETS \= {  
    'tier1-sp-stateful-sr-pcep': {  
      name: "Tier-1 Service Provider 400G Backbone (Stateful PCE-Initiated SR-TE)",  
      badge: "Tier-1 SP Core",  
      description: "High-capacity carrier core running Stateful Segment Routing (SR-TE) with RFC 8253 TLS mutual authentication, geo-redundant PCE controllers, incremental DB-versioning, and strict SRLG diversity.",  
      hostname: "NYC-CR-01",  
      localIp: "198.51.100.1",  
      peerIp: "198.51.100.2",  
      primaryPceIp: "198.51.100.10",  
      secondaryPceIp: "198.51.100.11",  
      authType: "TLS",  
      tlsEnabled: true,  
      md5Key: "",  
      delegated: true,  
      pceInitiated: true,  
      keepaliveSec: 30,  
      deadTimerSec: 120,  
      srlgSharedCount: 0,  
      nodeSharedCount: 0,  
      maxSidDepth: 5,  
      localFallbackEnabled: true,  
      dbVersioningEnabled: true,  
      latencyConstrained: true,  
      telemetryAlerting: true,  
      primaryDistanceKm: 1200,  
      secondaryDistanceKm: 1450,  
      bandwidthGbps: 400,  
      plspId: 101,  
      srpId: 5001  
    },  
    'cloud-dci-disjoint-pcep': {  
      name: "Cloud Hyper-Scaler DCI Dual-Disjoint SR-TE Fabric",  
      badge: "Cloud DCI",  
      description: "Multi-datacenter interconnect running dual disjoint SR-TE LSPs with strict node and fiber duct diversity, 100G capacity reservation, TCP MD5 auth, and microsecond latency bounds.",  
      hostname: "IAD-DCI-GW-01",  
      localIp: "10.50.0.1",  
      peerIp: "10.50.0.2",  
      primaryPceIp: "10.50.100.1",  
      secondaryPceIp: "10.50.100.2",  
      authType: "MD5",  
      tlsEnabled: false,  
      md5Key: "S3cur3PcepKey99\#",  
      delegated: true,  
      pceInitiated: true,  
      keepaliveSec: 30,  
      deadTimerSec: 120,  
      srlgSharedCount: 0,  
      nodeSharedCount: 0,  
      maxSidDepth: 6,  
      localFallbackEnabled: true,  
      dbVersioningEnabled: true,  
      latencyConstrained: true,  
      telemetryAlerting: true,  
      primaryDistanceKm: 850,  
      secondaryDistanceKm: 1020,  
      bandwidthGbps: 100,  
      plspId: 202,  
      srpId: 6001  
    },  
    'vulnerable-stateless-unauth': {  
      name: "Vulnerable Stateless Unauthenticated PCEP (Flap & Blackhole Hazard)",  
      badge: "Critical Risk",  
      description: "Unsafe legacy PCEP implementation running over unauthenticated cleartext TCP 4189, aggressive 3s keepalives, shared optical SRLGs, no backup controller, and no local fallback (Grade F).",  
      hostname: "LEGACY-ROUTER-04",  
      localIp: "192.168.1.1",  
      peerIp: "192.168.1.2",  
      primaryPceIp: "192.168.1.254",  
      secondaryPceIp: "",  
      authType: "NONE",  
      tlsEnabled: false,  
      md5Key: "",  
      delegated: false,  
      pceInitiated: false,  
      keepaliveSec: 3,  
      deadTimerSec: 6,  
      srlgSharedCount: 3,  
      nodeSharedCount: 1,  
      maxSidDepth: 14,  
      localFallbackEnabled: false,  
      dbVersioningEnabled: false,  
      latencyConstrained: false,  
      telemetryAlerting: false,  
      primaryDistanceKm: 500,  
      secondaryDistanceKm: 510,  
      bandwidthGbps: 10,  
      plspId: 1,  
      srpId: 1  
    }  
  };

  let activeTab \= 'cisco';

  // Elements  
  const presetSelect \= document.getElementById('presetSelect');  
  const presetBadge \= document.getElementById('presetBadge');  
  const presetDesc \= document.getElementById('presetDesc');

  const cfgHostname \= document.getElementById('cfgHostname');  
  const cfgLocalIp \= document.getElementById('cfgLocalIp');  
  const cfgPrimaryPce \= document.getElementById('cfgPrimaryPce');  
  const cfgSecondaryPce \= document.getElementById('cfgSecondaryPce');  
  const cfgAuthType \= document.getElementById('cfgAuthType');  
  const cfgMd5Key \= document.getElementById('cfgMd5Key');  
  const cfgKeepalive \= document.getElementById('cfgKeepalive');  
  const cfgDeadTimer \= document.getElementById('cfgDeadTimer');  
  const cfgMsd \= document.getElementById('cfgMsd');

  const cfgPrimaryDist \= document.getElementById('cfgPrimaryDist');  
  const cfgSecondaryDist \= document.getElementById('cfgSecondaryDist');  
  const cfgBandwidth \= document.getElementById('cfgBandwidth');  
  const cfgSharedSrlg \= document.getElementById('cfgSharedSrlg');  
  const cfgSharedNode \= document.getElementById('cfgSharedNode');  
  const cfgPlspId \= document.getElementById('cfgPlspId');  
  const cfgSrpId \= document.getElementById('cfgSrpId');

  const cfgDelegated \= document.getElementById('cfgDelegated');  
  const cfgPceInit \= document.getElementById('cfgPceInit');  
  const cfgLocalFallback \= document.getElementById('cfgLocalFallback');  
  const cfgDbVersion \= document.getElementById('cfgDbVersion');  
  const cfgLatencyConstrained \= document.getElementById('cfgLatencyConstrained');  
  const cfgTelemetry \= document.getElementById('cfgTelemetry');

  // Outputs  
  const disjointBadge \= document.getElementById('disjointBadge');  
  const mPrimaryLat \= document.getElementById('mPrimaryLat');  
  const mSecondaryLat \= document.getElementById('mSecondaryLat');  
  const mDeltaLat \= document.getElementById('mDeltaLat');  
  const mBandwidth \= document.getElementById('mBandwidth');  
  const mDiversitySummary \= document.getElementById('mDiversitySummary');

  const auditScoreBadge \= document.getElementById('auditScoreBadge');  
  const auditGradeCircle \= document.getElementById('auditGradeCircle');  
  const auditTitle \= document.getElementById('auditTitle');  
  const auditSummary \= document.getElementById('auditSummary');  
  const findingsContainer \= document.getElementById('findingsContainer');

  const pcepBreakdown \= document.getElementById('pcepBreakdown');  
  const hexDumpView \= document.getElementById('hexDumpView');  
  const btnCopyHex \= document.getElementById('btnCopyHex');

  const configOutputView \= document.getElementById('configOutputView');  
  const btnCopyConfig \= document.getElementById('btnCopyConfig');  
  const tabButtons \= document.querySelectorAll('.tab-btn');

  function gatherConfig() {  
    return {  
      hostname: cfgHostname.value.trim(),  
      localIp: cfgLocalIp.value.trim(),  
      peerIp: '198.51.100.2',  
      primaryPceIp: cfgPrimaryPce.value.trim(),  
      secondaryPceIp: cfgSecondaryPce.value.trim(),  
      authType: cfgAuthType.value,  
      tlsEnabled: cfgAuthType.value \=== 'TLS',  
      md5Key: cfgMd5Key.value.trim(),  
      keepaliveSec: parseInt(cfgKeepalive.value, 10\) || 30,  
      deadTimerSec: parseInt(cfgDeadTimer.value, 10\) || 120,  
      maxSidDepth: parseInt(cfgMsd.value, 10\) || 5,  
      primaryDistanceKm: parseFloat(cfgPrimaryDist.value) || 1200,  
      secondaryDistanceKm: parseFloat(cfgSecondaryDist.value) || 1450,  
      bandwidthGbps: parseFloat(cfgBandwidth.value) || 100,  
      srlgSharedCount: parseInt(cfgSharedSrlg.value, 10\) || 0,  
      nodeSharedCount: parseInt(cfgSharedNode.value, 10\) || 0,  
      plspId: parseInt(cfgPlspId.value, 10\) || 1,  
      srpId: parseInt(cfgSrpId.value, 10\) || 1001,  
      delegated: cfgDelegated.checked,  
      pceInitiated: cfgPceInit.checked,  
      localFallbackEnabled: cfgLocalFallback.checked,  
      dbVersioningEnabled: cfgDbVersion.checked,  
      latencyConstrained: cfgLatencyConstrained.checked,  
      telemetryAlerting: cfgTelemetry.checked  
    };  
  }

  function applyPreset(presetKey) {  
    const p \= PRESETS\[presetKey\];  
    if (\!p) return;

    presetBadge.textContent \= p.badge;  
    presetDesc.textContent \= p.description;

    cfgHostname.value \= p.hostname;  
    cfgLocalIp.value \= p.localIp;  
    cfgPrimaryPce.value \= p.primaryPceIp;  
    cfgSecondaryPce.value \= p.secondaryPceIp;  
    cfgAuthType.value \= p.authType;  
    cfgMd5Key.value \= p.md5Key;  
    cfgKeepalive.value \= p.keepaliveSec;  
    cfgDeadTimer.value \= p.deadTimerSec;  
    cfgMsd.value \= p.maxSidDepth;

    cfgPrimaryDist.value \= p.primaryDistanceKm;  
    cfgSecondaryDist.value \= p.secondaryDistanceKm;  
    cfgBandwidth.value \= p.bandwidthGbps;  
    cfgSharedSrlg.value \= p.srlgSharedCount;  
    cfgSharedNode.value \= p.nodeSharedCount;  
    cfgPlspId.value \= p.plspId;  
    cfgSrpId.value \= p.srpId;

    cfgDelegated.checked \= p.delegated;  
    cfgPceInit.checked \= p.pceInitiated;  
    cfgLocalFallback.checked \= p.localFallbackEnabled;  
    cfgDbVersion.checked \= p.dbVersioningEnabled;  
    cfgLatencyConstrained.checked \= p.latencyConstrained;  
    cfgTelemetry.checked \= p.telemetryAlerting;

    updateUI();  
  }

  function updateUI() {  
    const cfg \= gatherConfig();

    // 1\. Calculate Path Latency & Disjointness  
    const metrics \= PCEPProtocolEngine.calculatePathMetrics({  
      primaryDistanceKm: cfg.primaryDistanceKm,  
      secondaryDistanceKm: cfg.secondaryDistanceKm,  
      bandwidthGbps: cfg.bandwidthGbps,  
      srlgSharedCount: cfg.srlgSharedCount,  
      nodeSharedCount: cfg.nodeSharedCount  
    });

    mPrimaryLat.textContent \= \`\${metrics.primaryLatencyMs} ms\`;  
    mSecondaryLat.textContent \= \`\${metrics.secondaryLatencyMs} ms\`;  
    mDeltaLat.textContent \= \`\${metrics.latencyDeltaMs} ms\`;  
    mBandwidth.textContent \= \`\${metrics.bandwidthGbps} Gbps\`;  
    mDiversitySummary.textContent \= metrics.summary;

    if (metrics.riskLevel \=== 'CRITICAL') {  
      disjointBadge.className \= 'badge badge-red';  
      disjointBadge.textContent \= 'SHARED NODE';  
    } else if (metrics.riskLevel \=== 'HIGH') {  
      disjointBadge.className \= 'badge badge-amber';  
      disjointBadge.textContent \= 'SRLG COLLISION';  
    } else {  
      disjointBadge.className \= 'badge badge-emerald';  
      disjointBadge.textContent \= 'STRICT DISJOINT';  
    }

    // 2\. Encode PCEP Message & Generate Hex Dump  
    const hops \= \[  
      { sid: 16001, loose: false, naiType: 1, ip: cfg.localIp },  
      { sid: 16002, loose: false, naiType: 1, ip: '198.51.100.2' },  
      { sid: 16003, loose: false, naiType: 1, ip: '198.51.100.3' }  
    \];

    const pcepMsg \= PCEPProtocolEngine.encodeStatefulPCEPMessage(  
      10, // PCRpt  
      cfg.plspId,  
      cfg.srpId,  
      cfg.delegated,  
      true, // Sync  
      1,    // UP  
      cfg.pceInitiated,  
      hops  
    );

    hexDumpView.textContent \= PCEPProtocolEngine.formatHexDump(pcepMsg);

    // Visual Object Dissection  
    pcepBreakdown.innerHTML \= '';  
    const objBlocks \= \[  
      { hex: '20 0A 00 3C', lbl: 'Common Header (4B: PCRpt)', cls: 'ob-hdr' },  
      { hex: '21 10 00 0C ...', lbl: \`SRP Object (12B: ID=\${cfg.srpId})\`, cls: 'ob-srp' },  
      { hex: \`20 10 00 08 (PLSP=\${cfg.plspId})\`, lbl: \`LSP Object (D=\${cfg.delegated ? 1 : 0}, C=\${cfg.pceInitiated ? 1 : 0})\`, cls: 'ob-lsp' },  
      { hex: '07 10 00 28 (3 Hops)', lbl: 'SR-ERO Object (Type 36)', cls: 'ob-ero' }  
    \];

    objBlocks.forEach(item \=\> {  
      const div \= document.createElement('div');  
      div.className \= \`obj-block \${item.cls}\`;  
      div.innerHTML \= \`\<div class="obj-hex"\>\${item.hex}\</div\>\<div class="obj-lbl"\>\${item.lbl}\</div\>\`;  
      pcepBreakdown.appendChild(div);  
    });

    // 3\. Run 10-Point Zero-Trust Audit  
    const auditRes \= PCEPSecurityAuditor.auditPCEPConfiguration(cfg);  
    auditScoreBadge.textContent \= \`\${auditRes.score} / 100\`;  
    auditGradeCircle.textContent \= auditRes.grade;  
    auditGradeCircle.className \= \`grade-badge-circle grade-\${auditRes.grade.toLowerCase().replace('+', 'plus')}\`;  
    auditTitle.textContent \= auditRes.summary;  
    auditSummary.textContent \= \`Zero-Trust Evaluation: \${auditRes.percentage}% compliance score across 10 security pillars.\`;

    findingsContainer.innerHTML \= '';  
    auditRes.findings.forEach(f \=\> {  
      const div \= document.createElement('div');  
      div.className \= 'finding-item';

      const ptsClass \= f.status \=== 'PASS' ? 'pts-pass' : (f.status \=== 'WARN' ? 'pts-warn' : 'pts-crit');  
      let html \= \`  
        \<div class="finding-top"\>  
          \<span class="finding-name"\>\${f.name}\</span\>  
          \<span class="finding-pts \${ptsClass}"\>\${f.score} / \${f.maxScore} pts \[\${f.status}\]\</span\>  
        \</div\>  
        \<div class="finding-desc"\>\${f.description}\</div\>  
      \`;  
      if (f.remediation) {  
        html \+= \`\<div class="finding-remed"\>\<strong\>Remediation:\</strong\> \${f.remediation}\</div\>\`;  
      }  
      div.innerHTML \= html;  
      findingsContainer.appendChild(div);  
    });

    // 4\. Synthesize Config  
    renderConfig(cfg);  
  }

  function renderConfig(cfg) {  
    if (\!cfg) cfg \= gatherConfig();  
    let text \= '';  
    if (activeTab \=== 'cisco') {  
      text \= PCEPConfigSynthesizer.generateCiscoIOSXR(cfg);  
    } else if (activeTab \=== 'arista') {  
      text \= PCEPConfigSynthesizer.generateAristaEOS(cfg);  
    } else if (activeTab \=== 'juniper') {  
      text \= PCEPConfigSynthesizer.generateJuniperJunos(cfg);  
    } else if (activeTab \=== 'nokia') {  
      text \= PCEPConfigSynthesizer.generateNokiaSROS(cfg);  
    } else if (activeTab \=== 'prometheus') {  
      text \= PCEPConfigSynthesizer.generatePrometheusRules(cfg);  
    }  
    configOutputView.textContent \= text;  
  }

  // Event Listeners  
  presetSelect.addEventListener('change', (e) \=\> applyPreset(e.target.value));

  const allInputs \= \[  
    cfgHostname, cfgLocalIp, cfgPrimaryPce, cfgSecondaryPce, cfgAuthType,  
    cfgMd5Key, cfgKeepalive, cfgDeadTimer, cfgMsd, cfgPrimaryDist,  
    cfgSecondaryDist, cfgBandwidth, cfgSharedSrlg, cfgSharedNode,  
    cfgPlspId, cfgSrpId, cfgDelegated, cfgPceInit, cfgLocalFallback,  
    cfgDbVersion, cfgLatencyConstrained, cfgTelemetry  
  \];  
  allInputs.forEach(input \=\> {  
    input.addEventListener('input', updateUI);  
    input.addEventListener('change', updateUI);  
  });

  tabButtons.forEach(btn \=\> {  
    btn.addEventListener('click', () \=\> {  
      tabButtons.forEach(b \=\> b.classList.remove('active'));  
      btn.classList.add('active');  
      activeTab \= btn.getAttribute('data-tab');  
      renderConfig();  
    });  
  });

  btnCopyHex.addEventListener('click', () \=\> {  
    navigator.clipboard.writeText(hexDumpView.textContent).then(() \=\> {  
      btnCopyHex.textContent \= 'Copied\!';  
      setTimeout(() \=\> btnCopyHex.textContent \= 'Copy Hex', 2000);  
    });  
  });

  btnCopyConfig.addEventListener('click', () \=\> {  
    navigator.clipboard.writeText(configOutputView.textContent).then(() \=\> {  
      btnCopyConfig.textContent \= 'Copied\!';  
      setTimeout(() \=\> btnCopyConfig.textContent \= 'Copy Config', 2000);  
    });  
  });

  // Default Init  
  applyPreset('tier1-sp-stateful-sr-pcep');  
});  
