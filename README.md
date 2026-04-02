# openshift-legacy-integration-patterns

> **Field notes from senior engineers who have connected legacy systems
> to OpenShift platforms in production — telecom and banking environments.**
>
> This is not a textbook. It is a cheat sheet built from real incidents,
> real misconfigurations, and real fixes.

---

## Who This Is For

- OpenShift / Kubernetes Platform Engineers working in production
- Cloud Native Architects dealing with legacy system coexistence
- SRE / Platform Ops teams in regulated environments (telecom, banking)

---

## Table of Contents

1. [Telecom Platform Context](#telecom-platform-context)
2. [The 3 Integration Strategies](#the-3-integration-strategies)
3. [Real Integration Flow — Step by Step](#real-integration-flow--step-by-step)
4. [Most Common Integration Problems](#most-common-integration-problems)
5. [Scenarios](#scenarios)

---

## Telecom Platform Context

Understanding the platform before integrating anything into it.

```
TYPICAL TELECOM OPENSHIFT PLATFORM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────────────────────────────────────────────┐
  │                   TELECOM CLOUD PLATFORM                     │
  │                                                              │
  │  ┌─────────────────────┐    ┌─────────────────────────────┐ │
  │  │  OpenShift          │    │  OpenShift                  │ │
  │  │  Cluster — Core NFs │    │  Cluster — OSS/BSS          │ │
  │  │                     │    │                             │ │
  │  │  AMF  SMF  UPF      │    │  Mediation  Charging  OCS   │ │
  │  │  NRF  AUSF UDM      │    │  CDR collection  Billing    │ │
  │  │  CU-CP CU-UP DU     │    │                             │ │
  │  └──────────┬──────────┘    └──────────────┬──────────────┘ │
  │             │                              │                 │
  │  ┌──────────▼──────────────────────────────▼──────────────┐ │
  │  │                    LEGACY LAYER                        │ │
  │  │                                                        │ │
  │  │   EMS / NMS      NFVO          OSS/BSS legacy          │ │
  │  │   (on VMs)     (on VMs)        (Java / CORBA)          │ │
  │  │                                                        │ │
  │  │   CFT server     SNMP traps    Legacy NRF (4G)         │ │
  │  │   (file xfer)    receiver      (VM-based)              │ │
  │  └────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────┘

  Key constraints specific to telecom:
  ─────────────────────────────────────────────────────────
  → Ultra-low latency workloads (RAN DU pods < 1ms)
  → NUMA / CPU pinning / HugePages mandatory for UPF / DU
  → SR-IOV for N3 / fronthaul network interfaces
  → Real-time kernel on RAN worker nodes
  → Airgap environments (lab, field deployments)
  → 3GPP interface naming: N1 N2 N3 N4 N6 N9
```

---

## The 3 Integration Strategies

Before writing a single manifest, choose your strategy.
The wrong choice costs weeks.

```
STRATEGY 1 — Keep Legacy Outside (most common in production)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Legacy app stays on its VM / bare metal.
  OpenShift pods connect TO it over the network.

  ┌───────────────────┐          ┌──────────────────────────┐
  │  OpenShift Cluster│          │  Legacy System (on-prem) │
  │                   │          │                          │
  │  [AMF pod]  ──────┼──────────┼──→ [EMS on VM]           │
  │  [Mediation pod]──┼──────────┼──→ [CFT server]          │
  │  [Charging pod]───┼──────────┼──→ [Oracle DB on VM]     │
  │                   │   TLS /  │                          │
  └───────────────────┘   mTLS   └──────────────────────────┘

  ✅ Zero risk for legacy app — nothing changes on legacy side
  ✅ Fastest to implement
  ✅ Fully reversible
  ⚠️  Network + Security = main engineering challenge
  ⚠️  Latency depends on network path to legacy

  → Used for: EMS/NMS alarms, CDR collection,
               external DB, legacy NRF discovery


STRATEGY 2 — Rehost / Lift & Shift
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Legacy app is containerized as-is.
  Same code, same config — wrapped in a container image.

  ┌─────────────────────────────────────────────────────────┐
  │  OpenShift Cluster                                      │
  │                                                         │
  │  ┌──────────────────────────────────────┐              │
  │  │  Legacy app running as a Pod         │              │
  │  │  (same binary, same config files)    │              │
  │  │                                      │              │
  │  │  ┌────────────────────────────────┐  │              │
  │  │  │  PersistentVolumeClaim (NFS)   │  │              │
  │  │  │  → replaces local filesystem   │  │              │
  │  │  └────────────────────────────────┘  │              │
  │  └──────────────────────────────────────┘              │
  └─────────────────────────────────────────────────────────┘

  ✅ Removes VM dependency
  ✅ Benefits from K8s scheduling and HA
  ⚠️  App is NOT cloud-native → state management is complex
  ⚠️  Needs PersistentVolumes if app writes to disk
  ⚠️  Startup time may not match pod lifecycle expectations

  → Used for: mediation components, batch containers,
               internal tooling with manageable complexity


STRATEGY 3 — Refactor / Modernize
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Legacy app rewritten as cloud-native microservices.

  ✅ Fully cloud-native — scales, self-heals
  ❌ Very expensive and time-consuming
  ❌ High risk — full rewrite of business logic
  ❌ Rarely justified unless app is end-of-life

  → Used for: greenfield 5G NFs, new cloud-native BSS modules


DECISION MATRIX
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────┬────────────┬──────────┬──────────┐
  │ Criteria             │ Strategy 1 │Strategy 2│Strategy 3│
  ├──────────────────────┼────────────┼──────────┼──────────┤
  │ Speed to implement   │ Fast       │ Medium   │ Slow     │
  │ Risk to legacy       │ Zero       │ Medium   │ High     │
  │ Operational benefit  │ Low        │ Medium   │ High     │
  │ Cloud-native result  │ No         │ Partial  │ Yes      │
  │ Recommended in prod  │ ✅ Yes     │ ⚠️  Maybe│ ❌ Rarely│
  └──────────────────────┴────────────┴──────────┴──────────┘
```

> **Field note:** In telecom production environments, Strategy 1 is almost
> always the right first step. Operators want zero impact on live network.
> Modernization comes later, in phases.

---

## Real Integration Flow — Step by Step

```
PHASE 1 — ASSESS THE LEGACY APP
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Do NOT touch anything before answering these questions.

  ┌─────────────────────────────────────────────────────────┐
  │  STATEFULNESS                                           │
  │  → Does the app store session data locally ?            │
  │  → Does it write to local filesystem ?                  │
  │  → Are connections long-lived (CORBA, JDBC pools) ?     │
  ├─────────────────────────────────────────────────────────┤
  │  DEPENDENCIES                                           │
  │  → Which databases does it connect to ?                 │
  │  → Does it consume message queues (JMS / Kafka) ?       │
  │  → Does it receive/send files (CFT / SFTP) ?            │
  │  → Does it depend on batch schedulers (Autosys) ?       │
  │  → Does it send/receive SNMP traps ?                    │
  ├─────────────────────────────────────────────────────────┤
  │  PROTOCOLS                                              │
  │  → HTTP/HTTPS  → straightforward, use Ingress/Route     │
  │  → JDBC/TCP    → needs ExternalName or direct IP        │
  │  → SNMP/UDP    → needs explicit egress NetworkPolicy    │
  │  → CORBA/IIOP  → complex, consider API adapter layer    │
  │  → SFTP        → non-standard port, egress whitelist    │
  ├─────────────────────────────────────────────────────────┤
  │  AUTHENTICATION                                         │
  │  → LDAP / Active Directory                              │
  │  → Kerberos (common in telecom OSS)                     │
  │  → Certificate-based (mutual TLS)                       │
  │  → Username/password (legacy DB)                        │
  ├─────────────────────────────────────────────────────────┤
  │  NETWORK CONSTRAINTS                                     │
  │  → IP range of legacy system                            │
  │  → Firewall rules already in place ?                    │
  │  → DNS resolvable from cluster ?                        │
  └─────────────────────────────────────────────────────────┘


PHASE 2 — CHOOSE CONNECTIVITY PATTERN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  OPTION A — ExternalName Service (DNS alias)
  ─────────────────────────────────────────────

    Pod calls internal K8s service name
    → K8s resolves to legacy hostname
    → No IP hardcoding anywhere

    Pod
     │  calls "ems-svc.telecom-core.svc.cluster.local"
     ▼
    ExternalName Service
     │  externalName: ems.internal.telco.com
     ▼
    EMS on VM (10.50.10.25)

    ✅ Clean — no IP in code or manifests
    ⚠️  Legacy hostname must be DNS-resolvable from CoreDNS


  OPTION B — Endpoints + Service (direct IP)
  ─────────────────────────────────────────────

    Service (no selector) + manual Endpoints object
    → IP of legacy system hardcoded in Endpoints
    → Works for any protocol (TCP, UDP, JDBC)

    Pod → ClusterIP Service → Endpoints (ip: 10.50.10.25)
                                              ↓
                                       Legacy App on VM

    ✅ Protocol-agnostic (TCP, UDP, SNMP, JDBC)
    ✅ No DNS dependency
    ⚠️  IP hardcoded → must update if legacy IP changes


  OPTION C — API Gateway / Adapter
  ─────────────────────────────────────────────

    Adapter pod translates modern protocol to legacy protocol
    → REST/JSON → CORBA / SOAP / SNMP

    Pod → REST calls → Adapter Pod → legacy protocol → Legacy App

    ✅ Decouples modern NFs from legacy protocol
    ✅ Add auth, logging, rate-limiting at gateway
    ✅ Legacy app never changes
    ⚠️  Additional component to maintain


PHASE 3 — SECURITY LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Apply in this order — do not skip steps.

  3.1 NetworkPolicy
  ──────────────────
    Default posture: deny all egress from namespace.
    Then whitelist only the exact pods, IPs, and ports needed.

  3.2 TLS / Certificate trust
  ────────────────────────────
    Legacy systems use internal CA.
    Pods will reject their certs by default.
    Fix: inject internal CA bundle as ConfigMap → mount in pod.

  3.3 Secrets management
  ────────────────────────
    Never hardcode legacy credentials in manifests.
    Use Vault + ESO (External Secrets Operator):
    Vault → ExternalSecret CR → OpenShift Secret → pod env var


PHASE 4 — STATE AND STORAGE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Legacy app writes files locally
  → PersistentVolumeClaim backed by NFS or Ceph
  → Mount at the same path the app expects

  Legacy app stores session state in memory
  → Externalize to Redis or DB before containerizing
  → Pod restart = session loss if not externalized

  CDR / batch file collection
  → Init container or sidecar stages files from CFT into PVC
  → Main container processes files from PVC


PHASE 5 — OBSERVABILITY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  The pod → legacy integration point is a blind spot.
  Instrument it explicitly.

  What to monitor:
  → Is the legacy app reachable ? (TCP probe)
  → Latency of calls from pod to legacy
  → Error rate on legacy calls
  → Connection pool usage
  → File transfer success/failure rate


PHASE 6 — FULL ARCHITECTURE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

              gNB / RAN (external)
                     │
              N2 / N3 interfaces
                     │
   ┌─────────────────▼──────────────────────────┐
   │            OpenShift Cluster                │
   │                                             │
   │  ┌──────────┐          ┌─────────────────┐  │
   │  │ AMF pod  │          │  UPF pod        │  │
   │  └────┬─────┘          └────────┬────────┘  │
   │       │                         │            │
   │  ┌────▼─────────────────────────▼────────┐  │
   │  │         NetworkPolicy layer            │  │
   │  │  (whitelist per interface / port)      │  │
   │  └────────────────┬──────────────────────┘  │
   │                   │                          │
   │  ┌────────────────▼──────────────────────┐  │
   │  │  ExternalName Services / Endpoints    │  │
   │  │  (connectivity bridge to legacy)      │  │
   │  └────────────────┬──────────────────────┘  │
   └──────────────────┬┴─────────────────────────┘
                      │
               TLS / mTLS
                      │
   ┌──────────────────▼─────────────────────────┐
   │         LEGACY LAYER (on-prem VMs)          │
   │                                             │
   │  ┌──────────┐    ┌──────────────────────┐  │
   │  │ EMS/NMS  │    │  Oracle / DB2        │  │
   │  └──────────┘    └──────────────────────┘  │
   │                                             │
   │  ┌──────────┐    ┌──────────────────────┐  │
   │  │ CFT srv  │    │  Legacy NRF (4G VM)  │  │
   │  └──────────┘    └──────────────────────┘  │
   └─────────────────────────────────────────────┘
                      │
   ┌──────────────────▼─────────────────────────┐
   │  Vault + ESO  /  Prometheus + Grafana + Loki│
   └─────────────────────────────────────────────┘
```

---

## Most Common Integration Problems

| Symptom | Most likely cause |
|---|---|
| Pod Running but app unreachable | NetworkPolicy blocking egress |
| DNS name not resolving in pod | CoreDNS missing forwarder for internal domain |
| TLS handshake failure | Internal CA not trusted by pod |
| Works with 1 replica, fails at 10 | Connection pool exhaustion on legacy system |
| SNMP alarms not received by EMS | UDP port not whitelisted in NetworkPolicy |
| Intermittent timeout to legacy | Firewall idle timeout killing long-lived TCP connections |
| Pod crashes after file transfer | PVC full — no disk space monitoring on volume |

Detailed diagnosis commands and fixes are included in each scenario.

---

## Scenarios

Each scenario includes:
- **Context** — what the platform looks like
- **The problem** — what broke and why
- **Diagnosis** — exact commands with expected output
- **Fix** — working manifests ready to apply
- **Prevention** — what to put in place to avoid recurrence

### Telecom Scenarios

| # | Scenario | Strategy | Status |
|---|---|---|---|
| 01 | [AMF pod → EMS alarm integration (SNMP/TCP)](./scenarios/telecom/01-amf-ems-snmp-integration/) | Strategy 1 | 🚧 Coming |
| 02 | [Legacy NRF coexistence — 4G to 5G transition](./scenarios/telecom/02-legacy-nrf-coexistence/) | Strategy 1 | 🚧 Coming |
| 03 | [CDR collection via CFT file transfer](./scenarios/telecom/03-cdr-cft-file-transfer/) | Strategy 1 | 🚧 Coming |
| 04 | [Oracle subscriber DB access from NF pods](./scenarios/telecom/04-oracle-db-connectivity/) | Strategy 1 | 🚧 Coming |

### Banking Scenarios

| # | Scenario | Strategy | Status |
|---|---|---|---|
| 05 | [Websphere app connectivity from K8s pods](./scenarios/banking/05-websphere-connectivity/) | Strategy 1 | 🚧 Coming |
| 06 | [Autosys batch job integration](./scenarios/banking/06-autosys-batch-integration/) | Strategy 1/2 | 🚧 Coming |

---

## Environment Tested On

- OpenShift 4.12 / 4.13 / 4.14
- OKD 4.15 (SNO homelab)
- Kubernetes 1.26+

---

## Related

- [openshift-mco-incident-runbooks](https://github.com/Z3ROX-lab/openshift-mco-incident-runbooks) — Production incident runbooks for OpenShift platform teams

---

## Author

Cloud Native Security Architect — 20+ years in telecom and cloud infrastructure.
OpenShift platforms in production: Nokia, 5G Core, O-RAN, CloudRAN.

GitHub: [Z3ROX-lab](https://github.com/Z3ROX-lab) · Medium: [@Z3R0X](https://medium.com/@Z3R0X)

---

*Field notes. Real incidents. No fluff.*
