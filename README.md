# RHACS Splunk TA demo (Cursor skill)

## What this skill does

Cursor skill for a **lab Splunk** on **OpenShift** (Splunk Operator, Standalone, Splunk Free) with **RHACS** via the [Splunkbase TA (5315)](https://splunkbase.splunk.com/app/5315): preflight, install, Route/TLS, TA install, Violations input, quick checks. Not for production.

## Cluster requirements

- An OpenShift cluster you may use for a **lab** (not production).
- **`cluster-admin`** on that cluster for the identity you use with **`oc login`** (installing the Splunk Operator pulls CRDs and cluster-scoped objects).
- **RHACS (ACS)** Central already installed and reachable from workloads on the cluster (typical: a public **Route** for Central).

**Sizing and storage** (matches the lab `Standalone` in `REFERENCE.md`; adjust if pods won’t schedule):

- **PVCs:** two **ReadWriteOnce** volumes—**10Gi** (Splunk `etc`) and **20Gi** (`var`). You need a **StorageClass** that can provision them (`oc get storageclass`).
- **Splunk pod:** **500m–2** CPUs and **4–6Gi** memory (requests/limits in the reference manifest). Budget extra on the cluster for the Splunk Operator controller.

**Splunkbase:** account to download the TA from [https://splunkbase.splunk.com/app/5315](https://splunkbase.splunk.com/app/5315).

## RHACS API token (prerequisite for the TA)

The Splunk TA talks to **RHACS Central** over HTTPS using an **API token** (sometimes called an API key in UIs). Create it **before** you paste it into the TA’s Configuration screen.

1. Open **RHACS Central** in a browser (the same instance your cluster’s workloads should reach—often the OpenShift **Route** hostname for Central).
2. Open your **user menu** (avatar / profile area) and go to **user profile** or **API tokens** (exact labels vary slightly by ACS version).
3. **Create** a new token, give it a recognizable name (e.g. `splunk-ta-lab`), and **copy the value once** when shown; Central will not show it again.
4. Use a token whose **role** can **read** the data the TA pulls (Red Hat’s Splunk integration doc uses an **Analyst**-class token as the example). If the TA cannot query violations, create a token tied to a user/role with sufficient read access and try again.

You will paste this token only into **Splunk’s TA configuration** (or tell the agent the token **in chat** if you accept that risk)—never into this git repo, never into a committed file. See [Red Hat ACS — Integrating with Splunk](https://docs.openshift.com/acs/4.6/integration/integrate-with-splunk.html) for product-specific detail.

## What you need locally

- **Cursor** (Agent / skills)
- **`oc`** logged into the cluster
- **`helm`** (Splunk Operator is installed with Helm)

You do not need to publish this repo to use it.

## Install the skill

1. Clone or copy the repo ([https://github.com/boazmichaely/rhacs-splunk-ta-demo-skill](https://github.com/boazmichaely/rhacs-splunk-ta-demo-skill)).
2. Place it under `~/.cursor/skills/` so `SKILL.md` is at the folder root, with `REFERENCE.md` beside it.
3. Restart Cursor (or reload the window).

## Use the skill — example Agent prompts

Say the lines below (or close paraphrases) in **Cursor Agent** chat after the skill is installed. The agent uses `SKILL.md` plus command blocks from `REFERENCE.md`.

| Say this | You get | What that gives you |
|----------|---------|----------------------|
| “Run **read-only preflight** for the Splunk-on-OpenShift lab from my rhacs-splunk-ta-demo-skill; don’t install anything yet.” | A short checklist driven by `oc`/`helm` (identity, version, nodes, default **StorageClass**, SCCs, CRD rights). | You learn whether the cluster is likely to succeed **before** you apply CRDs or Helm; surfaces missing **nonroot-v2**, storage, or permissions early. |
| “**Deploy lab Splunk** in namespace `splunk-demo` following REFERENCE.md: operator, Standalone (Free), Route. I’m cluster-admin and want the default lab sizing.” | Ordered **Helm + CR + Route** steps with your **StorageClass** substituted where needed. | A running **Splunk Web** URL (after pods go Ready) and commands to read the **admin** password from a **Secret** (not stored in git). |
| “Splunk **HTTPS** on the Route gives **503** but HTTP works—apply the **edge TLS** fix from the skill.” | `oc patch` (or equivalent) on the **Route** so termination is **edge** with redirect. | Browser users hit **https://** reliably instead of a broken TLS edge case common right after `oc expose`. |
| “I saved the RHACS TA **.tgz** at `~/Downloads/…tgz`. **Install it** on pod `splunk-lab-standalone-0` in `splunk-demo` and restart Splunk.” | **`oc cp`**, **`splunk install app`**, **`splunk restart`** sequence aligned with REFERENCE.md. | TA appears under **Apps** in Splunk; you can open **TA-stackrox** to enter **Central endpoint** + **API token** + enable **Violations**. |
| “What **Central hostname** format and **Violations** defaults should I use in the TA? I’ll paste the token only in Splunk.” | Field-level guidance from the skill (**host:443**, no `https://` prefix; default interval/index/checkpoint). | Correct TA **Configuration** and **Inputs → Violations** so the modular input can call **`/api/splunk/ta/violations`** without misformatted URLs or bad index values. |
| “**Verify** the TA is ingesting RHACS violations—what logs and Splunk search should I run?” | Paths under `/opt/splunk/var/log/splunk/` on the Splunk pod plus example **`index=` / `sourcetype=`** searches. | Confirms HTTP success vs **ERROR** lines in TA logs and whether events land in Splunk for dashboards or ad-hoc search. |

Replace paths, namespace, and pod name if yours differ. Do not paste **RHACS tokens** or **Splunk admin passwords** into the repository.

## Splunk Web: Home vs Edge Processor “First-time setup”

On **Splunk 10** (and recent Enterprise), **Home** can open an **Edge Processor** first-time page. **Cancel** often sends you back to the same screen because the launcher still wants that flow.

**Use Splunk without that wizard:** open **Search & Reporting** directly:

`https://<your-splunk-route-host>/en-US/app/search/search`

or **Apps → Search & Reporting**. Optionally set **Settings → User preferences → Default application** to **Search & Reporting** so you land in Search after login. You do **not** need Edge Processor for the RHACS TA lab.

## Install the RHACS TA (CLI summary)

Download the `.tgz` from [Splunkbase app 5315](https://splunkbase.splunk.com/app/5315). On the machine where **`oc`** runs, copy the archive into pod **`splunk-lab-standalone-0`** in **`splunk-demo`**, install with **`splunk install app`**, then **`splunk restart`**—full commands and variables are in **`REFERENCE.md`** (RHACS Technology Add-on section). Example: set `TA_LOCAL` to your file path, then run the `oc cp` / `oc exec` … `install app` / `restart` block.

## Files

- **`SKILL.md`** — Workflow and TA/verification guidance.
- **`REFERENCE.md`** — Command blocks (`oc`, Helm, etc.).

## References

- [Red Hat ACS — Integrating with Splunk](https://docs.openshift.com/acs/4.6/integration/integrate-with-splunk.html)
- [Splunk Operator for Kubernetes](https://splunk.github.io/splunk-operator/)
- [Splunkbase — RHACS Technology Add-on](https://splunkbase.splunk.com/app/5315)
