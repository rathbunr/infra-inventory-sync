# infra-inventory-sync

Reusable inventory refresh for AAP 2.7. Select one or both authoritative
inventory containers through a survey, then choose explicit refresh or native
launch-time behavior. Execution uses the maintained `ansible.controller`
lookup and update module.

## Normalized targets

The defaults in `vars/targets.yml` bind existing AAP objects:

| Survey selection | Authoritative source | Constructed inventory |
|---|---|---|
| `inventory_ad` | `source_ad_ldap` | `managed_windows` |
| `inventory_satellite` | `source_satellite` | `managed_linux` |

These are names, not fixed numeric IDs. The playbook resolves current IDs at
runtime within `inventory_sync_organization` (default `RITCSUSA`).
Constructed sources are found by source type within the selected constructed
inventory, so stale auto-created source display names do not break selection.
Legacy inventories and `managed_devices` are outside this mapping.

Override `inventory_sync_catalog` in job/node extra vars to reuse the playbook
elsewhere. Keep each mapping's inventory, source, source_type, and
constructed_inventory fields. The current model requires each constructed
inventory to have exactly its mapped authoritative inventory as its sole input.
It deliberately does not discover or refresh every legacy dependent inventory.

## Refresh modes

| Mode | Behavior |
|---|---|
| `force` (default) | Resolve and validate the complete plan; explicitly update all selected authoritative sources; then update their mapped constructed inventories. Wait for each native update to complete. |
| `automatic` | Make no controller API calls and request no explicit updates. Subsequent consuming jobs use AAP's existing update-on-launch and cache settings. |

For both selections in force mode, the phase order is:

1. `inventory_ad / source_ad_ldap`.
2. `inventory_satellite / source_satellite`.
3. Native constructed source for `managed_windows`.
4. Native constructed source for `managed_linux`.

Selection order controls order within each phase. With AD alone, only AD and
managed_windows are explicitly refreshed; Satellite alone selects Satellite
and managed_linux. Existing AAP source dependencies may also cause native updates
according to the platform's configured dependency/cache behavior.

Force means explicit update requests; it does not change source configuration,
clear caches globally, or unconditionally refresh an SCM project. Project revision
refresh follows the existing project/source settings. Automatic does not verify
freshness or enable update-on-launch on an authoritative source.

Before any update, force mode checks exact inventory/organization names,
authoritative source name/type, constructed inventory kind, its input dependency,
and its one native constructed source. A failed authoritative phase prevents the
constructed phase. A module loop may still attempt other entries in the same
phase after one entry fails; the overall job fails and success-linked downstream
nodes do not launch.

The native module handles monitoring. A timeout fails this job but does not cancel
an update already running in AAP. Concurrent update requests can be rejected by
AAP; no custom retry, cancellation, or polling engine is added.

## Inputs

| Variable | Meaning | Default |
|---|---|---|
| `inventory_sync_inventories` | Nonempty list of catalog keys; survey selects authoritative inventory names | Required |
| `inventory_sync_mode` | `force` or `automatic` | `force` |
| `inventory_sync_organization` | Exact organization name for inventory lookup | `RITCSUSA` |
| `inventory_sync_timeout` | Maximum monitoring seconds per update | 600 |
| `inventory_sync_interval` | Native polling interval in seconds | 2 |
| `inventory_sync_catalog` | Inventory/source/constructed bindings | `vars/targets.yml` |

Surrounding whitespace is trimmed and duplicate selections removed. Names retain
their case; unknown or blank names fail validation instead of broadening scope.
The multiselect survey returns a list. Comma-separated strings and numeric IDs
are not accepted as survey selections.

`vars/inventory-sync.example.yml` provides a populated example. The previous
`inventory_sync_sources` list is replaced by this survey contract; update old
job/node extra vars before using this revision. The playbook explicitly rejects
the legacy variable instead of silently ignoring it.

## AAP Job Template and survey

1. Sync the Project for this repository.
2. Create/select the reusable Job Template using `inventory-sync.yml`.
3. Select the existing EE containing `ansible.controller` with AAP 2.7 gateway
   support (4.8.0 or later); this repository does not rebuild the EE.
