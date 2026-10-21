# PCEPForge Pro \- Technical Architecture & Protocol Engineering Manual

## 1\. Standards Foundation & Scope

PCEPForge Pro implements the core Internet Engineering Task Force (IETF) Request for Comments (RFCs) governing the Path Computation Element Communication Protocol (PCEP) and Segment Routing Traffic Engineering (SR-TE):

- **RFC 5440**: Path Computation Element (PCE) Communication Protocol (PCEP)  
- **RFC 8231**: PCEP Extensions for Stateful PCE (LSP delegation, state synchronization, PCRpt, PCUpd)  
- **RFC 8281**: PCEP Extensions for PCE-Initiated LSP Setup in a Stateful PCE Model (PCInit, C-bit)  
- **RFC 8664**: PCEP Extensions for Segment Routing (SR-ERO, SR-RRO, NAI types)  
- **RFC 8253**: PCEPS: Usage of TLS to Provide a Secure Transport Channel for PCEP  
- **RFC 8232**: Optimizing the Path Computation Element Communication Protocol (PCEP) State Synchronization (Incremental DB-Versioning)  
- **RFC 8800**: Optimizing the Extended Association ID for Disjointness and Association in PCEP (SRLG, Node, Link diversity)

---

## 2\. PCEP Common Header & Binary Wireframe

Every PCEP message starts with a mandatory 4-octet Common Header formatted in big-endian network byte order:

 0                   1                   2                   3

 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1

\+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

| Ver |  Flags  |  Message-Type |       Message-Length          |

\+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

### 2.1 Bit-Field Specifications:

1. **Ver (Version, 3 bits)**: Fixed value `1` (`001b`).  
2. **Flags (5 bits)**: Reserved (`00000b`).  
3. **Message-Type (8 bits)**:  
   - `1`: PCReq (Path Computation Request)  
   - `2`: PCRep (Path Computation Reply)  
   - `4`: PCErr (PCEP Error)  
   - `7`: Open (Session establishment)  
   - `8`: Keepalive (Liveness verification)  
   - `10`: PCRpt (Path Computation Report \- PCC reports LSP state to PCE)  
   - `11`: PCUpd (Path Computation Update \- PCE pushes updated path to PCC)  
   - `12`: PCInit (Path Computation Initiate \- PCE creates new LSP on PCC)  
4. **Message-Length (16 bits)**: Total length of the PCEP message in octets, including the 4-byte header.

---

## 3\. Stateful PCEP Objects & Segment Routing ERO

### 3.1 SRP (Stateful Request Parameters) Object (Class 33, Type 1, 12 Octets)

Correlates path computation update requests between PCE and PCC:

- **Flags (32 bits)**: Bit 31 is the `R (Remove)` bit (triggers LSP teardown).  
- **SRP-ID-number (32 bits)**: Monotonically increasing sequence number.

### 3.2 LSP Object (Class 32, Type 1, 8 Octets)

Specifies target LSP parameters and operational states:

- **PLSP-ID (20 bits)**: Persistent numeric identifier assigned by PCC.  
- **Flags (12 bits)**:  
  - `D (Delegate, bit 11)`: If `1`, LSP control is delegated to the PCE.  
  - `S (Sync, bit 10)`: If `1`, state synchronization is active.  
  - `R (Remove, bit 9)`: If `1`, delete LSP.  
  - `A (Admin, bit 8)`: Administrative state (1=UP, 0=DOWN).  
  - `O (Operational, bits 7-5)`: 0=Down, 1=Up, 2=Active, 3=Going-Down, 4=Going-Up.  
  - `C (Create, bit 4)`: Set to `1` for PCE-Initiated LSPs (RFC 8281).

### 3.3 SR-ERO Subobject (RFC 8664, Type 36, 12 Octets per Hop)

Encodes explicit Segment Routing hops using MPLS Segment Identifiers (SIDs) or SRv6 Functions:

- **L-bit (1 bit)**: `0` for Strict hop, `1` for Loose hop.  
- **Type (7 bits)**: `36` (SR-ERO Subobject).  
- **Length (8 bits)**: `12` octets.  
- **NAI Type (NT, 4 bits)**: 1 \= IPv4 Node ID, 2 \= IPv6 Node ID, 3 \= IPv4 Adjacency.  
- **Flags (12 bits)**: `M=1` (MPLS SID), `S=1` (SID present).  
- **SID (32 bits)**: 20-bit MPLS label (e.g., 16001, 16002).  
- **NAI (32 bits)**: IPv4 address of the target hop.

---

## 4\. Latency Calculation & Optical Propagation Dynamics

Optical transmission latency in single-mode fiber (SMF-28) is governed by the speed of light in silica glass core (\$n \\approx 1.4682\$, \$c \\approx 204,190\\text{ km/s}\$):

\$\$\\text{One-Way Fiber Latency} \\approx 4.90,\\mu\\text{s per km} \\quad (0.0049,\\text{ms/km})\$\$

\$\$\\text{Round-Trip Time (RTT)} \\approx 9.80,\\mu\\text{s per km}\$\$

On long-haul links (e.g. NYC to London, \~5,500 km), physical fiber latency alone is \~27.0 ms one-way (\~54 ms RTT). PCEP Metric Type 11 allows centralized controllers to enforce strict sub-millisecond latency budgets for high-frequency trading (HFT) and 5G URLLC network slices.

---

## 5\. SRLG Diversity & Disjoint Association

Shared Risk Link Groups (SRLGs) model physical fiber cables sharing the same conduit, bridge crossing, or trench. PCEP Association Object (RFC 8800\) enforces mathematical path diversity:

- **Node-Disjoint**: Primary and secondary paths traverse zero common transit routers.  
- **SRLG-Disjoint**: Primary and secondary paths traverse zero shared optical ducts. A backhoe cut severing conduit A leaves conduit B 100% operational.