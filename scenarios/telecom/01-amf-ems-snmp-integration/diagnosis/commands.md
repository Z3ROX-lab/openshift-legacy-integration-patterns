# DIAGNOSIS COMMANDS — Scenario 01
# AMF pod → EMS SNMP/HTTPS integration
# Run these in order to identify which of the 3 issues is present.

# ── STEP 1 — Check pod is running ───────────────────────────────

oc get pods -n 5g-core
# Expected (broken): pod Running but EMS unreachable
# NAME        READY   STATUS    RESTARTS   AGE
# amf-xxx     1/1     Running   0          2m


# ── STEP 2 — Check NetworkPolicy ────────────────────────────────

# list all policies in namespace
oc get networkpolicy -n 5g-core

# check if egress is allowed
oc describe networkpolicy default-deny-all -n 5g-core
# Look for:
# → "Allowing egress traffic: <none>" = problem confirmed

# test TCP reachability to EMS directly by IP
oc exec -n 5g-core deploy/amf -- \
  nc -zv -w3 10.50.10.25 9043
# broken  → nc: Connection timed out
# fixed   → Connection to 10.50.10.25 9043 port [tcp/*] succeeded

# test UDP reachability (SNMP)
oc exec -n 5g-core deploy/amf -- \
  nc -u -zv -w3 10.50.10.25 162
# broken  → timeout (silent — UDP has no connection error)
# fixed   → succeeded


# ── STEP 3 — Check DNS resolution ───────────────────────────────

# test from inside pod
oc exec -n 5g-core deploy/amf -- \
  nslookup ems.internal.telco.com
# broken  → NXDOMAIN
# fixed   → Address: 10.50.10.25

# check CoreDNS config for forwarder
oc get configmap dns-default -n openshift-dns -o yaml | grep -A5 telco
# broken  → nothing returned (no forwarder)
# fixed   → internal.telco.com:5353 block present

# check CoreDNS pods are running
oc get pods -n openshift-dns
oc logs -n openshift-dns -l dns.operator.openshift.io/daemonset-dns \
  --tail=20


# ── STEP 4 — Check TLS certificate trust ────────────────────────

# test TLS handshake from pod (use IP since DNS may be broken)
oc exec -n 5g-core deploy/amf -- \
  openssl s_client -connect 10.50.10.25:9043 \
  -CAfile /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem \
  </dev/null 2>&1 | grep -E "Verify|issuer|subject"
# broken  → Verify return code: 21 (unable to verify)
# fixed   → Verify return code: 0 (ok)

# check CA bundle is mounted in pod
oc exec -n 5g-core deploy/amf -- \
  ls /etc/pki/ca-trust/source/anchors/
# broken  → empty
# fixed   → internal-telco-ca.crt present

# test full HTTPS call
oc exec -n 5g-core deploy/amf -- \
  curl -sv https://ems.internal.telco.com:9043/health 2>&1 \
  | grep -E "SSL|HTTP|Connected"
# broken  → SSL: no alternative certificate subject name matches
# fixed   → SSL certificate verify ok / HTTP/1.1 200


# ── STEP 5 — Full integration test (all 3 fixes applied) ────────

echo "=== DNS ==="
oc exec -n 5g-core deploy/amf -- \
  nslookup ems.internal.telco.com

echo "=== SNMP UDP ==="
oc exec -n 5g-core deploy/amf -- \
  nc -u -zv -w3 ems.internal.telco.com 162

echo "=== HTTPS TCP ==="
oc exec -n 5g-core deploy/amf -- \
  curl -s -o /dev/null -w "%{http_code}" \
  https://ems.internal.telco.com:9043/health
# expected: 200
