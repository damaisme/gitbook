# Setup Neutron on Controller

## Setup Neutron Database (Exec on controller-01)

1. Create mysql database for neutron

```bash
mysql
CREATE DATABASE neutron;
```

2. Grant neutron user for any host access

```bash
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
IDENTIFIED BY 'neutron!dama';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
IDENTIFIED BY 'neutron!dama';
```

```bash
FLUSH PRIVILEGES;
EXIT;
```

## Create Neutron Endpoint (Exec on controller-01)

1. Create neutron user in OpenStack

```bash
source admin-openrc
openstack user create --domain default --password 'neutron!dama' neutron
```

2. Grant admin role for neutron user

```bash
openstack role add --project service --user neutron admin
```

3. Create neutron service in openstack

```bash
openstack service create --name neutron \
--description "OpenStack Networking" network
```

4. Create neutron internal, public, and admin endpoint

```bash
openstack endpoint create --region java \
  network public http://public.java.dama.id:9696
openstack endpoint create --region java \
  network internal http://internal.java.dama.id:9696
openstack endpoint create --region java \
  network admin http://admin.java.dama.id:9696
```



## Setup Haproxy for Neutron service (Exec on all controller nodes)

1. Add haproxy configuration

```
vi /etc/haproxy/haproxy.cfg

# NEUTRON
listen neutron_api_cluster
  bind 10.10.10.100:9696
  bind 202.10.10.100:9696
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
    server os-controller-01 10.10.10.11:9696 maxconn 4000 check inter 2000 rise 2 fall 5
    server os-controller-02 10.10.10.12:9696 maxconn 4000 check inter 2000 rise 2 fall 5 backup
    server os-controller-03 10.10.10.13:9696 maxconn 4000 check inter 2000 rise 2 fall 5 backup


#OVN-REMOTE VIP
frontend ovn_northbound_ovsdb
  bind 10.10.10.100:6641
  default_backend be_ovn_northbound_ovsdb
  mode tcp
backend be_ovn_northbound_ovsdb
  balance leastconn
  mode tcp
  option tcp-check
    server lab-r01-oscontroller-01 10.10.10.11:6641 check inter 2000 rise 2 fall 5
    server lab-r02-oscontroller-02 10.10.10.12:6641 check inter 2000 rise 2 fall 5
    server lab-r03-oscontroller-03 10.10.10.13:6641 check inter 2000 rise 2 fall 5

frontend ovn_southbound_ovsdb
  bind 10.10.10.100:6642
  default_backend be_ovn_southbound_ovsdb
  mode tcp
backend be_ovn_southbound_ovsdb
  balance leastconn
  mode tcp
  option tcp-check
    server lab-r01-oscontroller-01 10.10.10.11:6642 check inter 2000 rise 2 fall 5
    server lab-r02-oscontroller-02 10.10.10.12:6642 check inter 2000 rise 2 fall 5
    server lab-r03-oscontroller-03 10.10.10.13:6642 check inter 2000 rise 2 fall 5
```

## Restart Haproxy with Pacemaker (Exec on controller-01)

```
pcs resource restart lb-haproxy
```



## Clustering OVN (Exec on all controller nodes)

1. Install neutron packages

```bash
apt install -y neutron-server neutron-plugin-ml2 openvswitch-common ovn-common ovn-host ovn-central
```

2. Stop ovn-central di semua controller

```html
systemctl stop ovn-central

systemctl status ovn-central
```

3.  Create OVN central configuration

    controller 1 (master)

    ```
    cat << EOF > /etc/default/ovn-central
    OVN_CTL_OPTS=" \
    --db-nb-addr=10.10.10.11 \
    --db-nb-cluster-local-addr=10.10.10.11 \
    --db-nb-create-insecure-remote=yes \
    --ovn-northd-nb-db=tcp:10.10.10.11:6641,tcp:10.10.10.12:6641,tcp:10.10.10.13:6641 \
    --db-sb-addr=10.10.10.11 \
    --db-sb-cluster-local-addr=10.10.10.11 \
    --db-sb-create-insecure-remote=yes \
    --ovn-northd-sb-db=tcp:10.10.10.11:6642,tcp:10.10.10.12:6642,tcp:10.10.10.13:6642"
    EOF
    ```

    controller 2(slave)

    ```
    cat << EOF > /etc/default/ovn-central
    OVN_CTL_OPTS=" \
    --db-nb-addr=10.10.10.12 \
    --db-nb-cluster-local-addr=10.10.10.12 \
    --db-nb-cluster-remote-addr=10.10.10.11 \
    --db-nb-create-insecure-remote=yes \
    --ovn-northd-nb-db=tcp:10.10.10.11:6641,tcp:10.10.10.12:6641,tcp:10.10.10.13:6641 \
    --db-sb-addr=10.10.10.12 \
    --db-sb-cluster-local-addr=10.10.10.12 \
    --db-sb-cluster-remote-addr=10.10.10.11 \
    --db-sb-create-insecure-remote=yes \
    --ovn-northd-sb-db=tcp:10.10.10.78:6642,tcp:10.10.10.12:6642,tcp:10.10.10.13:6642"
    EOF
    ```

    controller 3 (slave)

    ```
    cat << EOF > /etc/default/ovn-central
    OVN_CTL_OPTS=" \
    --db-nb-addr=10.10.10.13 \
    --db-nb-cluster-local-addr=10.10.10.13 \
    --db-nb-cluster-remote-addr=10.10.10.11 \
    --db-nb-create-insecure-remote=yes \
    --ovn-northd-nb-db=tcp:10.10.10.11:6641,tcp:10.10.10.12:6641,tcp:10.10.10.13:6641 \
    --db-sb-addr=10.10.10.13 \
    --db-sb-cluster-local-addr=10.10.10.13 \
    --db-sb-cluster-remote-addr=10.10.10.11 \
    --db-sb-create-insecure-remote=yes \
    --ovn-northd-sb-db=tcp:10.10.10.11:6642,tcp:10.10.10.12:6642,tcp:10.10.10.13:6642
    EOF
    ```
