# Cluster shutdown and startup

Use this runbook when you need to power down the whole cluster (planned
maintenance, moving hardware, an extended outage) and bring it back up
cleanly afterwards.

The cluster ships with Rook-Ceph for storage, spread across all nodes, so a
naive shutdown can leave Ceph mid-rebalance when nodes come back. The steps
below quiesce Ceph first so the storage layer comes back exactly as it left.

## Shut down

### 1. Pause GitOps reconciliation

If ArgoCD auto-sync is enabled for your apps, disable it (or work quickly
through the remaining steps) so it doesn't try to "heal" anything you cordon
or scale down.

### 2. Set Ceph maintenance flags

Exec into the toolbox pod and set flags that stop Ceph from reacting to
nodes going down:

```sh
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd set noout
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd set nobackfill
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd set norecover
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd set norebalance
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd set nodown
```

Confirm the flags are active and the cluster has no in-flight recovery:

```sh
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status
```

You should see `nodown,noout,nobackfill,norebalance,norecover flag(s) set`
in the health section and all PGs `active+clean`.

!!! note

    A `HEALTH_WARN` for something unrelated (e.g. clock skew) doesn't block
    shutdown, but is worth fixing separately — clock skew across mons can
    cause quorum issues.

### 3. Cordon the nodes

```sh
kubectl cordon node01 node02 node03
```

This is a full shutdown, not a drain: pods aren't going anywhere else to
run, so there's no need to evict them — just stop new scheduling.

### 4. Shut down workers first, control plane last

Keeping the control-plane/mon-bearing node up longest keeps k3s and Ceph
mons coordinated for as long as possible.

```sh
ssh root@<node02-ip> shutdown -h now
ssh root@<node03-ip> shutdown -h now
ssh root@<node01-ip> shutdown -h now
```

Node IPs and SSH details are in `metal/inventories/prod.yml` and
`metal/group_vars/all.yml`. You can also drive this through
[`ansible-console`](run-commands-on-multiple-nodes.md) instead of raw SSH.

!!! warning

    If SSH warns that a host key has changed, stop and confirm you actually
    reimaged or rotated keys on that node before accepting the new key. A
    host key mismatch you can't explain may mean something other than the
    node you expect is answering on that IP.

## Start up

### 1. Power the nodes back on

If the nodes support Wake-on-LAN, use the `wake` Ansible role
(`metal/roles/wake`) to send magic packets, starting with the control-plane
node (`node01`) so it's ready to accept the workers rejoining.

Otherwise, power them on physically in the same order: control plane first,
then workers.

### 2. Wait for the cluster to be healthy

```sh
kubectl get nodes
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status
```

Wait for all nodes to show `Ready` and Ceph to report `HEALTH_OK` (or
`HEALTH_WARN` only for pre-existing, unrelated issues), with all mons and
OSDs up.

### 3. Uncordon the nodes

```sh
kubectl uncordon node01 node02 node03
```

### 4. Unset the Ceph maintenance flags

```sh
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd unset noout
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd unset nobackfill
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd unset norecover
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd unset norebalance
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd unset nodown
```

### 5. Resume GitOps reconciliation

Re-enable ArgoCD auto-sync if you disabled it in step 1 of the shutdown
procedure.
