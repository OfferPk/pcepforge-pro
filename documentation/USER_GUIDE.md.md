# PCEPForge Pro \- User Guide & DevSecOps Manual

## 1\. Quickstart: Interactive Web Studio

1. Open `source-code/index.html` in any web browser. Completely standalone, zero dependencies.  
2. Select an architecture preset from the **Industry Architecture Presets** dropdown:  
   - **Tier-1 Service Provider 400G Backbone**: Stateful SR-TE with TLS mutual auth, geo-redundant PCE, incremental DB sync, and SRLG diversity (Grade A+).  
   - **Cloud Hyper-Scaler DCI Dual-Disjoint Fabric**: 100G capacity reservation with microsecond latency SLA bounds (Grade A+).  
   - **Vulnerable Stateless Unauthenticated PCEP**: Demonstrates risks of cleartext TCP 4189, tight 3s timers, and optical SRLG collisions (Grade F).  
3. Review the **Path Latency & SRLG Disjointness Analytics** metrics panel to verify propagation delay and diversity status.  
4. Inspect the **10-Point Zero-Trust PCEP Security Audit** to evaluate design vulnerabilities and remediation guidance.  
5. Review the **RFC 8231 PCEP PCRpt / PCUpd Binary Dissection** and live **Wireshark Hex Dump** with 1-click clipboard copy.  
6. Switch across vendor tabs under **Multi-Vendor PCEP Configuration Synthesizer** (Cisco IOS-XR, Arista EOS, Juniper Junos, Nokia SR-OS, Prometheus Alerts) to copy verified configurations.

---

## 2\. DevSecOps CLI Tool (`pcepforge-cli.py`)

PCEPForge Pro includes a zero-dependency Python 3 CLI tool located at `build/pcepforge-cli.py`.

### 2.1 Encoding Stateful PCEP PCRpt Frames

python3 build/pcepforge-cli.py encode-pcep \-p source-code/presets/tier1-sp-stateful-sr-pcep.json

Output:

RFC 8231 Stateful PCEP PCRpt Frame (64 Bytes):

\=================================================================

0000  20 0a 00 40 21 10 00 0c  00 00 00 00 00 00 13 89  | ..@\!...........|

0010  20 10 00 08 00 06 5d 30  07 10 00 28 24 0c 11 08  | .....\]0...(\$...|

0020  00 00 3e 81 c6 33 64 01  24 0c 11 08 00 00 3e 82  |..\>..3d.\$.....\>.|

0030  c6 33 64 02 24 0c 11 08  00 00 3e 83 c6 33 64 03  |.3d.\$.....\>..3d.|

### 2.2 Calculating Path Latency & SRLG Disjointness

python3 build/pcepforge-cli.py calc-path \--primary-km 1200 \--secondary-km 1450 \--srlg-shared 0 \--node-shared 0

### 2.3 Running 10-Point Zero-Trust Audits in CI/CD Pipelines

\# Regular audit

python3 build/pcepforge-cli.py audit \-p source-code/presets/tier1-sp-stateful-sr-pcep.json

\# Strict CI/CD gate: exits with code 1 if score \< 90 or grade is not A/A+

python3 build/pcepforge-cli.py audit \-p source-code/presets/tier1-sp-stateful-sr-pcep.json \--strict

### 2.4 Synthesizing Router Configurations

\# Cisco IOS-XR SR-TE PCC

python3 build/pcepforge-cli.py synthesize \-p source-code/presets/tier1-sp-stateful-sr-pcep.json \-f cisco

\# Arista EOS

python3 build/pcepforge-cli.py synthesize \-p source-code/presets/tier1-sp-stateful-sr-pcep.json \-f arista

\# Juniper Junos

python3 build/pcepforge-cli.py synthesize \-p source-code/presets/tier1-sp-stateful-sr-pcep.json \-f juniper

\# Nokia SR-OS

python3 build/pcepforge-cli.py synthesize \-p source-code/presets/tier1-sp-stateful-sr-pcep.json \-f nokia

\# Prometheus Alerting Rules

python3 build/pcepforge-cli.py synthesize \-p source-code/presets/tier1-sp-stateful-sr-pcep.json \-f prometheus