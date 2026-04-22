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

## Splunkbase add-on (prerequisite for “Install the TA for me”)

1. Log into [Splunkbase](https://splunkbase.splunk.com/app/5315) and download the **Red Hat Advanced Cluster Security** Splunk add-on as a **`.tgz`**.
2. On the **same machine where you run `oc`**, create **`~/code/Splunk/`** if it does not exist and **save the `.tgz` there** (keep Splunkbase’s filename, including any version suffix such as `_204`).
3. For the minimal **Install the TA for me** prompt, the skill assumes that layout: **one** Splunkbase add-on **`.tgz`** under **`~/code/Splunk/`**. If the file is missing, it lives outside that folder, or more than one **`.tgz`** there could be the wrong package, the agent asks once for the exact path.

This repository does not ship the **`.tgz`** (Splunkbase login and license terms apply).

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

Say these **in order** in **Agent** chat. They are enough **when this skill is in scope** for the session (skill enabled for the project, `@` mention to the skill if your Cursor build supports it, or you already asked something Splunk/OpenShift–related so the agent loaded **`SKILL.md`**). The agent follows lab **defaults** from **`REFERENCE.md`**: namespace **`splunk-demo`**, Splunk pod **`splunk-lab-standalone-0`**, Standalone sizing and **Splunk Free**; it chooses **StorageClass** from **`oc get storageclass`** (typically the default) unless you overrode that earlier. Prompt **3** expects the Splunkbase **`.tgz`** under **`~/code/Splunk/`** as described above.

| # | Say | Outcome (defaults) |
|---|-----|--------------------|
| 1 | Run a preflight check. | Read-only **`oc` / helm** checks from **`REFERENCE.md`**—cluster version, nodes, default **StorageClass**, **SCCs** including **nonroot-v2**, and whether **CRDs** can be created. **Nothing is installed.** |
| 2 | Install Splunk for me. | **Splunk Operator** (Helm) + **Standalone** lab CR + **Route** (with **edge TLS** per the skill). You get **Splunk Web** URL and the **`oc`** line to read **admin** from **`splunk-lab-standalone-secret-v1`**. |
| 3 | Install the TA for me. | **`oc cp`** from **`~/code/Splunk/*.tgz`** (the Splunkbase add-on you placed there), then **`splunk install app`** and **`splunk restart`**. After restart, **Apps** lists **Red Hat Advanced Cluster Security**. |

Then configure **Central**, **API token**, and **Violations** in Splunk Web using **`SKILL.md`**. Do not commit **tokens** or **passwords** to this repo.

## After install: where to find **Red Hat Advanced Cluster Security**

When **`splunk install app`** finishes and Splunk has restarted, open **Splunk Web**. In the **Apps** sidebar you should see **Red Hat Advanced Cluster Security** (same place as Search & Reporting and other apps). Select it to open **Configuration** (Central + API token) and **Inputs → Violations**.

![Splunk Web Apps sidebar listing Red Hat Advanced Cluster Security](docs/splunk-apps-red-hat-advanced-cluster-security.png)

That screen is your confirmation the Splunkbase package is installed. The next step is **configuration** inside that app—not more CLI—until you troubleshoot with **`oc exec`** and log files under **`/opt/splunk/var/log/splunk/`** if something fails.

**Splunk UI name vs directory on the server:** Splunk installs every app under **`/opt/splunk/etc/apps/<directory>/`**. For this Splunkbase package the directory name is **`TA-stackrox`**, while the product title in the UI is **Red Hat Advanced Cluster Security**. You use the friendly name in the browser; **`TA-stackrox`** shows up in paths such as **`/opt/splunk/etc/apps/TA-stackrox/...`** and in some log filenames—handy for support and **`oc exec`**, not something you say in the three prompts.

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
