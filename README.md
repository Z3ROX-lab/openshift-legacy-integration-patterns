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

## Container Images in Production

Once you have chosen your strategy, the next question is:
**what image runs inside the pod ?**

### Strategy 1 — No pod for legacy, no image needed

```
Legacy app stays on its VM.
You only create a Service object (ExternalName or Endpoints).
No image, no pod, no container.
```

### Strategy 2 / 3 — A pod is created, image source matters

```
IMAGE SOURCE 1 — Vendor-provided image (most common in telecom)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Software vendor delivers official container images
  packaged in a Helm chart or Operator.

  Nokia      → AMF, SMF, UPF images via their private registry
  Ericsson   → 5G Core NF images via their private registry
  Oracle     → container-registry.oracle.com/database/enterprise:19c
  IBM        → icr.io/appcafe/websphere-traditional:9.0.5

  You pull using credentials provided with the vendor license.
  Image is tested and certified by the vendor for OpenShift.


IMAGE SOURCE 2 — You build the image (rehost scenario)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  No official image exists → your team builds one.
  Base image is always Red Hat UBI in production:

  FROM registry.access.redhat.com/ubi8/ubi:latest

  RUN dnf install -y java-11-openjdk

  COPY mylegacyapp.jar /opt/app/
  COPY config/         /opt/app/config/

  CMD ["java", "-jar", "/opt/app/mylegacyapp.jar"]

  Why UBI (Universal Base Image) ?
  ─────────────────────────────────
  → officially supported on OpenShift
  → passes security scans (Trivy, ACS/Stackrox)
  → no license issues
  → standard base in telecom and banking prod environments


IMAGE SOURCE 3 — Community / upstream image (non-critical only)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  docker.io/library/postgres:15
  quay.io/prometheus/prometheus:v2.45.0

  ⚠️  Never used directly in production telecom / banking.
  Always mirrored and scanned first via internal Harbor:

  docker.io/postgres:15
       │
       │  pull → Trivy scan → push
       ▼
  harbor.internal.telco.com/library/postgres:15
       │
       │  pods pull from Harbor only
       ▼
  [pod in OpenShift]
```

### Image mapping for common scenarios

```
┌───────────────────────────────┬──────────────────────────────────────┐
│ Workload                      │ Image source                         │
├───────────────────────────────┼──────────────────────────────────────┤
│ AMF / SMF / UPF (5G Core)     │ Nokia / Ericsson vendor registry     │
│ EMS adapter pod (custom)      │ UBI8 base + your adapter binary      │
│ CDR mediation pod (rehost)    │ UBI8 base + mediation binary         │
│ Oracle subscriber DB          │ Oracle official image                │
│ Legacy NRF adapter            │ UBI8 base + custom REST→SBI adapter  │
│ Websphere (banking, rehost)   │ IBM official image (icr.io)          │
│ Batch / CronJob               │ UBI8 base + batch binary / scripts   │
└───────────────────────────────┴──────────────────────────────────────┘
```

