# RHACS Splunk TA demo (Cursor skill)

## What this skill does

Cursor skill for a **lab Splunk** on **OpenShift** (Splunk Operator, Standalone, Splunk Free) with **RHACS** via the [Splunkbase TA (5315)](https://splunkbase.splunk.com/app/5315): preflight, install, Route/TLS, TA install, Violations input, quick checks. Not for production.

## Cluster requirements

- An OpenShift cluster you may use for a **lab** (not production).
- **`cluster-admin`** on that cluster for the identity you use with **`oc login`** (installing the Splunk Operator pulls CRDs and cluster-scoped objects).

**Sizing and storage** (matches the lab `Standalone` in `REFERENCE.md`; adjust if pods won’t schedule):

- **PVCs:** two **ReadWriteOnce** volumes—**10Gi** (Splunk `etc`) and **20Gi** (`var`). You need a **StorageClass** that can provision them (`oc get storageclass`).
- **Splunk pod:** **500m–2** CPUs and **4–6Gi** memory (requests/limits in the reference manifest). Budget extra on the cluster for the Splunk Operator controller.

For the RHACS add-on: a Splunkbase account to download the TA from [https://splunkbase.splunk.com/app/5315](https://splunkbase.splunk.com/app/5315), RHACS Central reachable from the cluster, and an API token for the TA.

## What you need locally

- **Cursor** (Agent / skills)
- **`oc`** logged into the cluster
- **`helm`** (Splunk Operator is installed with Helm)

You do not need to publish this repo to use it.

## Install the skill

1. Clone or copy the repo.
2. Place it under `~/.cursor/skills/` so `SKILL.md` is at the folder root, with `REFERENCE.md` beside it.
3. Restart Cursor (or reload the window).

## Use the skill

In Agent chat, say what you want (preflight only, full deploy, TA, TLS, etc.). The agent follows `SKILL.md` and pulls commands from `REFERENCE.md`. Use your cluster’s storage class and paths; do not commit passwords or tokens.

## Files

- **`SKILL.md`** — Workflow and TA/verification guidance.
- **`REFERENCE.md`** — Command blocks (`oc`, Helm, etc.).

## References

- [Red Hat ACS — Integrating with Splunk](https://docs.openshift.com/acs/4.6/integration/integrate-with-splunk.html)
- [Splunk Operator for Kubernetes](https://splunk.github.io/splunk-operator/)
- [Splunkbase — RHACS Technology Add-on](https://splunkbase.splunk.com/app/5315)
