---
description: How to install memcached
---

# Setup Memcached

Memcached is typically used by high-speed components like Keystone (Identity Service) and Nova (Compute Service) in OpenStack to store frequently accessed temporary data, such as authentication information and instance status. By utilizing Memcached, OpenStack components can retrieve data from the cache faster than accessing the original data source, which is typically persistent storage like a database.



## Install Memcached (All controller node)

1. Install memcached package

```
apt install memcached
```

2. Change listen address

{% code overflow="wrap" %}
```
sed -i "s/127.0.0.1/$(ip -o -4 addr show dev ens4 | awk '{split($4,a,"/"); print a[1]}')/g" /etc/memcached.conf
```
{% endcode %}

3. Enable memcached service

```
systemctl enable --now memcached
systemctl restart memcached
```

