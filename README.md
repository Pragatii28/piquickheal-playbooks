# PiQuickHeal — Ansible/AWX playbooks

Converts the `PLAYBOOKS` dict from `app/services/playbooks.py` (Pi-OpsHeal)
into standalone Ansible playbooks, one per alert type, tagged by stage so
AWX can run diagnose separately from remediate/validate with a human
approval step in between.

## Layout

```
group_vars/all.yml       # protected_processes list, shared across all playbooks
playbooks/cpu_spike.yml
playbooks/memory_pressure.yml
# add disk_pressure.yml, cpu_iowait.yml etc. following the same pattern
```

## 1. Put this in a git repo AWX can reach

AWX pulls playbooks from a Project backed by a git repo (or SCM). Push this
folder to a repo, then in AWX:

- **Projects → Add** → SCM type: Git → point at the repo → Sync.

## 2. Inventory

- **Inventories → Add** → add your managed hosts (the same servers Datadog
  monitors). Group them however matches your fleet — by role, by
  environment, etc.
- Attach an SSH credential AWX will use to connect (Credentials → Add →
  Machine).

## 3. Job Templates — one pair per alert type

For each alert type with `requires_approval: True` in the original Python
(cpu_spike, cpu_iowait, memory_pressure, disk_pressure), create **two** Job
Templates:

| Job Template | Playbook | Tags | Notes |
|---|---|---|---|
| `piquickheal-cpu_spike-diagnose` | `playbooks/cpu_spike.yml` | `diagnose` | Always safe to auto-run, no approval needed |
| `piquickheal-cpu_spike-remediate` | `playbooks/cpu_spike.yml` | `remediate,validate` | Only runs after approval |

For alert types with `requires_approval: False` (`disk_diagnostic`,
`memory_diagnostic`, `observe_only`), a **single** Job Template with no tag
restriction is enough — nothing needs gating.

In each Job Template, set **Limit** to `{{ target_host }}` so PiQuickHeal
can pass the specific affected host at launch time via `extra_vars`,
matching how `main.py`'s `trigger_awx_remediation()` already calls
`json={"limit": hostname}`.

## 4. Workflow Templates — the approval gate

For each `requires_approval: True` alert type, build a **Workflow
Template**:

```
[Job Template: cpu_spike-diagnose]
        │
        ▼
[Approval node]   <- pauses here, notifies you, waits for a click
        │
        ▼
[Job Template: cpu_spike-remediate]
```

- **Templates → Add → Workflow Template**.
- Add the diagnose Job Template as the first node.
- Add an **Approval** node after it (set a timeout if you want auto-reject
  after N hours).
- Add the remediate Job Template as the node after approval, set to run
  **on success** of the approval.

This Workflow Template ID is what `AWX_TEMPLATE_MAP` in PiQuickHeal's
`main.py` should point to — not the plain Job Template ID — since launching
a workflow is what gives you the pause-for-approval behavior:

```python
AWX_TEMPLATE_MAP = {
    "cpu_spike": <workflow_template_id>,
    "memory_pressure": <workflow_template_id>,
    "disk_diagnostic": <job_template_id>,   # no approval needed, plain job template is fine
}
```

`trigger_awx_remediation()` in `main.py` already does `POST
/api/v2/job_templates/{id}/launch/` — swap the URL to
`/api/v2/workflow_job_templates/{id}/launch/` when the ID is a workflow, and
poll `/api/v2/workflow_jobs/{id}/` instead of `/api/v2/jobs/{id}/` for
status.

## 5. Protecting the top process (the `PROTECTED_PROCESS_FOUND` logic)

Same behavior as the Python version: Stage 1 always runs and always exits
0. If the top consumer is on the `protected_processes` list, the playbook
sets `top_process_protected: true` and publishes it via `set_stats` (AWX
surfaces this as job artifacts/stats you can read via the API). The
remediate stage tasks have `when: not top_process_protected` guards, so
even if someone approves anyway, nothing gets killed — the same
fail-safe the Python `_detect_protected_top_process()` gate provided.

If you want PiQuickHeal to *skip showing the approval step at all* when the
top process is protected (rather than showing it and having it silently
no-op), have your orchestrator check the diagnose Job Template's stats via
`GET /api/v2/jobs/{id}/stats/` before deciding whether to proceed to the
approval node — you'd swap the Workflow Template's pure linear chain for a
conditional workflow node ("run remediate only if diagnose stats say
`top_process_protected: false`").

## 6. Remaining alert types

`cpu_iowait`, `disk_pressure`, `disk_diagnostic`, `memory_diagnostic`, and
`observe_only` all follow the exact same translation pattern as
`cpu_spike.yml` and `memory_pressure.yml`:

1. Copy the `"cmd"` bash block from each stage in `playbooks.py` almost
   verbatim into a `shell:` task.
2. Tag each task by its `"phase"` (`diagnose` / `remediate`).
3. Replace `{{PROTECTED_LIST}}` with `{{ protected_processes | join(" ") }}`.
4. For diagnostic-only playbooks (`disk_diagnostic`, `memory_diagnostic`),
   skip the approval workflow entirely — they're read-only, matching
   `safe_to_auto_heal: True` / `requires_approval: False` in the source.
5. Keep the same `/var/log/auto-healer.log` append line at the end of the
   final remediate/validate stage, so existing log-based tooling or alerts
   built on that file keep working unchanged.

Want me to convert the remaining playbooks (`cpu_iowait`, `disk_pressure`,
`disk_diagnostic`, `memory_diagnostic`, `observe_only`) the same way?
