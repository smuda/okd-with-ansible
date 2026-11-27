# A hosted cluster on OpenShift Virtualization

This describes how to run a second OpenShift cluster, `hcp`, as
virtual machines on the existing management cluster, using Hosted
Control Planes (HCP) with the KubeVirt provider. The control plane
runs as pods on the management cluster and a NodePool turns into
virtual machines. The playbook `deploy-hosted-cluster.yml` does the
whole job.

Status: removed and installed again as `hcp` on 2026-11-07, with the
API reached by DNS name. The removal took about 3 minutes and the
install about 11 (636 seconds), with the operators already in place:
two Ready nodes, version 4.22.17, 21 cluster operators Available and
none Degraded. A second run took 19 seconds and reported changed=0,
failed=0.

The examples use `inventories/example-ocp-hcp`. The management
cluster is okd4.example.com (OpenShift 4.22.17). All commands are run
from the repository root.

## Prerequisites

The README's [Preparation](../README.md#preparation) applies, and
`make dependencies` from [Run playbook](../README.md#run-playbook). An
older kubernetes.core in `~/.ansible/collections` still wins over the
pinned one; replace it with `ansible-galaxy collection install
'kubernetes.core:6.6.0' --force`. In addition:

- A running OpenShift (not OKD) management cluster, since multicluster
  engine and OpenShift Virtualization come from `redhat-operators`.
- `openshift-files/okd4.example.com/auth/kubeconfig` for it.
- `~/.ssh/id_ansible.pub` (or set `hosted_cluster_ssh_public_key_file`).
- `helm` and `dig` on the machine running the playbook.
- ssh access to the DNS server (pihole in the examples).
- A virtualenv with the Python `kubernetes` library, since Ansible's
  own Python lacks it. Python 3.14 works. On Python 3.15, PyYAML had no
  macOS arm64 wheel when this was written:

  ```shell
  /opt/homebrew/opt/python@3.14/bin/python3.14 -m venv \
    openshift-client/venv
  openshift-client/venv/bin/pip install 'kubernetes==37.0.1'
  ```

## 1. Prerequisites and capacity

Only worker2 (NVMe) hosts anything. The control plane is a
SingleReplica pod set, and the NodePool is two virtual machines.

| Item              | Value                          |
|-------------------|--------------------------------|
| Node pool VMs     | 2, each 2 cores, 8Gi, 64 GB    |
| Placement         | worker2 only                   |
| Longhorn replicas | 1, `dataLocality: best-effort` |
| VM networking     | pod network                    |

The OpenShift Virtualization documentation expects at least three
workers. With one usable worker this is a home lab setup. If worker2
does not have the CPU and memory for the pods and VMs, the fallback
is to make the masters schedulable:

```shell
oc patch schedulers.config.openshift.io cluster --type merge \
  -p '{"spec":{"mastersSchedulable":true}}'
```

Worker2 was enough without that fallback, which has not been run. With
the hosted cluster up it used about 19% CPU and 71% memory. Worker1
has too little free CPU for Longhorn's instance manager, so that pod
keeps failing with `OutOfcpu` there. This does not block anything,
because no volume uses worker1.

## 2. Pick IPs and set the inventory values

MetalLB gets a pool of exactly two free addresses on the node
network. The API takes the lowest one automatically and the apps
service is pinned to the other. `hosted_cluster_api_ip` states which
one the API gets, and the playbook checks that the API service
receives it.

| Value                       | Setting        |
|-----------------------------|----------------|
| `hosted_cluster_pool_start` | 192.168.60.190 |
| `hosted_cluster_pool_end`   | 192.168.60.191 |
| `hosted_cluster_api_ip`     | 192.168.60.190 |
| `hosted_cluster_apps_ip`    | 192.168.60.191 |

Set these, and `hosted_cluster_name`, `hosted_cluster_base_domain`,
`hosted_cluster_node` and `hosted_cluster_release_version`, in
`inventories/example-ocp-hcp/group_vars/all.yml`. `cluster_name`
and `base_domain` there name the management cluster, not the hosted
one, because they select the kubeconfig that the playbook uses.

## 3. Add the DNS records

The playbook fails early if `*.apps.hcp.example.com` does not resolve
to the apps address, or if `api.hcp.example.com` does not resolve to
the API address. Authorize your SSH key on the pihole once, typing
the password at the prompt rather than storing it:

```shell
ssh-copy-id -i ~/.ssh/id_ansible.pub pihole
```

Add both records as described under
[Preparation](../README.md#preparation) in the README:

```shell
ssh pihole
sudo pihole-FTL --config misc.dnsmasq_lines
sudo cp -p /etc/pihole/pihole.toml /etc/pihole/pihole.toml.bak
sudo pihole-FTL --config misc.dnsmasq_lines \
  '["<existing entries>",
    "address=/.apps.hcp.example.com/192.168.60.191",
    "address=/api.hcp.example.com/192.168.60.190"]'
exit
dig +short foo.apps.hcp.example.com
dig +short api.hcp.example.com
```

The first check must print 192.168.60.191 and the second
192.168.60.190.

Setting `misc.dnsmasq_lines` makes FTL restart itself. Once, on
FTL 6.3.3, that restart hung and the pihole stopped answering DNS
until `sudo systemctl restart pihole-FTL` was run, so check that it
answers again after the change.

## 4. Deploy

```shell
PY=$PWD/openshift-client/venv/bin/python
ANSIBLE_EXTRA_ARGS="-e ansible_python_interpreter=$PY" \
  make -e CLUSTER_NAME=example-ocp-hcp hosted-cluster
```

The steps, in order, each safe to run again:

1. Longhorn. A MachineConfig enables `iscsid` and loads `iscsi_tcp`
   on the workers, so the workers reboot once, one at a time. Then
   worker2 is labelled and tagged, Longhorn is installed with Helm
   and the StorageClass `longhorn-local` is created.
2. OpenShift Virtualization and the HyperConverged resource.
3. MetalLB with the address pool and an L2 advertisement.
4. multicluster engine, the HyperShift operator, wildcard routes in
   the default IngressController, and the `hcp` executable in
   `openshift-client/`. It is fetched again when the multicluster
   engine version or the download URL changes (see `.hcp.version`).
5. The DNS checks, then `hcp create cluster kubevirt` (only if the
   HostedCluster does not exist). The manifests are rendered to
   stdout, never to a file, because they hold the pull secret, the
   SSH key and the etcd encryption key. The APIServer service is
   given the LoadBalancer hostname `api.hcp.example.com` before they
   are applied, because the service settings cannot be changed
   afterwards. Killing Ansible mid-run can leave module payloads
   under `~/.ansible/tmp`, which are owner-only. Then the playbook waits
   for the control plane, writes the guest kubeconfig, waits for the
   VMs, creates the apps LoadBalancer service and waits for the
   install to finish. It then checks that the kubeconfig points at
   the API name and that the API service has the API address.

The pull secret is written to
`openshift-files/hcp.example.com/pull-secret` by the same handling as
the other clusters. With
`opt_out_of_remote_health_reporting: true` the telemetry
credentials are removed from it, and an existing HostedCluster gets
the pull secret Secret updated. The update channel is
`hosted_cluster_channel`, `stable-4.22` for 4.22.17.

After the install, `configure-cluster.yml` runs against the hosted
cluster, including ArgoCD with `argocd_value_file: values-hcp.yaml`.
That file must exist in the ArgoCD repository, otherwise the ArgoCD
application does not sync.

With the operators already in place, the whole playbook took about 11
minutes, including creating the hosted cluster. A first run with the
operators missing has not been timed as one run, so allow at least an
hour for it. A later run took 21 seconds and changed nothing.

## 5. Verify

```shell
export KUBECONFIG=openshift-files/hcp.example.com/auth/kubeconfig
oc get nodes
oc get clusterversion
oc get co
```

Expect two Ready nodes, version 4.22.17 with no update in progress, and
every operator Available. The API is at https://api.hcp.example.com:6443
and the console at
https://console-openshift-console.apps.hcp.example.com. The console
certificate is not signed by a CA that macOS trusts, so `curl` needs the
CA or `-k` there. The kubeconfig carries the CA for the API.

The console login is `kubeadmin`. Its password is in a Secret on the
management cluster:

```shell
KUBECONFIG=openshift-files/okd4.example.com/auth/kubeconfig \
  oc extract secret/hcp-kubeadmin-password -n clusters \
  --keys=password --to=-
```

## 6. Limitations

- Everything runs on worker2, so it is a single point of failure.
- The 8Gi nodes leave little room for workloads. Memory requests were
  89% and 57% right after the replace, while actual use was 42% and
  19%. At 6Gi they were 96% and 99%, and two DaemonSet pods,
  `iptables-alerter` and `insights-runtime-extractor`, stayed Pending
  with `Insufficient memory` on one node. A workload may need larger
  nodes, set with `hosted_cluster_memory`.
- There is no live migration. Upgrading or draining worker2 stops
  the virtual machines.
- Longhorn as storage for the hosted cluster is not stated as
  supported by Red Hat.
- The Longhorn chart is pinned by version only, with no digest or
  signature check. The chart also ships RBAC equivalent to
  cluster-admin.
- Longhorn's iSCSI port (3260) on the instance managers is open to
  every pod on the management cluster by the chart's own policy. An
  added NetworkPolicy cannot close it, because policies are additive.
- The Longhorn UI has no Route and only accepts traffic from its own
  namespace. Reach it when needed with
  `oc -n longhorn-system port-forward svc/longhorn-frontend 8080:80`.

## 7. Removal

This has been run twice, first under an earlier cluster name and then
as `hcp` on 2026-11-07. Both times it took about 3 minutes (183
seconds as `hcp`) and exited cleanly.

```shell
KUBECONFIG=openshift-files/okd4.example.com/auth/kubeconfig \
  openshift-client/hcp destroy cluster kubevirt --name hcp
```

This removed the HostedCluster, the NodePool, the virtual machines,
the namespace `clusters-<name>`, the three PVCs with their PVs and
Longhorn volumes, and the namespaces `<name>` and `klusterlet-<name>`.
Afterwards `local-cluster` was the only ManagedCluster.

It left Longhorn, OpenShift Virtualization, MetalLB, multicluster
engine, the node labels, the MachineConfig, the namespace `clusters`,
the DNS records and `openshift-files/<name>.example.com`, whose
kubeconfig no longer works and can be deleted. The MetalLB pool stays
scoped to `clusters-<name>`, so a reinstall under the same name reuses
it, and a new name needs a playbook run. The first time, five
CertificateSigningRequests with the old name in theirs remained; as
`hcp`, none did.

The default IngressController keeps `wildcardPolicy:
WildcardsAllowed`. To revert it:

```shell
oc -n openshift-ingress-operator patch ingresscontroller/default \
  --type merge \
  -p '{"spec":{"routeAdmission":{"wildcardPolicy":"WildcardsDisallowed"}}}'
```
