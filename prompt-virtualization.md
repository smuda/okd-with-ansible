# Goal

I want to have a Openshift cluster running with IPI on top of 
openshift virtualization in the existing cluster at okd4.example.com.

As this will be a long session, use sub-agents as much as possible to
not exhaust the context window.

For the new cluster, use base name ipi.example.com and
store its inventory under inventories/example-ocp-ipi-agent

Use agent based installation (like inventories/example-ocp-agent).

# Documentation

I want you to document how I am supposed to re-create all the steps later
in a new cluster under docs/openshift-ipi-on-virtualization.md.

Any scripts or files needed are stored in the repository the same way before.

# DNS Server

The DNS server in this enviroment is a pihole. You can run SSH to it with
`ssh pihole`
and the password is available in `.env` as PIHOLE_PASSWORD.

# IP Addresses

Prefer static IPs. If DHCP must be used, you need my help to create the
DHCP static entries.

Use the 192.168.60.0/24 network, but make sure there is no conflicts.

# Steps 

Verify before starting, but I believe the correct way to accomplish this is to :

1. Install Longhorn for storage
2. Configure longhorn for local storage
3. install the Openshift virtualization operator
4. Configure the openshift virtualization operator?
5. update the DNS for the new cluster at the pihole according to the README.md
6. Install Openshift with IPI on top of virtualization and Longhorn.
