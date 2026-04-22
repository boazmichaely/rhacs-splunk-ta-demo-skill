# RHACS Splunk TA demo (Cursor skill)

## What you build

This skill helps you stand up a **small Splunk Enterprise lab on OpenShift** and wire it to **Red Hat Advanced Cluster Security (RHACS / ACS)** using the official Splunkbase package ([app 5315](https://splunkbase.splunk.com/app/5315)).

There are **two** concrete outcomes:

1. **Splunk on the cluster** — Splunk Operator (Helm), a **Standalone** instance (Splunk Free), **PVCs**, and a **Route** to Splunk Web so people can log in with a browser.
2. **Red Hat Advanced Cluster Security in Splunk** — the Splunkbase **.tgz** is installed into Splunk; in the UI the add-on appears under **Apps** as **Red Hat Advanced Cluster Security**. You then point it at **RHACS Central** (hostname + port), paste an **API token**, and turn on the **Violations** input so events show up in search.

Everything here is for **learning and demos**, not a production hardening guide.

## How you use this in Cursor

Install the skill folder under `~/.cursor/skills/` (see below). In **Agent** chat, describe what you want in plain English. The agent follows **`SKILL.md`** (workflow, TA fields, pitfalls) and runs or prints the command blocks in **`REFERENCE.md`** (`oc`, Helm, Splunk CLI). You supply cluster-specific values (storage class, paths to the downloaded **.tgz**, secrets).

## End-to-end flow

| Step | What happens |
|------|----------------|
| **1. Preflight** | Read-only checks: OpenShift version, nodes, **StorageClass**, **SCCs** (`nonroot-v2`), Helm, whether you can create **CRDs**. |
| **2. Deploy Splunk** | Create **`splunk-demo`** (or your namespace), apply Splunk Operator **CRDs**, bind SCCs, **Helm install** the operator, apply the **Standalone** CR (disk + CPU/RAM), wait for the pod, **expose** Splunk Web on a **Route** with **edge TLS** if needed. |
| **3. Log into Splunk Web** | Get the **admin** password from an OpenShift **Secret** (`REFERENCE.md`); open the **https://** Route URL. |
| **4. Install the add-on** | Download the **.tgz** from Splunkbase (login required). From the machine with **`oc`**: copy the file into **`splunk-lab-standalone-0`**, run **`splunk install app`**, then **`splunk restart`** (commands in **`REFERENCE.md`**). |
| **5. Configure RHACS** | In Splunk, open **Apps → Red Hat Advanced Cluster Security**. Set **Central endpoint**, **API token**, and **Violations** defaults per **`SKILL.md`**. |
| **6. Verify** | TA logs on the pod and simple searches (`SKILL.md`) to confirm data is flowing. |

On the Splunk **filesystem**, the add-on often lives under a directory such as **`TA-stackrox`**; Splunk still lists it in the UI under the product name **Red Hat Advanced Cluster Security**.

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

## Example Agent prompts

| Say this | You get | What that gives you |
|----------|---------|----------------------|
| “Run **read-only preflight** for the Splunk-on-OpenShift lab from my rhacs-splunk-ta-demo-skill; don’t install anything yet.” | `oc` / **helm** checks from **`REFERENCE.md`**. | Whether storage, **SCCs**, and **CRD** rights look viable **before** you install anything. |
| “**Deploy lab Splunk** in `splunk-demo` following **REFERENCE.md** (operator, Standalone Free, Route). I’m cluster-admin.” | Ordered **CRD → Helm → Standalone → Route** steps. | A **Splunk Web** URL and the **`oc`** command to read the **admin** password from a **Secret**. |
| “Splunk **HTTPS** on the Route returns **503** but HTTP works—apply the **edge TLS** fix from the skill.” | **`oc patch`** on the **Route**. | **https://** works for users behind the OpenShift edge. |
| “Install the Splunkbase **.tgz** from **`<path>`** on **`splunk-lab-standalone-0`** in **`splunk-demo`** and restart Splunk.” | **`oc cp`**, **`splunk install app`**, **`splunk restart`**. | After restart, **Apps** lists **Red Hat Advanced Cluster Security**; open it to set **Central**, token, and **Violations**. |
| “What **Central hostname** and **Violations** defaults should I use? I’ll paste the token only in Splunk.” | Field guidance from **`SKILL.md`**. | Correct **host:443** format (no stray **`https://`** in the field if the add-on builds it), sane interval / index / checkpoint. |
| “How do I **verify** violations are landing in Splunk?” | Log paths on the pod and example **searches**. | Confirms the modular input is calling Central and events are searchable. |

Replace namespace, pod name, and paths with yours. Do not put **tokens** or **passwords** in this repo.

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
