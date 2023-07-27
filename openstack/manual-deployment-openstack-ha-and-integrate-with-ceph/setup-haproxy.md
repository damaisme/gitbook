# Setup HAproxy

HAProxy is a commonly used load balancer and proxy server that can be deployed in a high-availability (HA) configuration in an OpenStack environment. It helps distribute incoming traffic across multiple backend servers, providing redundancy, scalability, and improved performance.



## Setup HAproxy (Execute on all controller nodes)

1. Install haproxy

```
apt install haproxy -y 
```

2. Enable ip forward and ip nonlocal bind kernel parameter

```
cat << EOF > /etc/sysctl.d/50-haproxy.conf
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
EOF

sysctl -p /etc/sysctl.d/50-haproxy.conf
```

3. Generate self signed certificate

{% code overflow="wrap" %}
```
openssl req -x509 -new -nodes -newkey rsa:2048 -keyout root.key -sha256 -days 1024 -out root.crt -subj "/OU=DAMA ID/CN=dama.id"

openssl req -nodes -newkey rsa:2048 -keyout java.dama.id.key -out java.dama.id.csr -subj "/OU=DAMA ID/CN=*.java.dama.id"

openssl x509 -req -in java.dama.id.csr -CA root.crt -CAkey root.key -CAcreateserial -out java.dama.id.crt -days 500 -sha256

cat java.dama.id.crt java.dama.id.key > /etc/ssl/certs/java.dama.id.pem
```
{% endcode %}

4. Distribute cert to other controllers node

```
scp /etc/ssl/certs/java.dama.id.pem os-controller-01:/etc/ssl/certs/
scp /etc/ssl/certs/java.dama.id.pem os-controller-02:/etc/ssl/certs/
```

5. Generate user for haproxy mysql health check

```
mysql
CREATE USER 'haproxy'@'localhost';
CREATE USER 'haproxy'@'%';
```

6. Edit haproxy configuration

```
global
  chroot  /var/lib/haproxy
  daemon
  group  haproxy
  maxconn  4000
  pidfile  /var/run/haproxy.pid
  user  haproxy

defaults
  log  global
  maxconn  4000
  option  redispatch
  retries  3
  timeout  http-request 10s
  timeout  queue 1m
  timeout  connect 10s
  timeout  client 480m
  timeout  server 480m
  timeout  check 10s

# HAPROXY Status Page

listen stats
  bind *:1945
  mode http
  stats enable
  stats hide-version
  stats uri /stats
  stats refresh 10s
  stats show-node

# GALERA
listen galera_cluster
  bind 10.10.10.100:3306
  balance  source
  option  tcpka
  mode tcp
  option mysql-check user haproxy
        server os-controller-01 10.10.10.11:3306 check weight 1
        server os-controller-02 10.10.10.12:3306 check weight 1
        server os-controller-03 10.10.10.13:3306 check weight 1

```

7. Restart haproxy service

```
systemctl restart haproxy
```

