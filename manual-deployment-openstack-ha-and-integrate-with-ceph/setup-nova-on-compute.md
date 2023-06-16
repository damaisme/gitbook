# Setup Nova on Compute

## Install and Configure nova-compute (Execute on compute nodes)

1. Install packages

```bash
apt install -y nova-compute
```

2. Modify nova-compute configuration

```bash
vi /etc/nova/nova.conf

[DEFAULT]
osapi_compute_listen = 10.10.10.X
metadata_listen = 10.10.10.X
my_ip = 10.10.10.X
state_path = /var/lib/nova
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:rabbit!dama@10.10.10.11:5672,openstack:rabbit!dama@10.10.10.12:5672,openstack:rabbit!dama@10.10.10.13:5672//
log_dir = /var/log/nova
cpu_allocation_ratio = 4.0
ram_allocation_ratio = 2.0

[oslo_messaging_rabbit]
rabbit_hosts=10.10.10.11:5672,10.10.10.12:5672,10.10.10.13:5672
rabbit_retry_interval=1
rabbit_retry_backoff=2
rabbit_max_retries=0
rabbit_durable_queues=true
rabbit_ha_queues=true

[api]
auth_strategy = keystone

[glance]
api_servers = http://10.10.10.100:9292
cafile =
num_retries = 3
debug = False

[cinder]
catalog_info = volumev3:cinderv3:internalURL
os_region_name = gti
auth_url = http://internal.java.dama.id:5000/v3
auth_type = password
project_domain_name = Default
user_domain_id = default
project_name = service
username = cinder
password = cinder!dama
cafile =

[oslo_concurrency]
lock_path = $state_path/tmp

[api_database]
connection = mysql+pymysql://nova:nova!dama@10.10.10.100/nova_api

[database]
connection = mysql+pymysql://nova:nova!dama@10.10.10.100/nova

[keystone_authtoken]
www_authenticate_uri = http://internal.java.dama.id:5000
auth_url = http://internal.java.dama.id:5000/v3
memcached_servers = 10.10.10.11:11211,10.10.10.12:11211,10.10.10.13:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = nova!dama

[placement]
auth_url = http://internal.java.dama.id:5000/v3
os_region_name = java
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = placement
password = placement!dama

[wsgi]
api_paste_config = /etc/nova/api-paste.ini

[neutron]
auth_url = http://internal.java.dama.id:5000/v3
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = gti
project_name = service
username = neutron
password = rahasia
service_metadata_proxy = True
metadata_proxy_shared_secret = skskdjqoqlaoslod

[vnc]
enabled = true
server_listen = 10.10.10.X
server_proxyclient_address = 10.10.10.X
novncproxy_host = 10.10.10.X
novncproxy_port = 6080
novncproxy_base_url = http://public.java.dama.id:6080/vnc_auto.html
#ssl_only=true
#cert=/etc/ssl/certs/cert.dce.bri.co.id.pem

[libvirt]
images_rbd_pool = vms
images_type = rbd
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = f2912ae6-c71c-4fa0-bfb7-4a30798bdf38

[scheduler]
discover_hosts_in_cells_interval = 300
```



## Configure Nova User (Execute on all compute nodes)

1. Modify shell for nova user

```bash
usermod -s /bin/bash nova
```

2. Change nova user password

```bash
echo 'nova:nova!dama' | chpasswd
```

2. Generating private and public key for nova

```bash
su nova
ssh-keygen -t rsa
cat > /var/lib/nova/.ssh/config <<EOF
Host os-compute*
	StrictHostKeyChecking no
EOF
```

3. Distribute public key to each node

```bash
ssh-copy-id os-compute-01
ssh-copy-id os-compute-02
ssh-copy-id os-compute-03
```

4. Verify ssh connection not using password

```bash
ssh os-compute-01 hostname
ssh os-compute-02 hostname
ssh os-compute-03 hostname
```

5. Exit from nova user

```bash
exit
```

6. Modify ssh client configuration for root

```bash
cat >> /root/.ssh/config <<EOF
Host os-comp*
	StrictHostKeyChecking no
EOF
```

7. Restart and enable service nova-compute

```bash
systemctl restart nova-compute
systemctl enable nova-compute
systemctl status nova-compute
```

## Discovering compute node (exec on controller-01)

1. Discovering compute node

```bash
source admin-openrc
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

2. Verify node compute has join to the cluster

```bash
openstack compute service list
```
