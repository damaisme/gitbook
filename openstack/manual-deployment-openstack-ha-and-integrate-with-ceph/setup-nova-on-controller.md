# Setup Nova on Controller

## Setup Nova Database (Exec on controller-01)

1. Create mysql database for keystone

```bash
mysql
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
```

2. Grant nova user for any host access

```bash
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \\
IDENTIFIED BY 'nova!dama';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \\
IDENTIFIED BY 'nova!dama';
```

```bash
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \\
IDENTIFIED BY 'nova!dama';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \\
IDENTIFIED BY 'nova!dama';
```

```bash
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \\
IDENTIFIED BY 'nova!dama';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \\
IDENTIFIED BY 'nova!dama';
```

```bash
FLUSH PRIVILEGES;
EXIT;
```

## Setup Nova Endpoint (Exec on controller-01)

1. Create nova user in OpenStack

```bash
source admin-openrc
openstack user create --domain default --password 'nova!dama' nova
```

2. Grant admin role for nova user

```bash
openstack role add --project service --user nova admin
```

3. Create nova service in openstack

```bash
openstack service create --name nova \\
--description "OpenStack Compute" compute
```

4. Create nova internal, public, and admin endpoint

```bash
openstack endpoint create --region java \
  compute public http://public.java.dama.id:8774/v2.1
openstack endpoint create --region java \
  compute internal http://internal.java.dama.id:8774/v2.1
openstack endpoint create --region java \
  compute admin http://admin.java.dama.id:8774/v2.1
```



## Install and Configure Nova (Exec on All Controller Nodes)

1. Install nova packages

```bash
apt install -y nova-api nova-conductor nova-novncproxy nova-scheduler
```

2. Modify nova configuration

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
os_region_name = java
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
region_name = java
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

3. Add haproxy configuration

```
# NOVA COMPUTE
 listen nova_compute_api_cluster
  bind 10.10.10.100:8774
  bind 202.10.10.100:8774
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
    server os-controller-01 10.10.10.11:8774 check inter 2000 rise 2 fall 5
    server os-controller-02 10.10.10.12:8774 check inter 2000 rise 2 fall 5
    server os-controller-03 10.10.10.13:8774 check inter 2000 rise 2 fall 5

# NOVA METADATA
 listen nova_metadata_api_cluster
  bind 10.10.10.100:8775
  bind 202.10.10.100:8775
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
    server os-controller-01 10.10.10.11:8775 check inter 2000 rise 2 fall 5
    server os-controller-02 10.10.10.12:8775 check inter 2000 rise 2 fall 5
    server os-controller-03 10.10.10.13:8775 check inter 2000 rise 2 fall 5

# NOVA VNC
 listen nova_vncproxy_cluster
  bind 202.10.10.100:6080
  #ssl crt /etc/ssl/certs/cert.dce.bri.co.id.pem
  balance  source
  option  tcpka
  option  tcplog
    server os-controller-01 10.10.10.11:6080 check inter 2000 rise 2 fall 5
    server os-controller-02 10.10.10.12:6080 check inter 2000 rise 2 fall 5
    server os-controller-03 10.10.10.13:6080 check inter 2000 rise 2 fall 5

```



## Populate nova database (Exec on controller-01)

```bash
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

## Restart Haproxy (Exec on controller-01)

```
pcs resource restart lb-haproxy
```

## Restart and enable services nova (Execute on all controller-01)

1. Restart and enable service

```
systemctl restart nova-api nova-scheduler nova-conductor nova-novncproxy
systemctl enable nova-api nova-scheduler nova-conductor nova-novncproxy
systemctl status nova-api nova-scheduler nova-conductor nova-novncproxy
```

2. Verify nova-scheduler and nova-conductor on OpenStack

```
openstack compute service list
```

