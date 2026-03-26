# Splunk OpenShift lab — command reference

Replace **`splunk-demo`** if using another namespace. OpenShift **4.14+**: use **`nonroot-v2`** SCC (older: `nonroot` per Splunk OpenShift doc).

## Preflight (read-only)

```bash
oc whoami
oc version -o yaml | head -40
oc get nodes -o wide
oc get storageclass
oc get scc nonroot-v2 nonroot restricted-v2 --no-headers 2>/dev/null || true
command -v helm && helm version
oc auth can-i create customresourcedefinitions --all-namespaces
oc adm top nodes 2>/dev/null || true
```

## Install Splunk Operator + Standalone (lab)

**CRDs** (use a GitHub **release** that publishes `splunk-operator-crds.yaml`; if only **3.0.0** exists while Helm is **3.1.x**, apply **3.0.0** CRDs and proceed—verify compatibility in operator release notes):

```bash
oc new-project splunk-demo || oc project splunk-demo
oc apply -f https://github.com/splunk/splunk-operator/releases/download/3.0.0/splunk-operator-crds.yaml --server-side
oc adm policy add-scc-to-user nonroot-v2 -z splunk-operator-controller-manager -n splunk-demo
oc adm policy add-scc-to-user nonroot-v2 -z default -n splunk-demo
```

**Helm** (Splunk 10.x requires General Terms acceptance; **override operator image** if the chart’s default tag is not published on Docker Hub):

```bash
cat > /tmp/splunk-operator-values.yaml <<'EOF'
splunkOperator:
  splunkGeneralTerms: "--accept-sgt-current-at-splunk-com"
EOF
helm repo add splunk https://splunk.github.io/splunk-operator/ && helm repo update
helm upgrade --install splunk-operator splunk/splunk-operator -n splunk-demo \
  -f /tmp/splunk-operator-values.yaml --version 3.1.0 \
  --set splunkOperator.image.repository=docker.io/splunk/splunk-operator:3.0.0
oc wait --for=condition=available deployment/splunk-operator-controller-manager -n splunk-demo --timeout=300s
```

**Standalone** (Splunk Free; minimal PVCs/resources—tune for lab). Set **`storageClassName`** from `oc get storageclass` (example placeholder below):

```bash
oc apply -f - <<'EOF'
apiVersion: enterprise.splunk.com/v4
kind: Standalone
metadata:
  name: lab
  namespace: splunk-demo
  finalizers:
    - enterprise.splunk.com/delete-pvc
spec:
  etcVolumeStorageConfig:
    storageClassName: YOUR_STORAGE_CLASS
    storageCapacity: 10Gi
  varVolumeStorageConfig:
    storageClassName: YOUR_STORAGE_CLASS
    storageCapacity: 20Gi
  resources:
    requests:
      cpu: 500m
      memory: 4Gi
    limits:
      cpu: "2"
      memory: 6Gi
  extraEnv:
    - name: SPLUNK_START_ARGS
      value: "--accept-license"
    - name: SPLUNK_LICENSE_URI
      value: "Free"
EOF
```

Replace **`YOUR_STORAGE_CLASS`** before apply (or use `oc patch` / re-apply). Then:

```bash
oc wait --for=condition=ready pod -l app.kubernetes.io/instance=splunk-lab-standalone -n splunk-demo --timeout=600s
```

**Route** (Splunk Web = port **8000**); add **edge TLS** if HTTPS returns 503 while HTTP works:

```bash
oc expose svc splunk-lab-standalone-service -n splunk-demo --port=8000 --name=splunk-web
oc patch route splunk-web -n splunk-demo -p '{"spec":{"tls":{"termination":"edge","insecureEdgeTerminationPolicy":"Redirect"}}}'
```

**Admin password** (run locally; **do not** commit output):

```bash
oc get secret splunk-lab-standalone-secret-v1 -n splunk-demo -o go-template='{{index .data "password" | base64decode}}{{println}}'
```

## RHACS Technology Add-on install (after user download)

User must download from [Splunkbase app 5315](https://splunkbase.splunk.com/app/5315). Use the actual filename Splunkbase provides.

```bash
TA_LOCAL=/path/to/red-hat-advanced-cluster-security-splunk-technology-add-on.tgz
NS=splunk-demo
POD=splunk-lab-standalone-0
REMOTE=/tmp/rhacs_ta.tgz
oc cp "$TA_LOCAL" "${NS}/${POD}:${REMOTE}"
ADMIN_PASS="$(oc get secret splunk-lab-standalone-secret-v1 -n "$NS" -o jsonpath='{.data.password}' | base64 -d)"
oc exec -n "$NS" "$POD" -- /opt/splunk/bin/splunk install app "$REMOTE" -auth "admin:${ADMIN_PASS}" -update 1
oc exec -n "$NS" "$POD" -- /opt/splunk/bin/splunk restart
```

Installed app directory is commonly **`TA-stackrox`**. Settings file:

`/opt/splunk/etc/apps/TA-stackrox/local/ta_stackrox_settings.conf`

## TA troubleshooting

```bash
oc exec -n splunk-demo splunk-lab-standalone-0 -- ls -la /opt/splunk/var/log/splunk/stackrox*.log
oc exec -n splunk-demo splunk-lab-standalone-0 -- tail -100 /opt/splunk/var/log/splunk/stackrox_violations_*.log
```

## Discover RHACS Central route

```bash
oc get routes -A | grep -i central
```

Use the **public Central hostname** Splunk can reach from inside the cluster (or the correct internal service name if your topology requires it).
