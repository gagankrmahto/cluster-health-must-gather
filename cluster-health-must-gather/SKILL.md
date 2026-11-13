---
name: cluster-health-must-gather
description: >-
  Produce an OpenShift 4 customer-shareable cluster health summary from must-gather,
  focused on Kubernetes API (kube-apiserver) and etcd, plus operators, nodes, and MCP.
  Uses omg/omc and upstream analysis scripts. Use when the user asks for cluster health
  check, must-gather analysis, API/etcd health, or a summary to share with a customer.
---

# Cluster Health from Must-Gather (API + etcd → customer summary)

**Primary deliverable:** a customer-facing health summary centered on **API** and **etcd**. Keep raw dumps for internal notes only.

## Prerequisites

- Extracted must-gather (has `cluster-scoped-resources/`, `namespaces/`, ideally `etcd_info/`)
- `omg` (`pip install o-must-gather`) or `omc` (https://github.com/gmeghnag/omc)
- `jq` (and `column` for some scripts)

```bash
MG=<path-to-extracted-must-gather>
omg use "$MG"   # or: omc use "$MG"
```

## Workflow

```
- [ ] 1. Collect API + etcd + core signals (commands below)
- [ ] 2. Optional: run upstream analysis scripts
- [ ] 3. Optional: etcd deep dive / KCS match
- [ ] 4. Fill customer summary template → share with customer
```

### 1) Collect facts (run these)

```bash
# Identity
omg get clusterversion
omg get infrastructure cluster -o json | jq '{platform: .spec.platformSpec.type, api: .status.apiServerURL, name: .status.infrastructureName}'
omg get clusterversion -o json | jq -r '.items[0].spec.clusterID // .spec.clusterID'

# Operators (flag anything not Available=True / Progressing=False / Degraded=False)
omg get co
omg get co | awk 'NR==1 || $3!="True" || $4!="False" || $5!="False"'

# API operators — conditions
for co in kube-apiserver openshift-apiserver etcd kube-controller-manager kube-scheduler; do
  echo "=== $co ==="
  omg get co "$co" -o json | jq -r '.status.conditions[]? | select(.type=="Available" or .type=="Degraded" or .type=="Progressing") | "\(.type)=\(.status) reason=\(.reason // "-") msg=\(.message // "-")"'
done

# API pods (ignore installer / revision-pruner)
omg get pod -n openshift-kube-apiserver -o wide | grep -Ev 'installer-|revision-pruner-'

# etcd encryption (informational)
omg get apiserver cluster -o json | jq -r '.spec.encryption.type // "None/disabled"'

# etcd pods
omg get pod -n openshift-etcd -o wide | grep -E 'NAME|etcd-'

# etcd_info (from official gather_etcd)
ROOT=$(find "$MG" -maxdepth 3 -type d -name etcd_info 2>/dev/null | head -1 | xargs dirname)
EH="$ROOT/etcd_info/endpoint_health.json"
ES="$ROOT/etcd_info/endpoint_status.json"
EM="$ROOT/etcd_info/member_list.json"
EA="$ROOT/etcd_info/alarm_list.json"

jq -r '(["ENDPOINT","HEALTH","TOOK"]|(.,map(length*"-"))),(.[]|[.endpoint,(.health|tostring),.took])|@tsv' "$EH" | column -t
jq -r '.[]|"\(.Endpoint) leader=\(.Status.leader) db=\(.Status.dbSize) raft=\(.Status.raftIndex) applied=\(.Status.raftAppliedIndex) ver=\(.Status.version)"' "$ES"
jq -r '.members[]?|"\(.name) peer=\(.peerURLs|join(",")) client=\(.clientURLs|join(","))"' "$EM"
jq '.' "$EA" 2>/dev/null || echo "no alarms file"

# etcd log signals (per pod)
find "$ROOT" -path '*/namespaces/openshift-etcd/pods/*/etcd/etcd/logs/current.log' | while read -r log; do
  pod=$(echo "$log" | awk -F/ '{for(i=1;i<=NF;i++) if($i=="pods"){print $(i+1);exit}}')
  echo "=== $pod ==="
  for pat in "took too long" "leader changed" "overloaded" "failed to send out heartbeat" "clock difference" "apply request took too long"; do
    printf "  %s: " "$pat"; grep -ci "$pat" "$log" || true
  done
done

# Nodes / MCP / failing pods
omg get nodes -o wide
omg get nodes | awk 'NR==1 || $2 !~ /^Ready$/'
omg get mcp
omg get pod -A | grep -Ev 'Running|Succeeded|Completed|NAME' | head -40
```

### 2) Optional upstream scripts

| Goal | Command |
|------|---------|
| Full health snapshot | `bash <(curl -fsSL https://raw.githubusercontent.com/ay-garg/must-gather-analysis/main/must-gather-analysis.sh) "$MG"` |
| Failures only | `bash <(curl -fsSL https://raw.githubusercontent.com/ay-garg/issue-identify-must-gather/main/issue-identify-must-gather.sh) "$MG"` |
| Support Shell suite | Clone https://github.com/vsolanki12/OpenShift — `new_must_gather.sh`, `sos-report-check.sh`, `etcd_ttl.sh`, `api-server_check.sh` (RH team; case dir layout) |
| `omc etcd status` | https://github.com/gmeghnag/omc |
| etcd analyzer suite | https://github.com/peterducai/etcd-tools (`quay.io/peterducai/openshift-etcd-suite`) |
| etcd log diag | https://github.com/cptmorgan-rh/etcd-ocp-diag-script |

Official MG etcd files come from https://github.com/openshift/must-gather/blob/master/collection-scripts/gather_etcd (`endpoint_health/status`, `member_list`, `alarm_list`).

## How to judge health

**API:** `kube-apiserver` / `openshift-apiserver` Available=True, Degraded=False; KA pods Ready on each master; note short condition message if not.

**etcd:** all endpoints healthy; member count = control-plane size (usually 3); one leader; no alarms; note DB size and elevated `took too long` / heartbeat / overloaded counts.

**Overall label for customer:**

| Status | When |
|--------|------|
| Healthy | API + etcd healthy; no degraded CO; all nodes Ready |
| Attention needed | API/etcd up but degraded CO, MCP issues, etcd latency signals, or non-critical NotReady workers |
| Critical | API impact, etcd quorum/alarms at risk, or majority control-plane NotReady |

## Customer summary template (share this)

Plain language. No script names, no raw YAML. Mask secrets/IPs if requested.

```markdown
# OpenShift cluster health summary

**Date:** YYYY-MM-DD
**Prepared by:** Red Hat Support
**Data source:** must-gather collected on <date/time>
**Cluster version:** x.y.z
**Platform:** <AWS/BareMetal/vSphere/…>
**API URL:** https://api.…

## Overall status

| Area | Status | Notes |
|------|--------|-------|
| Overall cluster | Healthy / Attention needed / Critical | One sentence |
| Kubernetes API (kube-apiserver) | Healthy / Degraded / Unavailable | |
| etcd | Healthy / Degraded / At risk | |
| Nodes | All Ready / Issues found | N Ready / M NotReady |
| Cluster Operators | All Available / Issues found | Count degraded |

**Executive summary (2–4 sentences):**
What works, what does not, customer impact.

## API server health

| Check | Result |
|-------|--------|
| ClusterOperator kube-apiserver | Available=… Progressing=… Degraded=… |
| ClusterOperator openshift-apiserver | … |
| kube-apiserver pods | N/N Ready on control-plane nodes |
| Notable message | Short operator message only |

**Interpretation:** business terms (API serving vs revision stuck on node X).

## etcd health

| Check | Result |
|-------|--------|
| Endpoint health | All healthy / N unhealthy |
| Member count | Expected 3 (or SNO 1) — actual N |
| Leader | Stable / recent changes |
| DB size | ~X GiB |
| Alarms | None / <NOSPACE|CORRUPT> |
| Log signals | High-level took-too-long / heartbeat / overloaded |

**Interpretation:** quorum intact vs member unhealthy / latency elevated.

## Related signals (brief)

- Nodes: …
- MachineConfigPools: …
- Other degraded operators: name + one-line reason (max 5)

## Impact

- User / workload: …
- Upgrade / maintenance: …

## Recommended next steps

1. Immediate (safe): …
2. Data still needed (e.g. sosreport from node X): …
3. Longer-term: …

## References (customer-visible only)

- Public docs/KCS links only
```

**Writing rules:** lead with Overall table; facts vs interpretation; if `etcd_info/` missing say so — do not invent health; severity words only Healthy / Attention needed / Critical.

## KCS / docs (search exact errors too)

**Health / CLI:** [7060695](https://access.redhat.com/solutions/7060695), [7004992](https://access.redhat.com/solutions/7004992), [7120645](https://access.redhat.com/solutions/7120645)

**API:** [6976343](https://access.redhat.com/solutions/6976343) (KA Progressing), [7141476](https://access.redhat.com/solutions/7141476) (etcd→API), [6980129](https://access.redhat.com/solutions/6980129), [5833131](https://access.redhat.com/articles/5833131)

**etcd:** [6271341](https://access.redhat.com/articles/6271341) (perf guide), [6967785](https://access.redhat.com/articles/6967785) (guidelines), [5736251](https://access.redhat.com/solutions/5736251), [5498971](https://access.redhat.com/solutions/5498971), [7132959](https://access.redhat.com/articles/7132959), [7142756](https://access.redhat.com/solutions/7142756)

**Must-gather:** [Gathering cluster data](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/support/gathering-cluster-data), [5459251](https://access.redhat.com/solutions/5459251), [4591521](https://access.redhat.com/articles/4591521), [4971311](https://access.redhat.com/articles/4971311)

## Live cluster (only if API access)

Mirror KCS 7004992/7060695: `oc get nodes|co|mcp`, bad pods, `oc get csr`, `oc get pdb -A`, Warning events, etcd endpoint health. Prefer must-gather commands above when offline.
