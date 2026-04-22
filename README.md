# RHACS Splunk TA demo (Cursor skill)

## What you build

This skill helps you stand up a **small Splunk Enterprise lab on OpenShift** and wire it to **Red Hat Advanced Cluster Security (RHACS / ACS)** using the official Splunkbase package ([app 5315](https://splunkbase.splunk.com/app/5315)).

There are **two** concrete outcomes:

1. **Splunk on the cluster** — Splunk Operator (Helm), a **Standalone** instance (Splunk Free), **PVCs**, and a **Route** to Splunk Web so people can log in with a browser.
2. **Red Hat Advanced Cluster Security in Splunk** — the Splunkbase **.tgz** is installed into Splunk; in the UI the add-on appears under **Apps** as **Red Hat Advanced Cluster Security**. You then point it at **RHACS Central** (hostname + port), paste an **API token**, and turn on the **Violations** input so events show up in search.

Everything here is for **learning and demos**, not a production hardening guide.

## Cluster requirements

- OpenShift cluster you may use for a **lab** (not production).
- **`cluster-admin`** for the identity you use with **`oc login`** (operator install uses cluster-scoped **CRDs**).
- **RHACS Central** already on the cluster and reachable the way your Splunk pods will need (often a public **Route**).

**Sizing and storage** (see **`REFERENCE.md`** Standalone manifest):

- **PVCs:** **10Gi** + **20Gi** ReadWriteOnce for Splunk **etc** / **var**; pick a **StorageClass** that provisions them (`oc get storageclass`).
- **Splunk pod:** about **500m–2** CPUs and **4–6Gi** memory in the reference example; leave headroom for the operator.

**Splunkbase:** account to download the add-on from [https://splunkbase.splunk.com/app/5315](https://splunkbase.splunk.com/app/5315).

## RHACS API token (before you configure the add-on)

The Splunk add-on calls **RHACS Central** over HTTPS with an **API token** (some UIs say “API key”).

1. Log into **RHACS Central** in a browser.
2. From your **user / profile** area, open **API tokens** (labels vary slightly by ACS version).
3. **Create** a token, name it (e.g. `splunk-lab`), **copy it once** when shown.
4. Prefer a role with **read** access to the data the add-on queries (Red Hat’s Splunk integration example uses an **Analyst**-style token).

Paste the token only into **Splunk** (or share with the agent in chat if you accept that)—never commit it to git. Details: [Red Hat ACS — Integrating with Splunk](https://docs.openshift.com/acs/4.6/integration/integrate-with-splunk.html).

## What you need locally

- **Cursor** (Agent / skills)
- **`oc`** logged into the cluster
- **`helm`**

## Install this repository as a Cursor skill

1. Clone: [https://github.com/boazmichaely/rhacs-splunk-ta-demo-skill](https://github.com/boazmichaely/rhacs-splunk-ta-demo-skill)
2. Copy or symlink so **`SKILL.md`** and **`REFERENCE.md`** sit together under e.g. `~/.cursor/skills/rhacs-splunk-ta-demo-skill/`.
3. Restart Cursor (or reload the window).

## Agent flow (three prompts)

In **Agent** chat, name this skill (**rhacs-splunk-ta-demo-skill** / Splunk-on-OpenShift lab) so the right **`SKILL.md`** applies. The agent runs or prints commands from **`REFERENCE.md`**. Use these **three** asks **in order**; substitute your **StorageClass** / **namespace** / **.tgz** path / pod name when they differ from the lab defaults.

| # | Say | Outcome |
|---|-----|---------|
| 1 | “**Preflight only** for the Splunk lab in **rhacs-splunk-ta-demo-skill**—read-only checks, **do not install**.” | `oc` / **helm** checks (version, nodes, **StorageClass**, **SCCs**, CRD rights). |
| 2 | “**Install Splunk** for that lab per **REFERENCE.md** (namespace **`splunk-demo`**, default Standalone sizing).” | Operator + **Standalone** + **Route**; **Splunk Web** URL + **`oc`** to read **admin** password from **Secret**. |
| 3 | “**Install the Splunkbase TA**: tarball **`/your/path/add-on.tgz`**, namespace **`splunk-demo`**, pod **`splunk-lab-standalone-0`**.” | **`oc cp`**, **`splunk install app`**, **`splunk restart`** → **Apps** shows **Red Hat Advanced Cluster Security** (on disk often **`TA-stackrox`**). |

Then configure **Central**, **API token**, and **Violations** in Splunk Web using **`SKILL.md`**. Do not commit **tokens** or **passwords** to this repo.

## After install: where to find **Red Hat Advanced Cluster Security**

When **`splunk install app`** finishes and Splunk has restarted, open **Splunk Web**. In the **Apps** sidebar you should see **Red Hat Advanced Cluster Security** (same place as Search & Reporting and other apps). Select it to open **Configuration** (Central + API token) and **Inputs → Violations**.

![Splunk Web Apps sidebar listing Red Hat Advanced Cluster Security](docs/splunk-apps-red-hat-advanced-cluster-security.png)

That screen is your confirmation the Splunkbase package is installed. The next step is **configuration** inside that app—not more CLI—until you troubleshoot with **`oc exec`** and log files under **`/opt/splunk/var/log/splunk/`** if something fails.

## Splunk Web: Home and Edge Processor

On **Splunk 10**, **Home** can loop on an **Edge Processor** “First-time setup” page; **Cancel** may not exit the loop. For this lab you do **not** need Edge Processor. Open **Search & Reporting** directly:

`https://<your-splunk-route-host>/en-US/app/search/search`

or **Apps → Search & Reporting**. Optionally set **Settings → User preferences → Default application** to **Search & Reporting**.

## Files in this repository

| File | Purpose |
|------|---------|
| **`SKILL.md`** | When the skill applies, full workflow, add-on UI fields, verification, guardrails. |
| **`REFERENCE.md`** | Copy-paste **`oc`** / **Helm** / Splunk CLI commands. |
| **`docs/splunk-apps-red-hat-advanced-cluster-security.png`** | Example outcome: add-on visible under **Apps**. |

## References

- [Red Hat ACS — Integrating with Splunk](https://docs.openshift.com/acs/4.6/integration/integrate-with-splunk.html)
- [Splunk Operator for Kubernetes](https://splunk.github.io/splunk-operator/)
- [Splunkbase — Red Hat Advanced Cluster Security Splunk Technology Add-on](https://splunkbase.splunk.com/app/5315)