4. Backup and delete original DB

```html
mv /var/lib/ovn/ovnsb_db.db /var/lib/ovn/ovnsb_db.db.backup
mv /var/lib/ovn/ovnnb_db.db /var/lib/ovn/ovnnb_db.db.backup
```

5. Start ovn-central service alternately starting from master

```html
systemctl start ovn-central
	
systemctl status ovn-central
systemctl status ovn-northd
systemctl status ovn-sb-ovsdb
systemctl status ovn-nb-ovsdb
```



## Set connection inactivity probe (Exec on controller-01)

1. Set connection inactivity probe to 30s.

```html
ovn-nbctl set-connection --inactivity_probe=30000 ptcp:6641:0.0.0.0
ovn-sbctl set-connection --inactivity_probe=30000 ptcp:6642:0.0.0.0
```

2. Cek ovn cluster connection

```html
ovn-nbctl --db=tcp:10.10.10.11:6641,tcp:10.10.10.12:6641,tcp:10.10.10.13:6641 get-connection
ovn-sbctl --db=tcp:10.10.10.11:6642,tcp:10.10.10.12:6642,tcp:10.10.10.13:6642 get-connection
```

## Configure Neutron (Exec on all controller nodes)

1. Modify neutron configuration

```bash
vi /etc/neutron/neutron.conf

[DEFAULT]
bind_host = 10.10.10.X
core_plugin = ml2
service_plugins = ovn-router
auth_strategy = keystone
state_path = /var/lib/neutron
dhcp_agent_notification = True
allow_overlapping_ips = True
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
policy_file = /etc/neutron/policy.json

[oslo_messaging_rabbit]
rabbit_hosts=10.10.10.11:5672,10.10.10.12:5672,10.10.10.13:5672
rabbit_retry_interval=1
rabbit_retry_backoff=2
rabbit_max_retries=0
rabbit_durable_queues=true
rabbit_ha_queues=true

[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf

[keystone_authtoken]
www_authenticate_uri = http://internal.java.dama.id:5000
auth_url = http://internal.java.dama.id:5000/v3
memcached_servers = 10.10.10.11:11211,10.10.10.12:11211,10.10.10.13:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron!dama

[database]
connection = mysql+pymysql://neutron:neutron!dama@10.10.10.100/neutron

[nova]
auth_url = http://internal.java.dama.id:5000/v3
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = java
project_name = service
username = nova
password = nova!dama

[oslo_concurrency]
lock_path = $state_path/tmp
```

2. Modify ml2 configuration

```bash
vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
mechanism_drivers = ovn
type_drivers = geneve,flat,vlan
tenant_network_types = geneve
extension_drivers = port_security
overlay_ip_version = 4

[ml2_type_geneve]
vni_ranges = 1:65536
max_header_size = 38

[ml2_type_flat]
flat_networks = physpro1,physpro2

[securitygroup]
enable_security_group = true

[ovn]
#ovn_nb_connection = tcp:10.10.10.100:6641
#ovn_sb_connection = tcp:10.10.10.100:6642
ovn_nb_connection = tcp:10.10.10.11:6641,tcp:10.10.10.12:6641,tcp:10.10.10.13:6641
ovn_sb_connection = tcp:10.10.10.11:6642,tcp:10.10.10.12:6642,tcp:10.10.10.13:6642
ovn_l3_scheduler = leastloaded
ovn_metadata_enabled = true
enable_distributed_floating_ip = false
```

## Populate and Sync Neutron Database (exec on controller -01)

1. Populate neutron database

```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

2. DB synchronization between Neutron to OVN DB

{% code overflow="wrap" %}
```html
neutron-ovn-db-sync-util --ovn-neutron_sync_mode log --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini

neutron-ovn-db-sync-util --ovn-neutron_sync_mode repair --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini
```
{% endcode %}

## Configure Openvswitch (Exec on all controller node)

1. Configuring tunnel network on openvswitch

```bash
ovs-vsctl set open . external-ids:ovn-remote="tcp:10.10.10.11:6642,tcp:10.10.10.12:6642,tcp:10.10.10.13:6642"
ovs-vsctl set open . external_ids:ovn-nb="tcp:10.10.10.100:6641"
ovs-vsctl set open . external-ids:ovn-encap-type=geneve
ovs-vsctl set open . external-ids:ovn-encap-ip=$(ip -4 addr show ens5 | grep -oP '(?<=inet\\s)\\d+(\\.\\d+){3}' | head -1)
```

2. Configuring provider network on openvswitch

```bash
ovs-vsctl --may-exist add-br br-pro1 -- set bridge br-pro1 protocols=OpenFlow13
ovs-vsctl --may-exist add-port br-pro1 ens8

ovs-vsctl --may-exist add-br br-pro2 -- set bridge br-pro2 protocols=OpenFlow13
ovs-vsctl --may-exist add-port br-pro2 ens9

ovs-vsctl set open . external-ids:ovn-bridge-mappings=physpro1:br-pro1,physpro2:br-pro2

ovs-vsctl set open . external-ids:ovn-cms-options=enable-chassis-as-gw
```

## Restart and enable service (Exec on all controller node)

```bash
systemctl restart neutron-server
systemctl enable neutron-server
systemctl status neutron-server
```

##

## Verify network ovn-controller on OpenStack

```bash
openstack network agent list
```
