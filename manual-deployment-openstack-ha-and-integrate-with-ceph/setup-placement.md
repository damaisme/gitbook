# Setup Placement

## Setup Placement Database (Exec on Controller-01)

1. Create mysql database for placement

```
mysql
CREATE DATABASE placement;
```

2. Grant placement user for any host access

```bash
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \\
  IDENTIFIED BY 'placement!dama';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \\
  IDENTIFIED BY 'placement!dama';

FLUSH PRIVILEGES;
EXIT;
```

## Setup Placement Endpoint (Exec on Controller-01)

1. Create placement user in OpenStack

```bash
source admin-openrc
openstack user create --domain default --password 'placement!dama' placement
```

2. Grant admin role for placement user

```bash
openstack role add --project service --user placement admin
```

3. Create placement service in openstack

```bash
openstack service create --name placement \\
  --description "Placement API" placement
```

3. Create placement public, internal and admin endpoint

```bash
openstack endpoint create --region java \
  placement public http://public.java.dama.id:8778
openstack endpoint create --region java \
  placement internal http://internal.java.dama.id:8778
openstack endpoint create --region java \
  placement admin http://admin.java.dama.id:8778
```

### Install and Confgiure Placement API (Exec on All Controller Nodes)

1. Install placement-api

```
apt install -y placement-api
```

2. Modify placement configuration

```bash
vim /etc/placement/placement.conf

[DEFAULT]
debug = False
state_path = /var/lib/placement
osapi_compute_listen = 10.10.10.X
my_ip = 10.10.10.X
transport_url = rabbit://openstack:rabbit!dama@10.10.10.11:5672,openstack:rabbit!dama@10.10.10.12:5672,openstack:rabbit!dama@10.10.10.13:5672//

[api]
use_forwarded_for = true

[oslo_middleware]
enable_proxy_headers_parsing = True

[oslo_concurrency]
lock_path = /var/lib/placement/tmp

[placement_database]
connection = mysql+pymysql://placement:placement!dama@10.10.10.100/placement
max_retries = -1

[cache]
backend = oslo_cache.memcache_pool
enabled = True
memcache_servers = 10.10.10.11:11211,10.10.10.12:11211,10.10.10.13:11211

[keystone_authtoken]
www_authenticate_uri = http://internal.java.dama.id:5000
auth_url = http://internal.java.dama.id:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = placement!dama
```

3. Change binding address

```bash
sed -i "s/Listen 8778.*/Listen $(ip -4 addr show ens4 | grep -oP '(?<=inet\\s)\\d+(\\.\\d+){3}' | head -1):8778/" /etc/apache2/sites-available/placement-api.conf
```

4. Create haproxy configuration

```
vi /etc/haproxy/haproxy.cfg
# PLACEMENT
 listen placement_internal_api_cluster
  bind 10.10.10.100:8778
  bind 202.10.10.100:8778
  balance  source
  option  tcpka
  option  tcplog
    server os-controller-01 10.10.10.11:8778 check inter 2000 rise 2 fall 5
    server os-controller-02 10.10.10.12:8778 check inter 2000 rise 2 fall 5
    server os-controller-03 10.10.10.13:8778 check inter 2000 rise 2 fall 5
```



## Populate and Enable Placement (Exec on controller-01)

1. Populate placement database

```bash
su -s /bin/sh -c "placement-manage db sync" placement
```

2. Restart apache with pcs

```
pcs resource restart apache2-clone
```

3. Restart haprocy with pcs

```
pcs resource restart lb-haproxy
```

