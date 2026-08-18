# Trino Virtual Warehouse on CDP (cloudera.cloud)

Provision a **Trino** Virtual Warehouse on Cloudera Data Warehouse (CDW) with one idempotent
Ansible playbook. Trino VW support landed in `cloudera.cloud`'s `dw_virtual_warehouse` at commit
[`5ad1809`](https://github.com/cloudera-labs/cloudera.cloud/commit/5ad1809) (#307) — `type: trino`
plus a Trino-only `connectors` full-sync list.

`provision-trino-vw.yml` chains the whole stack against an existing CDP environment:

1. Activate a CDW cluster on the environment (`dw_cluster`)
2. Create a Database Catalog inside it (`dw_database_catalog`)
3. Create the Trino Virtual Warehouse (`dw_virtual_warehouse`, `type: trino`)

Auth comes from the default profile in `~/.cdp/credentials` — no secrets in the playbook. Every
module reconciles to `state: present`, so the playbook is safe to re-run.

## The environment has to be CDW-capable

CDW/EKS activation needs **private worker subnets** and CDW-ready IAM/subnet tagging. A
public-subnet-only environment cannot activate CDW — the create is accepted, then EKS bring-up
flips to `Error` at ~6 minutes with no reason surfaced by the API. Build the target environment
with private worker subnets (on the [cdp-tf-quickstarts](https://github.com/cloudera-labs/cdp-tf-quickstarts)
this is the `semi-private` deployment template) before running this playbook.

## Environment setup

```bash
python3 -m venv ~/.venvs/clouderacloud
source ~/.venvs/clouderacloud/bin/activate
pip install ansible

# Pin the collection to the commit that introduced Trino VW support
ansible-galaxy collection install \
  git+https://github.com/cloudera-labs/cloudera.cloud.git,5ad1809 --force

# dw_cluster / dw_database_catalog still use the legacy cdpy SDK (only dw_virtual_warehouse
# moved to the new cdp_client in #307). Install the cloudera-labs cdpy from git — NOT the
# unrelated PyPI package that is also named "cdpy".
pip install git+https://github.com/cloudera-labs/cdpy.git
```

## Configure and run

Set `env_crn`, the Database Catalog / VW names, and the subnet IDs at the top of
`provision-trino-vw.yml`. Pull the private subnet IDs from the environment:

```bash
cdp environments describe-environment \
  --environment-name <your-env> | jq '.environment.network'
```

Then run it:

```bash
source ~/.venvs/clouderacloud/bin/activate
ansible-playbook provision-trino-vw.yml -v
```

The playbook waits for each step (up to 3600s): CDW cluster `AVAILABLE` → Database Catalog
`AVAILABLE` → Trino VW `AVAILABLE`. It prints the Trino coordinator / JDBC / Hue endpoints on the
VW at the end.

## CDW activation config that works

On a private-subnet (`semi-private`) environment, CDW activates with a **private** load balancer on
the **private** subnets:

| `dw_cluster` var | value | why |
|---|---|---|
| `private_load_balancer` | `true` | A public LB on a private-subnet env is rejected at intake. |
| `aws_lb_subnets` | the private subnets | Must be explicit — required parameter. |
| `aws_worker_subnets` | the same private subnets | Workers run on private subnets. |
| `public_worker_node` | `false` | Workers on private subnets. |
| `overlay` | `true` | Private Pod IP range — conserves VPC IPs in a shared tenant. |

Because the LB is private, the Trino coordinator endpoint is reachable **inside the CDP/VPC network
path**, not from the public internet. Public access needs a VPN or a bastion into the VPC.

## What NOT to do

- **Don't activate CDW with a public LB (`private_load_balancer: false`) on a private-subnet
  environment.** CDW intake accepts the request, then flips to `Error` at ~6 minutes with
  `statusReason: null` — no diagnostic via CLI, UI, or audit log. The LB type is the trap.
- **Don't omit the subnets.** `awsOptions: null` does not work via CLI/Ansible even though a
  UI-created cluster shows it. Omitting subnets fails immediately with
  `missing AWS activation parameters`. Always pass `aws_lb_subnets` and `aws_worker_subnets`.
- **Don't run `cdp environments initialize-aws-compute-cluster`.** It is not part of the CDW path.
  It creates a "default" externalized compute cluster that cannot be deleted on its own
  (`default Compute Cluster cannot be deleted by end user`), wedging the environment in
  `COMPUTE_CLUSTER_CREATION_IN_PROGRESS`. Classic CDW (`dw_cluster`) provisions its own EKS.
- **Don't `pip install cdpy` from PyPI.** That is a different, unrelated package. Install the
  cloudera-labs fork from git.
