# Scenario 01 — AMF Pod → EMS Alarm Integration (SNMP/TCP)

> **Strategy 1 — Legacy stays outside OpenShift**
> Tested on OpenShift 4.12 / 4.13 / 4.14

---

## Context

A 5G Core AMF (Access and Mobility Management Function) pod runs
inside OpenShift. It must send SNMP traps (alarms) to a legacy
EMS (Element Management System) running on a VM outside the cluster.

The EMS also exposes an HTTPS management interface on TCP 9043
used by the AMF for configuration sync.

```
PLATFORM SETUP
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  OpenShift Cluster
  ┌─────────────────────────────────────────────────────┐
  │  namespace: 5g-core                                 │
  │                                                     │
  │  ┌─────────────────────────────────────────────┐   │
  │  │  AMF pod                                    │   │
  │  │  image: vendor-registry/nokia-amf:23.6      │   │
  │  │                                             │   │
  │  │  sends SNMP traps → UDP 162  (alarms)       │   │
  │  │  calls HTTPS      → TCP 9043 (config sync)  │   │
  │  └─────────────────┬───────────────────────────┘   │
  │                    │                                │
  │  ┌─────────────────▼───────────────────────────┐   │
  │  │  Service: ems-svc                           │   │
  │  │  (bridge to legacy EMS)                     │   │
  │  └─────────────────┬───────────────────────────┘   │
  └────────────────────┼────────────────────────────────┘
                       │  TLS (TCP 9043)
                       │  UDP (SNMP 162)
                       ▼
  Legacy EMS on VM
  ┌─────────────────────────────────────────────────────┐
  │  hostname: ems.internal.telco.com                   │
  │  IP:       10.50.10.25                              │
  │  SNMP trap receiver : UDP 162                       │
  │  HTTPS management   : TCP 9043                      │
  │  TLS cert issued by : internal-telco-ca             │
  └─────────────────────────────────────────────────────┘

  Additional context:
  → Namespace has default-deny-all NetworkPolicy
  → CoreDNS has no forwarder for internal telco domain
  → EMS TLS cert signed by internal CA unknown to pods
```

---

## The Problem

Three issues combine to block the AMF → EMS integration.
Each one alone would break it. All three are present on day one.

```
ISSUE 1 — NetworkPolicy silently blocks egress
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  A default-deny-all NetworkPolicy was applied to namespace
  5g-core as part of the cluster security baseline.

  The AMF pod sends SNMP traps into a black hole.
  No error in pod logs — UDP is fire-and-forget.
  No K8s event. Pod stays Running. Completely silent.

  ⚠️  This is the most dangerous failure mode:
      the platform looks healthy but alarms never reach EMS.


ISSUE 2 — DNS resolution fails for EMS hostname
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  AMF config references ems.internal.telco.com
  CoreDNS has no forwarder for the internal telco domain.
  → "no such host" on every HTTPS config sync attempt
  → AMF logs show connection errors on startup


ISSUE 3 — EMS TLS certificate not trusted
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  EMS exposes HTTPS on TCP 9043 with a cert
  signed by the internal telecom CA.
  Pod trust store does not include this CA.
  → "x509: certificate signed by unknown authority"
  → HTTPS config sync fails even when DNS resolves
```

---

## Reproduce the Problem

Apply the broken state to your cluster:

```bash
# create namespace
oc new-project 5g-core

# apply broken manifests — reproduces all 3 issues
oc apply -f manifests/broken/
```

You will see:

```bash
# AMF pod running but EMS unreachable
oc get pods -n 5g-core
NAME        READY   STATUS    RESTARTS   AGE
amf-xxx     1/1     Running   0          2m

# DNS fails
oc exec -n 5g-core deploy/amf -- \
  nslookup ems.internal.telco.com
# → server can't find ems.internal.telco.com: NXDOMAIN

# SNMP blocked (no response, timeout)
oc exec -n 5g-core deploy/amf -- \
  nc -u -zv -w3 10.50.10.25 162
# → nc: ems.internal.telco.com (10.50.10.25:162): Connection timed out

# HTTPS fails
oc exec -n 5g-core deploy/amf -- \
  curl -v https://ems.internal.telco.com:9043/health
# → curl: (6) Could not resolve host: ems.internal.telco.com
```

---

## Diagnosis

Run these in order — each confirms one of the 3 issues.

