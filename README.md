# OKD with ansible

This repository holds ansible files for installing
OKD using Ansible on hardware.

[Installing a user-provisioned cluster on bare metal](https://docs.okd.io/latest/installing/installing_bare_metal/installing-bare-metal.html)

It assumes you have the following hardware:

1. A debian server (e.g. Raspberry PI) for infrastructure (load balancing and serving the ignition files)
2. An intel machine that will first serve as bootstrap (can be the same as the machine that will later serve as worker1).
3. 3 master intel machines.

If you have more hardware, adjust the inventory file accordingly.

NOTE: If you want to run a different version of OKD, extract
the installer under ./openshift-install which will stop retrieving the installer.
The exact command to extract the installer is available per beta version at
[OKD Nightly Releases](https://amd64.origin.releases.ci.openshift.org/).

There are optional components to install which are controlled by
ansible variables. These are defined in the inventory group vars.

| variable                                  | description                                                       |
|-------------------------------------------|-------------------------------------------------------------------|
| argocd                                    | Whether to install the ArgoCD operator.                           |
| sealed_secrets                            | Whether to pre-install a sealed secrets decryption key.           |
| set_etc_hostname_in_ignition_file         | Whether to set the hostname in /etc/hostname.                     |
| use_control_plane_nodes_for_compute       | Whether to allow masters to be used for regular pods.             |
| openshift_agent_installation              | Use agent-based installation instead of bootstrap.                |
| openshift_agent_rendezvous_ip             | Rendezvous IP for agent-based installation.                       |
| create_local_files_platform               | Platform type for install-config (`none` or `baremetal`).         |
| create_local_files_network_type           | Network type (`OVNKubernetes` or `OpenShiftSDN`).                 |
| create_local_files_machine_networks       | List of machine network CIDRs.                                    |
| opt_out_of_remote_health_reporting        | Opt out of remote health reporting.                               |
| okd_disable_default_operator_catalog_sources | Disable default operator catalog sources.                      |

# Load Balancer

The load balancer can be installed on a debian server (e.g. Raspberry PI) and is used for infrastructure (load balancing and serving the ignition files).

Nginx is used for static hosting (ignition files) and HAProxy is used for load balancing.
The HAProxy status interface is found at [infra1.okd4.example.com:1936/stats](infra1.okd4.example.com:1936/stats).

The load balancer installation can be controlled with the following parameters:

| Variable                                | Description                                | Default    |
|-----------------------------------------|--------------------------------------------|------------|
| lbs_loadbalancer_configure              | Whether to configure the HA proxy at all   | true       |
| lbs_loadbalancer_haproxy_stats_auth     | Enable HAProxy status page authentication. | false      |
| lbs_loadbalancer_haproxy_stats_username | HAProxy status page username.              | `admin`    |
| lbs_loadbalancer_haproxy_stats_password | HAProxy status page password.              | `password` |
| configure_ufw                           | Configure UFW port openings.               | `false`    |

## Using OpenShift's internal load balancer

Instead of an external HAProxy, you can let OpenShift
manage load balancing internally using keepalived VIPs
on the master nodes. Set the following:

```yaml
create_local_files_platform: "baremetal"
lbs_loadbalancer_configure: false
```

When `create_local_files_platform` is set to `baremetal`,
the installer configures `apiVIPs` and `ingressVIPs` by
resolving `api.<cluster>.<domain>` and
`*.apps.<cluster>.<domain>` from DNS. OpenShift then
manages these VIPs across master nodes automatically.

When `lbs_loadbalancer_configure` is `false`, the playbook
skips all external LBS provisioning (HAProxy, Nginx,
ignition file hosting, network boot configuration).

This is typically used together with agent-based
installation. See the
[Agent-based installation](#agent-based-installation)
section.

# Disconnected registry

In disconnected environments you normally have a docker registry which will supply all the OKD images.

To use a disconnected registry, set the following parameters:

| variable                                | description                                                                                     |
|-----------------------------------------|-------------------------------------------------------------------------------------------------|
| use_disconnected_registry               | Boolean. Indicating a registry should be used. Example: true                                    |
| disconnected_registry_trust_bundle_file | String. Filename of the root CA for the registry. Example: ./openshift-ca/example.crt           |
| disconnected_registries                 | Array of objects. Mirror registries.                                                            |
| disconnected_registries.source          | String. URL that will be replaced with mirror. Example: quay.io/okd/scos-release                |
| disconnected_registries.mirrors         | Array of strings. URL to mirror registry. Example: registry.okd4.example.com:5011/okd/scos-release |

# Pull-through-cache

In disconnected environments you normally have a docker registry which will supply all the OKD images. The
pull-through-cache will simulate that and should also, if you have a faster cache than your Internet connection,
improve your installation time.

There is a `Makefile` under `hack/docker-proxy` that can create the certificates and docker containers needed for that.
The certificates will be placed under `openshift-ca` and will need to be copied to the docker machine. Note that
the hostname `registry.okd4.example.com` needs to be setup in DNS.

| Usage          | description                                                                                                   |
|----------------|---------------------------------------------------------------------------------------------------------------|
| make ca        | creates both CA and registry certificates                                                                     |
| make quay.io   | creates the docker proxy registry for quay.io, authenticated with the quay.io credentials from `pull-secret` |
| make docker.io | creates the docker proxy registry for docker.io, needs `DOCKER_HUB_USERNAME` and `DOCKER_HUB_ACCESS_TOKEN`   |

# Preparation

1. Pull your [RedHat pull secret](https://console.redhat.com/openshift/install/metal/user-provisioned) and place in file `pull-secret`.
2. [If you want github integration](https://docs.openshift.com/container-platform/latest/authentication/identity_providers/configuring-github-identity-provider.html),
   create a file called "github-config.json" with similar content as this:
   ```json
   {
     "clientID": "< github client ID >",
     "clientSecret": "< github client secret >",
     "organizations": [
       "< your github organization >"
     ]
   }
   ```
3. Set up DNS with the following entries (of course, adjust
   addresses to your infrastructure):

   | hostname                       | Address                        |
   |--------------------------------|--------------------------------| 
   | infra1.okd4.example.com        | 192.168.60.180                 |
   | api-int.okd4.example.com       | CNAME infra1.okd4.example.com  |
   | api.okd4.example.com           | CNAME infra1.okd4.example.com  |
   | apps.okd4.example.com          | CNAME infra1.okd4.example.com  |
   | *.apps.okd4.example.com        | CNAME infra1.okd4.example.com  |
   | master1.okd4.example.com       | 192.168.60.181                 |
   | master2.okd4.example.com	    | 192.168.60.182                 |
   | master3.okd4.example.com	    | 192.168.60.183                 |
   | worker1.okd4.example.com       | 192.168.60.184                 |

   If you're using pihole (as I do), increase rate limiting, create `/etc/dnsmasq.d/99-openshift.conf`
   with the following content and restart dns (`pihole reloaddns`)
   ```
   address=/.apps.okd4.example.com/192.168.60.180
   ```

   If your DNS cannot handle wildcards, add these entries as CNAME, pointing to app.okd4.example.com:

   - alertmanager-main-openshift-monitoring.apps.okd4.example.com 
   - canary-openshift-ingress-canary.apps.okd4.example.com
   - console-openshift-console.apps.okd4.example.com
   - downloads-openshift-console.apps.okd4.example.com
   - grafana-openshift-monitoring.apps.okd4.example.com
   - oauth-openshift.apps.okd4.example.com
   - prometheus-k8s-openshift-monitoring.apps.okd4.example.com
   - thanos-querier-openshift-monitoring.apps.okd4.example.com

4. Create a new image for the rasperry pi with enabled ssh and boot it up.
5. Create a bootable USB from the correct version of CoreOS.
   Use the following command to get the ISO URL:

   ```shell
   ./openshift-install/openshift-install coreos print-stream-json | jq -r '.architectures.x86_64.artifacts.metal.formats.iso.disk.location'
   ```
6. Set up static DHCP entries for the machines, matching IP addresses above.

### Python PIP proxy

If you are using a proxy for PIP, you can configure this by defining the `PIP_INDEXURL` and `PIP_CERT` environment variables in the `.env` or `${CLUSTER_NAME}/.env` file.

### Ansible CA

If you are downloading tools from a self-hosted proxy that is using a custom CA, you need to configure Ansible to use this CA.

You can do this by defining the `SSL_CERT_FILE` environment variable in the `.env` or `${CLUSTER_NAME}/.env` file.

# Run playbook

It's time to run the playbook. There are a number of steps
that will be completed:

1. Cluster configuration will be created and added into
   ignition files.
2. The infrastructure node (RPI) will be setup.

This is how to run the playbook:

```shell
make dependencies
CLUSTER_NAME=example-ocp make cluster
```

Or if you wish to run the playbook directly:

```shell
ansible-playbook -i inventories/example-ocp -v deploy-okd.yml --extra-vars "use_control_plane_nodes_for_compute=true argocd=true"
```

Available inventories: `example-ocp`, `example-ocp-agent`,
`example-scos`. The minimum supported OKD/OCP version is 4.16.

Once the playbook tell you to, boot the masters on CoreOS USB.

To switch keyboard mapping on CoreOS, do the following:

```shell
$ sudo localectl set-keymap se
```

Then start the installation process.

```shell
# Use hostname from DHCP or DNS
$ bash -c "$(curl -fsSL http://infra1.okd4.example.com:8080/install.sh)"
```

```shell
# Manually set hostname
$ bash -c "$(curl -fsSL http://infra1.okd4.example.com:8080/install.sh)" -s master[1-3]
```

Once the installation has finished, remove the bootable media and reboot.

Verify that the master is trying to pull the secondary ignition from `https://api-int.okd4.example.com:22623`.

Once all masters are waiting for the secondary ignition, continue the playbook
which tell you to boot the first worker machine on
CoreOS USB and start the installation for the bootstrap process:

```shell
# Use hostname from DHCP or DNS
$ bash -c "$(curl -fsSL http://infra1.okd4.example.com:8080/install.sh)"
```

```shell
# Manually set hostname
$ bash -c "$(curl -fsSL http://infra1.okd4.example.com:8080/install.sh)" -s bootstrap
```

Once the installation has finished, remove the bootable media and reboot.

After some time, you will be able to login via ssh to the bootstrap machine
and follow the installation:

```shell
$ ssh core@worker1.okd4.example.com
The authenticity of host 'worker1.okd4.example.com (192.168.60.184)' can't be established.
ECDSA key fingerprint is SHA256:Z3edOf5ImnxO/x9tchkto5LoEQIaFm8DT/7zyGj5r6g.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'worker1.okd4.example.com,192.168.60.183' (ECDSA) to the list of known hosts.
This is the bootstrap node; it will be destroyed when the master is fully up.

The primary services are release-image.service followed by bootkube.service. To watch their status, run e.g.

  journalctl -b -f -u release-image.service -u bootkube.service
Fedora CoreOS 34
Tracker: https://github.com/coreos/fedora-coreos-tracker
Discuss: https://discussion.fedoraproject.org/c/server/coreos/

Last login: Sat Sep  4 05:55:40 2021 from 192.168.40.50
[core@nucbootstrap ~]$ journalctl -b -f -u release-image.service -u bootkube.service
```

When the installation has started, continue the playbook.
When the playbook detects that the installation is finished, the playbook will continue with post-installation configuration.

You can now open the cluster console by
opening https://console-openshift-console.apps.okd4.example.com.

Have fun with your cluster!

## Agent-based installation

As an alternative to the bootstrap flow described above,
you can use the agent-based installer. This generates a
single bootable ISO that all nodes boot from. The nodes
discover each other via a rendezvous IP, eliminating the
need for a separate bootstrap machine and external load
balancer.

To use agent-based installation, set the following in
your inventory group vars:

```yaml
openshift_agent_installation: true
openshift_agent_rendezvous_ip: 192.168.60.181

# Use OpenShift internal load balancer instead of
# external HAProxy
create_local_files_platform: "baremetal"
lbs_loadbalancer_configure: false
```

Each host in the inventory must define a `mac_address`:

```ini
[masters]
master1.okd4.example.com mac_address=1c:69:7a:a5:c8:4e
master2.okd4.example.com mac_address=1c:69:7a:a5:c9:5a
master3.okd4.example.com mac_address=1c:69:7a:a3:76:24

[workers]
worker1.okd4.example.com mac_address=f4:4d:30:6d:8d:d6
```

No `[bootstrap]` group is needed.

Run the playbook as usual:

```shell
CLUSTER_NAME=example-ocp-agent make cluster
```

The playbook will:

1. Generate `install-config.yaml` and `agent-config.yaml`.
2. Build a bootable ISO (`agent.x86_64.iso`).
3. Pause and ask you to create a bootable USB from the
   ISO and boot all nodes from it.
4. Wait for bootstrap and installation to complete using
   `openshift-install agent wait-for`.

Workers register automatically — no manual `oc` enrollment
or certificate approval is needed.

See `inventories/example-ocp-agent` for a complete example.

## Wiping cluster nodes

To reinstall a cluster on the same machines, the old
installation must be removed first. Otherwise the machines
keep booting from disk even when a USB stick is selected.

```shell
CLUSTER_NAME=example-scos-agent make wipe-nodes
```

This destroys the cluster. The playbook lists the masters
and workers it will wipe and asks you to type `yes`. It
then erases the partition table of each node's boot disk
and powers the node off immediately, without a normal
shutdown. Start the machines again yourself; with no
bootable disk left, they boot from USB or network.

To wipe only some nodes, use a limit:

```shell
CLUSTER_NAME=example-scos-agent \
  ANSIBLE_EXTRA_ARGS='-l master2.okd4.example.com' make wipe-nodes
```

Nodes are only wiped when running an installed CoreOS
system. A node already booted from a live ISO is refused,
and if any node fails before the wipe, no node is wiped.
Only the partition table is erased, so the data is still
on the disk; this is fine for a reinstall but not for
decommissioning.

## Adding the bootstrap or any other node to the cluster

Once the cluster has been correctly installed, shutdown the
bootstrap node, remove the partitions and reinstall using
this command:

```shell
# Use the generic ignition file for the role, "master" or "worker"
$ bash -c "$(curl -fsSL http://infra1.okd4.example.com:8080/install.sh)" -s worker
```

Since this has not been prepared in the cluster earlier,
you need to approve the certificate requests.

This is easiest done with the `issueCertificates.sh` script. Note that there is normally three certificate per node.

```shell
$ ./issueCertificates.sh
./issueCertificates.sh                                                                                           
No pending certificate requests
...
No pending certificate requests
No pending certificate requests
Approving pending certificate requests...
certificatesigningrequest.certificates.k8s.io/csr-qmkv6 approved
No pending certificate requests
Approving pending certificate requests...
certificatesigningrequest.certificates.k8s.io/csr-wfnjq approved
certificatesigningrequest.certificates.k8s.io/csr-qftjq approved
No pending certificate requests
```

## Network booting nodes

The OKD nodes can be booted over the network with iPXE.

| Variable                           | Description                                | Default |
|------------------------------------|--------------------------------------------|---------|
| network_boot_enabled               | Enable configuration for network booting.  | false   |
| network_boot_coreos_proxy_scheme   | Override coreos proxy scheme (http/https). |         |
| network_boot_coreos_proxy_hostname | Override coreos proxy hostname.            |         |
| network_boot_coreos_proxy_port     | Override coreos proxy port.                |         |
| network_boot_coreos_proxy_path     | Override coreos proxy path.                |         |

### Boot order

Configure the following boot order for the machines:

1. Hard Disk (Boot from disk after install)
2. Optical (e.g. boot from iPXE iso)
3. USB (e.g. boot from iPXE USB stick)

### Required DHCP configuration

Required DHCP information:

| DHCP field | Description                  | Example                                        |
|------------|------------------------------|------------------------------------------------|
| Hostname   | Provide hostname using DHCP. | `bootstrap`                                    |
| Domain     | Provide domain using DHCP.   | `okd4.example.com`                             |
| Filename   | Default BIOS file name.      | `lbs.okd4.example.com:8081/ipxe/autoexec.ipxe` |

### How it works

#### PXE

PXE network booting is not supported at the moment.

#### iPXE

iPXE can be booted from an iso, USB drive, etc.

More information: https://ipxe.org/

1. iPXE will start and get a DHCP assignment.
   If your DHCP provider does not set the hostname and domain, you can manually set them with
   ```
   $ set hostname master1
   $ set domain okd4.example.com
   ``` 
2. iPXE will first load `http://infra1.okd4.example.com:8081/ipxe/autoexec.ipxe`
   This can be done manually with
   ```
   $ chain http://infra1.okd4.example.com:8081/ipxe/autoexec.ipxe
   ```
3. Which will load `http://infra1.okd4.example.com:8081/ipxe/$hostname.$domain.ipxe`
   - Hostname and domain values are taken from DHCP.
4. Which will load CoreOS with the nodes ignition file. e.g. `http://lbs.okd4.example.com:8080/ignition/bootstrap.okd4.example.com.ign`

## Local testing using molecule

Test the ansible scripts locally using molecule.  
Molecule uses vagrant and virtualbox to create virtual machines to perform the installation on.

### Dependencies

* Virtualbox
* Vagrant
* Molecule

#### VirtualBox

https://www.virtualbox.org/wiki/Downloads

###### macOS

```shell
brew install --cask virtualbox
```

#### Vagrant

https://developer.hashicorp.com/vagrant/downloads

###### macOS

```shell
brew install hashicorp/tap/hashicorp-vagrant
```

#### Molecule (and python dependencies)

https://molecule.readthedocs.io/en/latest/

```shell
make local-dependencies
```

### Usage

#### Test

To perform a full test, run:

```shell
make molecule-test
```

### Manual testing

These are the commands used for testing manually:

```shell
molecule create
molecule converge
molecule verify
molecule destroy
```

To run a specific scenario:

```shell
molecule create -s <scenario name>
molecule converge -s <scenario name>
molecule verify -s <scenario name>
molecule destroy -s <scenario name>
```
