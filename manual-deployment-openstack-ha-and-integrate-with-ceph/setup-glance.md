# Setup Glance

## Setup Database (Exec on controller-01)

1. Create mysql database for glance

```bash
mysql
CREATE DATABASE glance;
```

2. Grant glance user for any host access

```
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glance!dama';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance!dama';
FLUSH PRIVILEGES;
EXIT;
```

## Setup Openstack Endpoint (Exec on controller-01)

1. Create glance user in OpenStack

```bash
source ~/admin-openrc
openstack user create --domain default --password 'glance!dama' glance
```

2. Grant admin role for glance user

```bash
openstack role add --project service --user glance admin
```

3. Create glance service in openstack

```bash
openstack service create --name glance \
  --description "OpenStack Image" image
```

4. Create glance public, internal and admin endpoint

```bash
openstack endpoint create --region java \
  image public http://public.java.dama.id:9292
openstack endpoint create --region java \
  image internal http://internal.java.dama.id:9292
openstack endpoint create --region java \
  image admin http://admin.java.dama.id:9292
```



## Install and Configure Glance (Exec on all Controller Nodes)

1. Install glance packages

```
apt install -y glance
```

2. Modify glance configuration

{% code overflow="wrap" %}
```
vi /etc/glance/glance-api.conf

[DEFAULT]
debug = False
bind_host = 10.10.10.1X
workers = 5
enabled_backends = file:file, http:http, rbd:rbd, cinder:cinder
transport_url = rabbit://openstack:rabbit!dama@10.10.10.11:5672,openstack:rabbit!dama@10.10.10.12:5672,openstack:rabbit!dama@10.10.10.13:5672//
[cinder]
[cors]

[database]
connection = mysql+pymysql://glance:glance!dama@10.10.10.100/glance
max_retries = -1

[glance_store]
default_backend = rbd
stores = rbd
default_store = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8

[rbd]
rbd_store_user = glance
rbd_store_pool = images
rbd_store_chunk_size = 8

[file]
filesystem_store_datadir = /var/lib/glance/images/

[glance.store.rbd.store]
rbd_store_user = glance
rbd_store_pool = images
rbd_store_chunk_size = 8

[image_format]
disk_formats = ami,ari,aki,vhd,vhdx,vmdk,raw,qcow2,vdi,iso,ploop.root-tar

[keystone_authtoken]
www_authenticate_uri = http://internal.java.dama.id:5000
auth_url = http://internal.java.dama.id:5000
memcached_servers = 10.10.10.11:11211,10.10.10.12:11211,10.10.10.13:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance!dama

[os_glance_tasks_store]
filesystem_store_datadir = /var/lib/glance/tasks_work_dir

[os_glance_staging_store]
filesystem_store_datadir = /var/lib/glance/staging

[oslo_middleware]
enable_proxy_headers_parsing = True

[paste_deploy]
flavor = keystone


[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]

[oslo_messaging_notifications]
transport_url = rabbit://openstack:rabbit!dama@10.10.10.11:5672,openstack:rabbit!dama@10.10.10.12:5672,openstack:rabbit!dama@10.10.10.13:5672//
driver = noop

[oslo_messaging_rabbit]
[oslo_policy]


[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]
```
{% endcode %}

3. Add haproxy configuration

```
vi /etc/haproxy/haproxy.cfg
...
# GLANCE
listen glance_api_cluster
  bind 10.10.10.100:9292
  bind 202.10.10.100:9292
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
    server os-controller-01 10.10.10.11:9292 check inter 2000 rise 2 fall 5
    server os-controller-02 10.10.10.12:9292 check inter 2000 rise 2 fall 5
    server os-controller-03 10.10.10.13:9292 check inter 2000 rise 2 fall 5
```

## Populate Glance Database (Exec on Controller-01)

```bash
su -s /bin/sh -c "glance-manage db_sync" glance
```

## Restart and enable Glance Service (Exec on all controller nodes)

1. Restart and enable service

```
systemctl restart glance-api 
systemctl enable glance-api
systemctl status glance-api 
```

2. Restart haproxy with pcs

```
pcs resource restart lb-haproxy
```

3. Verify glance port listen

```
ss -l '( sport = :9292 )'
```