```bash
# ── ISSUE 1 — Check NetworkPolicy ───────────────────────────────

# list all NetworkPolicies in namespace
oc get networkpolicy -n 5g-core

# expected output showing deny-all:
# NAME               POD-SELECTOR   AGE
# default-deny-all   <none>         5m

# describe to confirm no egress allowed
oc describe networkpolicy default-deny-all -n 5g-core
# → Allowing ingress traffic: <none>
# → Allowing egress traffic:  <none>   ← this is the problem

# confirm pod cannot reach EMS IP directly
oc exec -n 5g-core deploy/amf -- \
  nc -zv -w3 10.50.10.25 9043
# → connection timed out


# ── ISSUE 2 — Check DNS resolution ──────────────────────────────

# test DNS from inside the pod
oc exec -n 5g-core deploy/amf -- \
  nslookup ems.internal.telco.com
# → NXDOMAIN — CoreDNS does not know this domain

# check current CoreDNS config
oc get configmap dns-default -n openshift-dns -o yaml
# → no forwarder entry for internal.telco.com


# ── ISSUE 3 — Check TLS certificate trust ────────────────────────

# test TLS handshake from pod
oc exec -n 5g-core deploy/amf -- \
  openssl s_client -connect 10.50.10.25:9043 \
  -CAfile /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
# → Verify return code: 21 (unable to verify the first certificate)

# check what CAs the pod trusts
oc exec -n 5g-core deploy/amf -- \
  ls /etc/pki/ca-trust/source/anchors/
# → empty — no internal CA injected
```

---

## Fix

Apply fixes in order. Each fix is independent and can be applied
separately, but all three are required for full integration.

```bash
oc apply -f manifests/fixed/
```

### Fix 1 — NetworkPolicy: allow egress to EMS

```bash
oc apply -f manifests/fixed/01-networkpolicy-allow-ems-egress.yaml
```

Verify:

```bash
# SNMP UDP now reachable
oc exec -n 5g-core deploy/amf -- \
  nc -u -zv -w3 10.50.10.25 162
# → Connection to 10.50.10.25 162 port [udp/*] succeeded

# HTTPS TCP now reachable (DNS still fails — fix that next)
oc exec -n 5g-core deploy/amf -- \
  nc -zv -w3 10.50.10.25 9043
# → Connection to 10.50.10.25 9043 port [tcp/*] succeeded
```

### Fix 2 — CoreDNS: add forwarder for internal telco domain

```bash
oc apply -f manifests/fixed/02-coredns-forwarder.yaml
```

Restart CoreDNS pods to pick up the new config:

```bash
oc rollout restart deployment/dns-default -n openshift-dns
oc rollout status deployment/dns-default -n openshift-dns
```

Verify:

```bash
oc exec -n 5g-core deploy/amf -- \
  nslookup ems.internal.telco.com
# → Server: 172.30.0.10
# → Address: 10.50.10.25
```

### Fix 3 — CA bundle: inject internal telecom CA into AMF pod

```bash
oc apply -f manifests/fixed/03-ca-bundle-configmap.yaml
oc apply -f manifests/fixed/04-amf-deployment-with-ca.yaml
```

Verify:

```bash
oc exec -n 5g-core deploy/amf -- \
  curl -v https://ems.internal.telco.com:9043/health
# → SSL certificate verify ok
# → HTTP/1.1 200 OK
```

### Full integration test

```bash
# DNS resolves
oc exec -n 5g-core deploy/amf -- \
  nslookup ems.internal.telco.com

# SNMP reachable
oc exec -n 5g-core deploy/amf -- \
  nc -u -zv -w3 ems.internal.telco.com 162

# HTTPS reachable and TLS trusted
oc exec -n 5g-core deploy/amf -- \
  curl -s https://ems.internal.telco.com:9043/health
```

---

## Prevention

```
1. NEVER apply default-deny-all without also applying
   the required egress whitelists in the same commit.
   Use a GitOps PR that contains both together.

2. Document all legacy endpoints in a network dependency
   map BEFORE deploying any NF pod. Know your flows first.

3. Add CoreDNS forwarder as part of cluster bootstrap —
   not as a fix after the fact.

4. Inject internal CA bundle via a cluster-wide ConfigMap
   using the OpenShift trust bundle mechanism:

   oc label configmap internal-ca-bundle \
     config.openshift.io/inject-trusted-cabundle=true

5. Add a liveness/readiness probe that tests the EMS
   TCP endpoint — so pod health reflects EMS reachability:

   livenessProbe:
     tcpSocket:
       host: ems.internal.telco.com
       port: 9043
     initialDelaySeconds: 30
     periodSeconds: 10
```

---

## Files

```
manifests/
├── broken/
│   ├── 00-namespace.yaml                  ← namespace 5g-core
│   ├── 01-default-deny-networkpolicy.yaml ← blocks all egress
│   ├── 02-amf-deployment.yaml             ← AMF pod, no CA bundle
│   └── 03-ems-service-broken.yaml         ← ExternalName, DNS fails
└── fixed/
    ├── 01-networkpolicy-allow-ems-egress.yaml  ← whitelist UDP/TCP to EMS
    ├── 02-coredns-forwarder.yaml               ← CoreDNS internal domain
    ├── 03-ca-bundle-configmap.yaml             ← internal CA cert
    └── 04-amf-deployment-with-ca.yaml          ← AMF with CA mounted
```
