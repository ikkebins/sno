cat > audit-high-priv.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

command -v oc >/dev/null || { echo "oc fehlt"; exit 1; }
command -v jq >/dev/null || { echo "jq fehlt"; exit 1; }

echo "1) Clusterweite cluster-admin Subjects"
oc get clusterrolebindings -o json | jq -r '
.items[]
| select(.roleRef.kind=="ClusterRole" and .roleRef.name=="cluster-admin")
| .subjects[]?
| ["cluster-admin-clusterwide", .kind, (.namespace // "-"), .name]
| @tsv' | sort -u

echo
echo "2) Namespace-lokale cluster-admin Subjects"
oc get rolebindings -A -o json | jq -r '
.items[]
| select(.roleRef.kind=="ClusterRole" and .roleRef.name=="cluster-admin")
| .subjects[]?
| ["cluster-admin-namespaced", .kind, (.namespace // "-"), .name]
| @tsv' | sort -u

echo
echo "3) SCCs mit hoher Risikosignatur"
RISKY_SCCS=$(
  oc get scc -o json | jq -r '
  .items[]
  | select(
      .metadata.name=="privileged" or
      .metadata.name=="hostmount-anyuid" or
      .metadata.name=="hostaccess" or
      .metadata.name=="hostnetwork" or
      .metadata.name=="anyuid" or
      .metadata.name=="node-exporter" or
      .allowPrivilegedContainer == true or
      .allowHostDirVolumePlugin == true or
      .allowHostNetwork == true or
      .allowHostPID == true or
      .allowHostIPC == true or
      (.runAsUser.type == "RunAsAny") or
      ((.volumes // []) | index("hostPath")) != null or
      ((.allowedCapabilities // []) | index("*")) != null
    )
  | .metadata.name' | sort -u
)
printf '%s\n' "$RISKY_SCCS"

echo
echo "4) Direkte SCC-Zuweisungen aus users/groups"
while read -r scc; do
  [ -n "$scc" ] || continue
  oc get scc "$scc" -o json | jq -r --arg scc "$scc" '
    (.users[]?  | ["direct-user",  $scc, ., "", ""] | @tsv),
    (.groups[]? | ["direct-group", $scc, ., "", ""] | @tsv)'
done <<< "$RISKY_SCCS" | sort -u

echo
echo "5) RBAC: Subjects, die riskante SCCs nutzen duerfen"
while read -r scc; do
  [ -n "$scc" ] || continue
  oc adm policy who-can use scc "$scc" 2>/dev/null \
  | awk -v scc="$scc" '
      BEGIN{section=""}
      /^Users:/  {section="user"; next}
      /^Groups:/ {section="group"; next}
      /^[[:space:]]*$/ {next}
      section=="user"  {gsub(/^[[:space:]]+/, "", $0); print "rbac-user\t"  scc "\t" $0}
      section=="group" {gsub(/^[[:space:]]+/, "", $0); print "rbac-group\t" scc "\t" $0}
    '
done <<< "$RISKY_SCCS" | sort -u

echo
echo "6) Laufzeit: Pods mit riskanter SCC"
oc get pods -A -o json | jq -r --argjson risky "$(printf '%s\n' "$RISKY_SCCS" | jq -R . | jq -s .)" '
.items[]
| .metadata.annotations["openshift.io/scc"] as $scc
| select($scc != null and ($risky | index($scc)))
| ["pod", .metadata.namespace, .metadata.name, (.spec.serviceAccountName // "default"), $scc]
| @tsv' | sort -u

echo
echo "7) Namespaces mit openshift.io/run-level"
oc get ns -o json | jq -r '
.items[]
| select(.metadata.labels["openshift.io/run-level"] != null)
| [.metadata.name, .metadata.labels["openshift.io/run-level"]]
| @tsv' | sort -u
EOF

chmod +x audit-high-priv.sh
./audit-high-priv.sh | tee audit-high-priv.tsv



# Clusterweite cluster-admin Subjects
oc get clusterrolebindings -o json | jq -r '
.items[]
| select(.roleRef.kind=="ClusterRole" and .roleRef.name=="cluster-admin")
| .subjects[]?
| [.kind, (.namespace // "-"), .name]
| @tsv' | sort -u

# Namespace-lokale Bindings auf cluster-admin
oc get rolebindings -A -o json | jq -r '
.items[]
| select(.roleRef.kind=="ClusterRole" and .roleRef.name=="cluster-admin")
| .subjects[]?
| [input_filename, .kind, (.namespace // "-"), .name]
| @tsv' 2>/dev/null | sort -u

# Aktuell laufende Pods, die riskante SCCs nutzen
oc get pods -A -o json | jq -r '
.items[]
| .metadata.annotations["openshift.io/scc"] as $scc
| select($scc != null and ($scc|test("^(privileged|hostmount-anyuid|hostaccess|hostnetwork|anyuid|node-exporter)$")))
| [.metadata.namespace, .metadata.name, (.spec.serviceAccountName // "default"), $scc]
| @tsv' | sort -u

# Wer darf die bekannten riskanten SCCs nutzen – RBAC plus direkte SCC-Einträge
for scc in privileged hostmount-anyuid hostaccess hostnetwork anyuid node-exporter; do
  echo
  echo "=== SCC: $scc ==="
  echo "[RBAC]"
  oc adm policy who-can use scc "$scc" || true
  echo "[DIRECT]"
  oc get scc "$scc" -o json | jq -r '
    (.users[]?  | "user\t"  + .),
    (.groups[]? | "group\t" + .)' || true
done

#
oc auth can-i use scc/privileged \
  --as system:serviceaccount:my-namespace:my-sa \
  -n my-namespace
oc auth can-i use scc/meine-custom-scc \
  --as system:serviceaccount:my-namespace:my-sa \
  -n my-namespace