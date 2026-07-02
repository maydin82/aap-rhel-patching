# RHEL 7/8/9/10 Patching Automation (AAP)

## Layout

```
playbooks/
  01_precheck.yml          # registration check/re-register, optional service precheck
  02_stop_services.yml     # optional service stop
  03_patch_and_reboot.yml  # yum/dnf update (excludes docker), unconditional reboot
  04_posthealthcheck.yml   # wait for host, re-verify registration, verify services
  site.yml                 # all four stages in one playbook (single job template)
inventory/
  group_vars/all.yml           # shared Satellite vars (url, org id, JWT ref)
  group_vars/webservers.yml    # satellite_activation_key for this group
  group_vars/dbservers.yml     # satellite_activation_key for this group
  host_vars/example_webserver.yml
  host_vars/example_dbserver.yml
```

## Two ways to wire this into AAP

**Option A - Workflow Job Template (recommended)**
Create four Job Templates, one per playbook (01-04), all pointing at the
same Project/inventory. Chain them in a Workflow Job Template:

```
01_precheck  --(on success)-->  02_stop_services  --(on success)-->
03_patch_and_reboot  --(on success)-->  04_posthealthcheck
```

This gives you per-stage status in the workflow visualizer, the ability to
insert an **approval node** before step 3 (patch+reboot), and the ability
to branch differently on failure (e.g. notify on precheck failure without
touching the host).

**Option B - Single Job Template**
Point one Job Template at `playbooks/site.yml`. Simpler, but you lose
per-stage visibility/approval gates.

## Variables

| Variable | Where set | Purpose |
|---|---|---|
| `target_hosts` | survey / extra_vars | inventory pattern to run against (default `all`) |
| `stop_services_before_patch` | group_vars/host_vars/survey | bool, gates step 2 |
| `exclude_packages` | group_vars/host_vars | list of glob patterns excluded from update (default `docker*`, `containerd*`) |
| `patch_services` | host_vars (per host) | list of service dicts, see below - entirely optional |
| `satellite_url` | AAP survey or extra_vars | Satellite base URL, e.g. `https://satellite.example.com` |
| `satellite_organization_id` | group_vars/all.yml or survey | Satellite org ID, shared across the run |
| `satellite_registration_jwt` | AAP credential (secret) | JWT from Satellite's Hosts > Register Host > Generate (per Global Registration template) |
| `satellite_activation_key` | **group_vars, per group** | activation key for that group of hosts - see "Per-group activation keys" below |
| `satellite_registration_insecure` | optional | bool, adds `--insecure` to the curl call for the initial CA fetch |
| `satellite_location_id` / `satellite_hostgroup_id` | optional | passed as extra query params on the registration call if needed |

### `patch_services` item shape

```yaml
patch_services:
  - name: nginx              # required - systemd unit name (without .service)
    type: webserver           # generic | webserver | database
    url: "https://host/healthz"   # webserver only - if omitted, only systemd state is checked
    validate_certs: false     # webserver only, optional

  - name: postgresql-15
    type: database
    db_type: postgres         # postgres | mysql
    db_host: localhost
    db_port: 5432

  - name: myapp
    type: generic              # just systemd started/stopped, nothing else
```

Any host with no `patch_services` defined simply skips steps 2's service
stop and step 4's service-level checks - only the package update and
reboot happen. Nothing is assumed or enforced that wasn't configured.

### Per-group activation keys

If your inventory has separate groups (e.g. `webservers`, `dbservers`)
where every host in a group shares the same Satellite activation key,
set `satellite_activation_key` in that group's `group_vars` file instead
of per-host or in the playbook:

```yaml
# inventory/group_vars/webservers.yml
satellite_activation_key: "rhel-webservers-prod"

# inventory/group_vars/dbservers.yml
satellite_activation_key: "rhel-dbservers-prod"
```

Ansible's normal variable precedence merges the right key into each host
automatically based on group membership - the playbook itself has no
`when: group == ...` branching. `satellite_url`, `satellite_organization_id`,
and `satellite_registration_jwt` stay in `group_vars/all.yml` (or survey/
extra_vars) since those are the same for the whole run.

If your inventory is a **dynamic/synced source** in AAP rather than a
static file (cloud plugin, CMDB, etc.), you can't drop a `group_vars/`
directory next to it - instead set `satellite_activation_key` directly on
the corresponding **Group** object in the AAP inventory UI
(Inventories → inventory → Groups → select group → Variables).

The registration task builds the actual curl call as:

```
curl -sS 'https://satellite.example.com/register?activation_keys=<key>&organization_id=<id>' \
  -H 'Authorization: Bearer <jwt>' | bash
```

matching the format Satellite itself generates from Hosts > Register Host.

## Design notes

- **RHEL 7** uses the `yum` module; **RHEL 8/9/10** use the `dnf` module
  (RHEL 10 ships dnf5 with `yum` only as a compatibility shim, so `dnf` is
  the safe choice from 8 onward). Both branches honor the same
  `exclude_packages` list.
- Reboot (step 3) always runs, regardless of whether the update reported
  changes - as requested.
- Registration re-check happens both pre-patch (with re-register attempt)
  and post-reboot, so a failed re-registration or subscription drop after
  reboot is visible in the job output.
- Registration uses Satellite's **Global Registration** curl/JWT method
  (Hosts > Register Host in the Satellite UI generates the token). The JWT
  is short-lived (its lifetime is configurable when generated) and scoped
  only to the registration endpoint, so it's a good fit for an AAP
  credential that gets refreshed per run rather than a long-lived secret.
- Credentials (Satellite JWT, DB passwords) should be stored as AAP
  Credentials or Vault-encrypted variables, not committed to host_vars in
  plaintext - the example host_vars files show where they'd be referenced.
- Store secrets, this repo assumes a standard AAP Project pointing at a git
  repo containing this structure, plus a Vault credential attached to the
  Job Template(s) if you encrypt any of the above.
