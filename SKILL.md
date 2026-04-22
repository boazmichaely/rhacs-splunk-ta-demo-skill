---
name: rhacs-splunk-ta-demo-skill
description: >-
  Deploys a lab Splunk Enterprise instance on OpenShift with the Splunk Operator
  (Splunk Free, single Standalone) and guides RHACS integration via the Splunkbase
  Technology Add-on (app 5315). Runs cluster preflight checks, Helm install with
  OpenShift SCC bindings, Route + TLS, TA install, and Violations input verification.
  Use when the user asks to build a Splunk demo on a cluster, deploy Splunk on
  OpenShift, splunk-demo, Splunk Operator lab, RHACS/ACS Splunk TA demo, or
  connect RHACS to Splunk using the Red Hat ACS Splunk Technology Add-on.
  Matches short requests such as run a preflight check, install Splunk for me,
  or install the TA for me (Splunk OpenShift lab defaults).
---

# Splunk lab on OpenShift (+ RHACS TA)

## When this skill applies

User wants a **non-production** Splunk on **OpenShift** they can log into, with **RHACS/ACS** via the **Splunkbase TA** ([app 5315](https://splunkbase.splunk.com/app/5315)).

## Response pattern

1. **Short plan** (condensed): namespace → CRDs → Helm operator (EULA + image pin) → SCC → `Standalone` (Free license) → Route → RHACS TA → Violations input → verify search.
2. **Preconditions**: confirm `oc` context; user is logged in; intent (lab only). If deploy requested, confirm **cluster-admin** or sufficient rights for CRDs/cluster-scope Helm.
3. **Preflight** (execute read-only checks; report blockers): identity, OCP/Kube version, nodes + allocatable CPU/RAM, default `StorageClass`, `helm`, SCCs (`nonroot-v2` on OCP 4.14+), metrics if available. Note **Splunk Operator** quirks: CRD bundle may be GitHub **3.0.0** while Helm chart is **3.1.x**; **operator image** tag **3.1.0** may be missing on Docker Hub — pin **`docker.io/splunk/splunk-operator:3.0.0`** when pulls fail.
4. **Deploy** (only if user asks to proceed): follow the command sequence in [REFERENCE.md](REFERENCE.md); use namespace **`splunk-demo`** unless user overrides; patch Route **edge TLS** if `https://` returns 503 while `http://` works.
5. **RHACS TA**: Splunkbase is **login-gated** — user downloads the `.tgz`. For a minimal **“Install the TA for me”** ask, assume **`~/code/Splunk/`** on the **`oc` workstation** (README): use the Splunkbase **`.tgz`** there; if missing or ambiguous, ask once for the full path. Then `oc cp` + `splunk install app` + `splunk restart` (see REFERENCE). **Never log or commit** the Splunk `admin` password or RHACS API token.
6. **TA configuration (Violations)**:
   - **Configuration** tab: **Central endpoint** = **hostname:443** only (example pattern: `central-<something>.apps.<cluster-apps-domain>:443`). Do **not** prefix `https://` in the field unless the TA explicitly requires it—the add-on typically builds `https://` when calling Central.
   - **API token**: RHACS token with broad **read** access (e.g. **Analyst** role per Red Hat docs). Obtain from the user; do not embed in repo files.
   - **Inputs → Violations**: keep **defaults**—interval **60s**, index **`default`**, from checkpoint **`2000-01-01T00:00:00.000Z`** (UTC).
   - **UI “Status: false”** is often misleading; trust **TA debug logs** and **search results**.
7. **Verify**:
   - TA log on Splunk pod: `/opt/splunk/var/log/splunk/stackrox_violations_*.log` — look for `ERROR` and successful GETs to `/api/splunk/ta/violations`.
   - **First search**: `index=* sourcetype="stackrox*"` (or `index=default` / `index=main` if events land there).
8. **Docs**: [RHACS Splunk integration](https://docs.openshift.com/acs/4.6/integration/integrate-with-splunk.html); TA compatibility on Splunkbase may list **9.x**; **Splunk 10** may still work — note install risk briefly.

**Splunk Free:** multi-user roles are limited; sharing **`admin`** is the typical lab pattern—retrieve password only via `oc` from the user’s cluster; never store it in git.

## Do not

- Commit or paste **passwords**, **kubeconfigs**, **HEC tokens**, **RHACS API tokens**, or **customer hostnames** into the skill repo or chat logs intended for publication.
- Assume Splunkbase downloads work without user login.
- Put `https://` into **Central endpoint** if the TA expects `host:443` (confirm field help; TA logs should show `https://` on the wire).
- Recommend `*` as the **Index** value for the modular input destination.

## Optional project folder

Users may keep downloads and helper scripts under a personal directory (e.g. `~/code/Splunk/`). Not part of this skill repository.
