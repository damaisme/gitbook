# Setup Neutron on Compute

## Install and Configure Neutron (Exec on all compute nodes)

1. Install neutron packages

```html
apt install -y openvswitch-common ovn-common ovn-host ovn-central neutron-ovn-metadata-agent
```

2. Modify neutron configuration

```html
vi /etc/neutron/neutron_ovn_metadata_agent.ini

[DEFAULT]
nova_metadata_host = 10.10.10.100
metadata_proxy_shared_secret = skskdjqoqlaoslod

[ovs]
ovsdb_connection = unix:/var/run/openvswitch/db.sock

[ovn]
ovn_sb_connection = tcp:10.10.10.11:6642,tcp:10.10.10.12:6642,tcp:10.10.10.13:6642
```

3. Configuring tunnel network on openvswitch

```html
ovs-vsctl set open . external-ids:ovn-remote="tcp:10.10.10.11:6642,tcp:10.10.10.12:6642,tcp:10.10.10.13:6642"
ovs-vsctl set open . external-ids:ovn-encap-type=geneve
ovs-vsctl set open . external-ids:ovn-encap-ip=$(ip -4 addr show ens5 | grep -oP '(?<=inet\\s)\\d+(\\.\\d+){3}' | head -1)
```

4. Configuring provider network on openvswitch

```html
ovs-vsctl --may-exist add-br br-pro1 -- set bridge br-pro1 protocols=OpenFlow13
ovs-vsctl --may-exist add-port br-pro1 ens8

ovs-vsctl --may-exist add-br br-pro2 -- set bridge br-pro2 protocols=OpenFlow13
ovs-vsctl --may-exist add-port br-pro2 ens9

ovs-vsctl set open . external-ids:ovn-bridge-mappings=physpro1:br-pro1,physpro2:br-pro2

ovs-vsctl set open . external-ids:ovn-cms-options=enable-chassis-as-gw
```

5. Listen ovsdb-server on tcp socket

```html
ovs-vsctl set-manager ptcp:6640:0.0.0.0
```

6. Restart and enable service neutron-ovn-metadata-agent

```html
systemctl restart neutron-ovn-metadata-agent
systemctl enable neutron-ovn-metadata-agent
systemctl status neutron-ovn-metadata-agent
```

## Verify network agent on OpenStack (Exec on controller)

```html
source admin-openrc
openstack network agent list
```
