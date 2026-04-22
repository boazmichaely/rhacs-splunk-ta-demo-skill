---
name: rhacs-splunk-ta-demo-skill
description: >-
  Deploys a lab Splunk Enterprise instance on OpenShift with the Splunk Operator
  (Splunk Free, single Standalone) and guides RHACS integration via the Splunkbase
  Technology Add-on (app 5315). Runs cluster preflight checks, Helm install with
  OpenShift SCC bindings, Route + TLS, TA install, and modular input verification (Compliance, Vuln, Violations).
  Use when the user asks to build a Splunk demo on a cluster, deploy Splunk on
  OpenShift, splunk-demo, Splunk Operator lab, RHACS/ACS Splunk TA demo, or
  connect RHACS to Splunk using the Red Hat ACS Splunk Technology Add-on.
  Matches short requests such as run a preflight check, install Splunk for me,
  or install the TA for me (Splunk OpenShift lab defaults).
---

# Splunk lab on OpenShift (+ RHACS TA)

**Naming:** The Splunkbase add-on is listed in Splunk as **Red Hat Advanced Cluster Security**; on-disk paths and logs often use the directory **`TA-stackrox`**. In this file, **TA** means that Splunkbase add-on.

## When this skill applies

User wants a **non-production** Splunk on **OpenShift** they can log into, with **RHACS/ACS** via the **Splunkbase TA** ([app 5315](https://splunkbase.splunk.com/app/5315)).

## Response pattern

1. **Short plan** (condensed): namespace → CRDs → Helm operator (EULA + image pin) → SCC → `Standalone` (Free license) → Route → install TA → Splunk UI: Central + token → modular inputs (**ACS Compliance**, **ACS Vulnerability Management**, **ACS Violations**) → verify search.
2. **Preconditions**: confirm `oc` context; user is logged in; intent (lab only). If deploy requested, confirm **cluster-admin** or sufficient rights for CRDs/cluster-scope Helm.
3. **Preflight** (execute read-only checks; report blockers): identity, OCP/Kube version, nodes + allocatable CPU/RAM, default `StorageClass`, `helm`, SCCs (`nonroot-v2` on OCP 4.14+), metrics if available. Note **Splunk Operator** quirks: CRD bundle may be GitHub **3.0.0** while Helm chart is **3.1.x**; **operator image** tag **3.1.0** may be missing on Docker Hub — pin **`docker.io/splunk/splunk-operator:3.0.0`** when pulls fail.
4. **Deploy** (only if user asks to proceed): follow the command sequence in [REFERENCE.md](REFERENCE.md); use namespace **`splunk-demo`** unless user overrides; patch Route **edge TLS** if `https://` returns 503 while `http://` works.
5. **RHACS TA**: Splunkbase is **login-gated** — user downloads the `.tgz`. For a minimal **“Install the TA for me”** ask, assume **`~/code/Splunk/`** on the **`oc` workstation** (README): use the Splunkbase **`.tgz`** there; if missing or ambiguous, ask once for the full path. Then `oc cp` + `splunk install app` + `splunk restart` (see REFERENCE). **Never log or commit** the Splunk `admin` password or RHACS API token.
6. **TA configuration (Splunk UI)** — **Red Hat Advanced Cluster Security** app:
   - **Configuration → Add-on Settings**: **Central endpoint** = **hostname:443** (examples: `central-<something>.apps.<cluster-apps-domain>:443`, or the hostname from the cluster’s Central **Route**—use **`oc get routes`** per **REFERENCE.md** if unsure). Do **not** prefix `https://` unless the add-on field help requires it; the add-on typically builds `https://` when calling Central.
   - **API token**: Create in RHACS Central under **Integrations → Authentication**; use **Analyst** (read access). User copies the key when shown; do not embed in repo files. RHACS menu labels can differ slightly by ACS version.
   - **Create New Input**: add **ACS Compliance**, **ACS Vulnerability Management**, and **ACS Violations** (one wizard each). For **Compliance** and **Vulnerability Management**, keep the add-on **defaults** unless the user has a reason to change interval/index/checkpoint.
   - **ACS Violations** defaults (lab): interval **60s**, index **`default`**, from checkpoint **`2000-01-01T00:00:00.000Z`** (UTC), checkpoint type **Auto** when offered.
   - **UI “Status: false”** is often misleading; trust **TA debug logs** and **search results**.
7. **Verify**:
   - **Violations:** on the Splunk pod, `/opt/splunk/var/log/splunk/stackrox_violations_*.log` — look for `ERROR` and successful GETs to `/api/splunk/ta/violations`.
   - **Compliance / Vulnerability Management:** other **`stackrox*.log`** files may appear under `/opt/splunk/var/log/splunk/`—list and tail the file that matches the input under test (see **REFERENCE.md** TA troubleshooting).
   - **Search**: `index=* sourcetype="stackrox*"` (or `index=default` / `index=main` if events land there); adjust `sourcetype` if the TA uses different names for non-violations data.
8. **Docs**: [RHACS Splunk integration](https://docs.openshift.com/acs/4.6/integration/integrate-with-splunk.html) — use the doc revision that matches the user’s ACS version when possible; TA compatibility on Splunkbase may list **9.x**; **Splunk 10** may still work — note install risk briefly.

**Splunk Free:** multi-user roles are limited; sharing **`admin`** is the typical lab pattern—retrieve password only via `oc` from the user’s cluster; never store it in git.

## Do not

- Commit or paste **passwords**, **kubeconfigs**, **HEC tokens**, **RHACS API tokens**, or **customer hostnames** into the skill repo or chat logs intended for publication.
- Assume Splunkbase downloads work without user login.
- Put `https://` into **Central endpoint** if the TA expects `host:443` (confirm field help; TA logs should show `https://` on the wire).
- Recommend `*` as the **Index** value for the modular input destination.

## Optional project folder

Users may keep downloads and helper scripts under a personal directory (e.g. **`~/code/Splunk/`**). That path matches the **README** default for **Install the TA for me**. Not part of this skill repository.
