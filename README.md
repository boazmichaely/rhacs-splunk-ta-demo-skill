# rhacs-splunk-ta-demo-skill

Cursor **Agent Skill** for standing up a **lab Splunk** instance on **OpenShift** (Splunk Operator + Standalone, Splunk Free) and wiring **Red Hat Advanced Cluster Security (RHACS/ACS)** using the official [**Splunkbase Technology Add-on** (app 5315)](https://splunkbase.splunk.com/app/5315).

This repository intentionally contains **no credentials, tokens, cluster hostnames, or customer-specific data**—only procedural guidance and generic command examples.

## Install the skill in Cursor

Copy this folder (or clone the repo) so Cursor can load `SKILL.md`:

- **Personal (all projects):** `~/.cursor/skills/rhacs-splunk-ta-demo-skill/SKILL.md`  
  Recommended layout:
  ```text
  ~/.cursor/skills/rhacs-splunk-ta-demo-skill/
    SKILL.md
    REFERENCE.md
  ```

Restart Cursor or reload if skills are cached. The skill description in `SKILL.md` frontmatter drives when the agent applies it.

## Contents

| File | Purpose |
|------|---------|
| `SKILL.md` | Workflow: preflight → deploy → TA → Violations input → verification |
| `REFERENCE.md` | Copy-paste command blocks (placeholders for storage class, namespace, paths) |

## Security & contributions

- **Do not** commit Splunk/RHACS passwords, API tokens, HEC tokens, kubeconfigs, or internal URLs.
- Before opening a PR or pushing, search the diff for accidental secrets (passwords, bearer tokens, company domains).
- Upsell **Splunk General Terms** / **Splunkbase** / **Red Hat** licensing applies to upstream software; this repo is documentation only.

## References

- [Red Hat ACS — Integrating with Splunk](https://docs.openshift.com/acs/4.6/integration/integrate-with-splunk.html)
- [Splunk Operator for Kubernetes](https://splunk.github.io/splunk-operator/)
- [Splunkbase — RHACS TA](https://splunkbase.splunk.com/app/5315)
