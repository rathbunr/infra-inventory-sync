# infra-inventory-sync

Standalone AAP inventory-source synchronization using the maintained
`ansible.controller.inventory_source_update` module.

Run `inventory-sync.yml` independently or reuse its Job Template before deployment,
after verified deployment, and after teardown. The playbook synchronizes the
explicitly selected sources in order and waits for each update to finish.
A source failure or timeout fails the refresh job and prevents its success-linked
downstream workflow nodes from launching.

## Scope

- Supports two sources or any other explicitly configured nonempty source list.
- Uses existing inventory sources, their source credentials and source settings.
- Reports update job IDs and completion statuses.
- Validates the controller certificate.
- Does not create inventories, sources, hosts, credentials or target systems.
- Does not modify source filters, overwrite settings, schedules or lifecycle code.
- Does not launch deployment, hardening or teardown jobs.

An inventory can have multiple sources. Select the exact source to refresh rather
than assuming the inventory name identifies its source. Names or IDs are accepted.
An optional organization scopes inventory lookup by name. Source lookup is scoped
to its selected inventory.

## Requirements

- An AAP execution environment containing Red Hat's `ansible.controller` collection.
- Controller connection/authentication through the collection's standard
  `CONTROLLER_*` environment or supported controller configuration.
- Permission to update each selected inventory source.
- An existing, functional source configuration and its attached source credential.
- Trusted controller TLS certificates available to the execution environment.

Use the existing Red Hat Ansible Automation Platform credential to supply
controller connection values when available. No Machine credential or target SSH
connection is needed by this localhost playbook. Source jobs use the credentials
already attached to the inventory sources. Keep tokens and passwords out of Git.

`collections/requirements.yml` leaves collection installation disabled for AAP.
The EE supplies the collection. A project sync does not change an EE. The commented
declaration remains available for an operator preparing a CLI environment.

## Inputs

Copy `vars/inventory-sync.example.yml` to an operator-controlled non-secret file,
or place the same mappings in the AAP Job Template's extra variables. Populate
the exact names or IDs of your two inventories and their sources.

| Variable | Meaning | Default |
|---|---|---|
| `inventory_sync_sources` | List of `inventory`, `source`, optional `organization` mappings | Required |
| `inventory_sync_timeout` | Maximum monitoring time in seconds per source | 600 |
| `inventory_sync_interval` | Status polling interval in seconds | 2 |

The example intentionally has blank selections because the actual source names
have not been supplied. The playbook rejects blank selections before any update.
To support a different workflow, change the source list on that Job Template
or workflow node.

Sources run sequentially in list order. Put an upstream source before an inventory
that depends on it, such as a constructed inventory. This playbook does not discover
or reorder dependencies. If one loop item fails, Ansible can still attempt the
remaining selected items, but the overall refresh job remains failed.

A timeout ends monitoring and fails the playbook. It does not cancel the running
AAP inventory update job. Inspect that job before retrying. Concurrent update
requests may be rejected by the native controller API; this playbook reports the
failure rather than adding custom retry or cancellation behavior.

## AAP Job Template

1. Create or select the Project for this repository.
2. Create a Job Template using `inventory-sync.yml`.
3. Select the EE that already supplies `ansible.controller`.
4. Select an existing inventory for the template. Execution uses implicit localhost.
5. Attach the controller/API credential and configure `inventory_sync_sources`.
6. Keep the job's Limit blank. A guest-FQDN limit would exclude localhost.
7. Run the refresh independently to verify both source updates.

The playbook uses the maintained module's own completion monitoring and failure
handling. It does not implement custom HTTP polling or inventory reconciliation.
Inventory updates are operations and normally report changed on each invocation.

## Workflow placement

Reuse the same refresh Job Template at all three boundaries:

| Boundary | Placement and continuation |
|---|---|
| Before deployment | Refresh first, then launch provisioning on success |
| After deployment | Launch after verified provisioning, then launch guest jobs on refresh success |
| After teardown | Launch after cleanup, then continue dependent jobs on refresh success |

For a teardown that can partially fail, an Always edge can still refresh inventory
to reflect the state that remains. A successful refresh does not mean teardown
itself succeeded. Preserve the cleanup job's failure result for review.

Configure sources on the reusable Job Template or enable the appropriate prompt
so a workflow node can supply its own source list. This repository does not modify
the existing Hyper-V workflow or its transaction/rollback boundaries.

### Inventory freshness and teardown behavior

AAP loads a playbook job's inventory at launch. Refreshing inventory does not
rewrite an already-running job's host snapshot. Launch guest hardening and other
inventory-dependent jobs only after this refresh job has completed successfully.

A successful source update imports only what the upstream system exposes through
its configured filters. It does not prove the newly deployed host appears in every
selected inventory, that it joined a domain, or that it is ready for SSH/WinRM.
Those postconditions belong to the deployment and consuming workflows.

For teardown, source overwrite behavior governs reconciliation of removed hosts.
The host must also have disappeared from the upstream system. Review the existing
source's Overwrite/Overwrite Variables settings and ownership behavior; this
playbook does not change them or delete hosts independently.

## CLI execution

From an environment where `ansible.controller` is already installed:

```bash
ansible-playbook inventory-sync.yml -e @/path/to/inventory-sync.yml
```

Provide controller authentication privately using the collection's supported
environment/configuration mechanism. The vars file contains source selectors
and timing settings, not secrets.

## References

- [AAP 2.7 workflow visualizer and native Inventory Sync nodes](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/develop-con_controller_workflow_visualizer)
- [AAP 2.7 inventory source configuration](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/administer-proc_controller_add_source)
- [Upstream inventory_source_update implementation](https://github.com/ansible/awx/blob/devel/awx_collection/plugins/modules/inventory_source_update.py)
- [Upstream completion monitoring](https://github.com/ansible/awx/blob/devel/awx_collection/plugins/module_utils/controller_api.py)

AAP uses the Red Hat `ansible.controller` collection. The upstream AWX references
document the module's supported inventory/source selection and completion behavior;
they are not a dependency declaration for `awx.awx`.