4. Select an existing template inventory. Execution uses implicit localhost.
   Keep the template/node **Limit blank**; a guest-FQDN limit excludes localhost.
5. Attach the existing Red Hat Ansible Automation Platform API credential.
   Use the gateway URL and OAuth token via standard collection connection options.
6. Configure organization and optional timing/catalog values as job extra vars.
   Remove any old `inventory_sync_sources` value.
7. Enable the survey described by `aap/survey.json`.

The survey has two required questions:

| Prompt | Variable | Type | Choices/default |
|---|---|---|---|
| Authoritative inventories to refresh | `inventory_sync_inventories` | Multiple Choice (multiple select) | `inventory_ad`, `inventory_satellite`; both selected by default |
| Refresh mode | `inventory_sync_mode` | Multiple Choice (single select) | `force`, `automatic`; default `force` |

`aap/survey.json` is a native survey specification for the job template's
`survey_spec` API or equivalent UI setup. Committing it does not install or
enable the survey in live AAP. Survey choices are configured choices, not a live
inventory resource picker. If the catalog changes, update the survey too.

Use an account/token that can read the organization, selected inventories,
their sources and input relationships, and launch the selected source updates.
Source jobs retain their existing source credentials. No Machine credential,
guest SSH/WinRM identity, or secrets belong in this playbook or vars files.
TLS validation is explicitly enabled for native lookups and updates.

`collections/requirements.yml` keeps AAP project-time collection installation
disabled. The EE supplies the collection; the commented declaration is a CLI
placeholder.

## Workflow reuse

Use the same refresh template at these boundaries, normally in force mode:

| Boundary | Placement |
|---|---|
| Before deployment | Refresh, then provision on success |
| After deployment | Run after verified provisioning; start guest jobs on refresh success |
| After teardown | Refresh after cleanup to reconcile the remaining upstream state |

A workflow node can supply the same survey variables for unattended launches.
This repository does not change the Hyper-V lifecycle repository or live workflows.
An Always edge after teardown can still refresh after a partial cleanup failure;
a successful refresh must not erase or disguise the teardown failure.

An inventory refresh does not rewrite the host snapshot of an already-running
playbook job. Launch consuming jobs after force mode succeeds. A successful source
update does not prove that a particular new host passed source filters, joined AD,
or is ready for SSH/WinRM. Deployment/consuming workflows retain those checks.

Host removal depends on upstream removal and existing source Overwrite settings.
This playbook does not independently delete hosts or alter reconciliation rules.
It also does not change classification rules, names, descriptions, filters,
credentials, permissions, schedules, or cache settings. Source inventory imports
naturally update their inventory contents.

Force mode is not supported with Ansible check mode; it fails before API calls.
Automatic mode can run without an API credential because it makes no API calls.

## CLI

With the supported collection and trusted controller CA already available:

```bash
ansible-playbook inventory-sync.yml -e @vars/inventory-sync.example.yml
```

Supply authentication privately through the collection's standard `CONTROLLER_*`
environment/configuration mechanism. The example vars file contains no secrets.

## Validation

Local validation covers YAML/JSON parsing, Ansible syntax, and orchestration
using temporary controller lookup/module test doubles: phase ordering,
selection normalization, single-feed selection, automatic mode, invalid inputs,
dependency mismatch, and authoritative failure gating. Those checks do not
substitute for a live run with the certified collection in the AAP EE.
No live AAP update is performed by repository validation.

## References

- [AAP 2.7 inventory administration, including constructed update-on-launch](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/pdfs/aap-administer-2-7-pdf.pdf)
- [AAP 2.7 gateway API changes and collection requirements](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/upgrade-con_upgrade_api_changes_27)
- [Upstream native inventory_source_update module](https://github.com/ansible/awx/blob/devel/awx_collection/plugins/modules/inventory_source_update.py)
- [Upstream native controller_api lookup](https://github.com/ansible/awx/blob/devel/awx_collection/plugins/lookup/controller_api.py)

AAP uses Red Hat's `ansible.controller`; AWX links document the upstream
implementation, not an additional collection dependency.
