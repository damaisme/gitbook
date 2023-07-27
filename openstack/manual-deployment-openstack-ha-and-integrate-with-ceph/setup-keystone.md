# Setup Keystone

## Setup Keystone Database (Exec on controller-01)

1. Create keystone database

```
mysql
CREATE DATABASE keystone;
```

2. Grant keystone user for any host access

```
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'keystone!dama';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'keystone!dama';


FLUSH PRIVILEGES;
EXIT;
```



## Install and Configure Keystone (Exec on all controller nodes)

1. Install keystone packages

```
apt install -y keystone python3-openstackclient
```

2. Create koystone configuration

{% code overflow="wrap" %}
```
vi /etc/keystone/keystone.conf

[DEFAULT]
debug = False
transport_url = rabbit://openstack:rabbit!dama@10.10.10.11:5672,openstack:rabbit!dama@10.10.10.12:5672,openstack:rabbit!dama@10.10.10.13:5672//
use_stderr = True

[application_credential]
[assignment]
[auth]

[cache]
backend = oslo_cache.memcache_pool
enabled = True
memcached_servers = 10.10.10.11:11211,10.10.10.12:11211,10.10.10.13:11211

[catalog]
[cors]
[credential]

[database]
connection = mysql+pymysql://keystone:keystone!dama@10.10.10.100/keystone
max_retries = -1

[domain_config]
[endpoint_filter]
[endpoint_policy]

[eventlet_server]
bind_host = 10.10.10.X
public_bind_host = 202.10.10.X
admin_bind_host = 10.10.10.X

[federation]
[fernet_tokens]
[healthcheck]
[identity]
[identity_mapping]
[ldap]
[matchmaker_redis]
[memcache]
[oauth1]
[oslo_messaging_amqp]
[oslo_messaging_kafka]

[oslo_messaging_notifications]
transport_url = rabbit://openstack:rabbit!dama@10.10.10.11:5672,openstack:rabbit!dama@10.10.10.12:5672,openstack:rabbit!dama@10.10.10.13:5672//
driver = noop

[oslo_messaging_rabbit]
[oslo_messaging_zmq]

[oslo_middleware]
enable_proxy_headers_parsing = True

[oslo_policy]
[paste_deploy]
[policy]
[profiler]
[resource]
[revoke]
[role]
[saml]
[security_compliance]
[shadow_users]
[signing]

[token]
provider = fernet


[tokenless_auth]
[trust]
[unified_limit]
```
{% endcode %}



## Bootsraping Keystone (Exec on Controller-01)

1. Populate keystone database

```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

2. Initialize fernet

```bash
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```

3. Initialize credential

```
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

4. Distribute fernet to other controller nodes

```bash
ssh os-controller-02 "mkdir /etc/keystone/credential-keys"
ssh os-controller-03 "mkdir /etc/keystone/credential-keys"

cd /etc/keystone/credential-keys
scp 0 1 os-controller-02:/etc/keystone/credential-keys/
scp 0 1 os-controller-03:/etc/keystone/credential-keys/

cd /etc/keystone/fernet-keys
scp 0 1 os-controller-02:/etc/keystone/fernet-keys
scp 0 1 os-controller-03:/etc/keystone/fernet-keys

ssh os-controller-02 "chown -R keystone:keystone /etc/keystone"
ssh os-controller-03 "chown -R keystone:keystone /etc/keystone"
```

5. Bootrstraping keystone

```bash
keystone-manage bootstrap --bootstrap-password rahasia \
  --bootstrap-admin-url http://admin.java.dama.id:5000/v3/ \
  --bootstrap-internal-url http://internal.java.dama.id:5000/v3/ \
  --bootstrap-public-url http://public.java.dama.id:5000/v3/ \
  --bootstrap-region-id java
```



## Set Apache Keystone Listen (Exec on all controller nodes)

1. Change apache listen port

{% code overflow="wrap" %}
```
sed -i "s/Listen 80.*/Listen $(ip -4 addr show ens5 | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -1):80/" /etc/apache2/ports.conf
```
{% endcode %}

2. Change keystone listen address

{% code overflow="wrap" %}
```bash
sed -i "s/Listen 5000.*/Listen $(ip -4 addr show ens5 | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -1):5000/" /etc/apache2/sites-available/keystone.conf
```
{% endcode %}

3. Add haproxy configuration

```
vi /etc/haproxy/haproxy.cfg
...
# KEYSTONE
 listen keystone_cluster
  bind 10.10.10.100:5000
  bind 202.10.10.100:5000
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
    server os-controller-01 10.10.10.11:5000 check inter 2000 rise 2 fall 5
    server os-controller-02 10.10.10.12:5000 check inter 2000 rise 2 fall 5
    server os-controller-03 10.10.10.13:5000 check inter 2000 rise 2 fall 5
```



## Create PCS Apache Resource (Exec on Controller-01)

1. Create pcs resource

```
pcs resource create apache2 systemd:apache2
```

2. Create resource clone

```
 pcs resource clone apache2
```

3. Restart apache2 and haproxy from pcs

```
pcs resource restart lb-haproxy
pcs resource restart apache2-clone
```

4. Sow resource status

```
pcs status
```



## Create Project in Keystone (Exec on Controller-01)

1. Create rc file

```
vi ~/admin-openrc

export OS_USERNAME=admin
export OS_PASSWORD=rahasia
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://admin.java.dama.id:5000/v3
export OS_IDENTITY_API_VERSION=3
```

2. Apply environment variable to current shell session

```
source ~/admin-openrc
```

3. Verify identity

```
openstack token issue
```

4. Verify Openstack endpoint

```
openstack endpoint list
```

5. Create service project

```
openstack project create --domain default \
  --description "Service Project" service
```

6. Verify project

```
openstack project list
```