> **Production rule:** In regulated environments (telecom, banking), pods
> never pull directly from docker.io or external registries. All images
> go through an internal Harbor registry with Trivy scanning first.
> This is your supply chain security boundary.

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

    Pod calls internal K8s service name.
    K8s resolves it to the legacy hostname transparently.
    No IP hardcoded anywhere in manifests or app code.

    Pod
     │  calls "ems-svc.telecom-core.svc.cluster.local"
     ▼
    ExternalName Service
     │  externalName: ems.internal.telco.com
     ▼
    EMS on VM (10.50.10.25)

    # externalname-ems-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: ems-svc
      namespace: telecom-core
    spec:
      type: ExternalName
      externalName: ems.internal.telco.com   # legacy hostname
      ports:
      - port: 9043
        protocol: TCP

    ✅ Clean — no IP in code or manifests
    ✅ If legacy IP changes, only DNS record needs updating
    ✅ Transparent for app — calls K8s service name as usual
    ⚠️  Legacy hostname MUST be resolvable from cluster CoreDNS
    ⚠️  Does not work if legacy system has no DNS entry (IP only)

    → Use when: legacy system has a stable internal DNS name
    → Telecom example: AMF pod → EMS/NMS alarm endpoint
    → Banking example: payment pod → Websphere on-prem


  OPTION B — Service + Endpoints (IP hardcoded)
  ───────────────────────────────────────────────

    Service without selector + manual Endpoints object.
    IP of legacy system is hardcoded directly in Endpoints.
    Works for ANY protocol — TCP, UDP, JDBC, SNMP.

    Pod → ClusterIP Service → Endpoints → Legacy App on VM
                               (10.50.10.25:9043)

    # service-no-selector.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: ems-svc
      namespace: telecom-core
    spec:
      ports:
      - port: 9043          # port pods will call
        targetPort: 9043    # port on legacy system
        protocol: TCP
      # no selector — managed manually via Endpoints

    ---
    # endpoints-ems.yaml
    apiVersion: v1
    kind: Endpoints
    metadata:
      name: ems-svc         # MUST match Service name exactly
      namespace: telecom-core
    subsets:
    - addresses:
      - ip: 10.50.10.25     # legacy EMS IP — hardcoded here
      ports:
      - port: 9043

    ✅ Protocol-agnostic — TCP, UDP, JDBC, SNMP all work
    ✅ No DNS dependency — works even with IP-only legacy systems
    ✅ Simple and explicit
    ⚠️  IP is hardcoded → must update Endpoints if legacy IP changes
    ⚠️  No automatic failover if legacy has multiple IPs

    → Use when: legacy system has no DNS entry, or uses TCP/UDP
                protocols that ExternalName cannot handle
    → Telecom example: SMF pod → Oracle subscriber DB (JDBC)
    → Banking example: batch pod → DB2 on-prem (JDBC port 50000)


  OPTION C — API Gateway / Adapter (Kong or custom adapter)
  ───────────────────────────────────────────────────────────

    An adapter pod (or Kong) sits between modern NF pods
    and the legacy system. It translates protocols:
    REST/JSON → CORBA / SOAP / SNMP / proprietary.

    Pod (REST/JSON)
     │
     ▼
    ┌───────────────────────────────┐
    │  API Gateway (Kong)           │
    │  OR custom adapter pod        │
    │                               │
    │  - protocol translation       │
    │  - authentication             │
    │  - rate limiting              │
    │  - request/response logging   │
    └───────────────┬───────────────┘
                    │  legacy protocol
                    │  (CORBA / SOAP / SNMP)
                    ▼
    Legacy App on VM (unchanged)

    Kong deployed in OpenShift as an Operator:
    ────────────────────────────────────────────
    # kong route pointing to legacy backend
    apiVersion: configuration.konghq.com/v1
    kind: KongIngress
    metadata:
      name: legacy-ems-route
      namespace: telecom-core
    upstream:
      host: ems.internal.telco.com
      port: 9043

    Custom adapter pod (lightweight approach):
    ────────────────────────────────────────────
    # adapter translates REST POST /alarm
    # → SNMP trap to EMS
    # deployed as a standard OpenShift Deployment
    # exposed as ClusterIP Service to NF pods

    ✅ Modern NF pods speak REST — no legacy protocol knowledge needed
    ✅ Legacy app never changes
    ✅ Add auth, rate-limiting, logging at gateway layer
    ✅ Single point to monitor legacy connectivity
    ⚠️  Extra component to build, deploy, and maintain
    ⚠️  Adds latency (one extra hop)
    ⚠️  Gateway becomes a single point of failure → needs HA

    → Use when: legacy uses CORBA / SOAP / proprietary protocol
                that pods cannot speak natively
    → Telecom example: 5G NF (REST) → legacy OSS (CORBA/SOAP)
    → Banking example: microservice (REST) → Websphere EJB (IIOP)


  CONNECTIVITY PATTERN DECISION GUIDE
  ─────────────────────────────────────

  ┌──────────────────────────────┬────────────┬────────────┬──────────┐
  │ Situation                    │ Option A   │ Option B   │ Option C │
  ├──────────────────────────────┼────────────┼────────────┼──────────┤
  │ Legacy has DNS entry         │ ✅ ideal   │ works      │ works    │
  │ Legacy has IP only           │ ❌         │ ✅ ideal   │ works    │
  │ Protocol is HTTP/HTTPS       │ ✅         │ ✅         │ overkill │
  │ Protocol is JDBC/TCP         │ ⚠️ limited │ ✅ ideal   │ works    │
  │ Protocol is SNMP/UDP         │ ❌         │ ✅ ideal   │ works    │
  │ Protocol is CORBA/SOAP       │ ❌         │ ❌         │ ✅ ideal │
  │ Need auth / rate limiting    │ ❌         │ ❌         │ ✅       │
  │ Simplest to implement        │ ✅         │ ✅         │ ❌       │
  └──────────────────────────────┴────────────┴────────────┴──────────┘


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
