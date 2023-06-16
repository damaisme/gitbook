# Setup Pacemaker Cluster

## Preparation (Exec on all controller nodes)

1. Install pcs package

```html
apt install pacemaker corosync fence-agents pcs resource-agents -y
```

2. Change user hacluster password

```bash
echo 'hacluster:dama!pcs' | chpasswd
```

3. Enable pcs service

```html
systemctl enable --now pcsd pacemaker corosync
```



## Initiate PCS Cluster (Exec only on controller-01)

1. Authenticate both the nodes using pcs command

{% code overflow="wrap" %}
```bash
pcs host auth os-controller-01 os-controller-02 os-controller-03
```
{% endcode %}

2. Configure cluster

{% code overflow="wrap" %}
```html
pcs cluster setup os-ha os-controller-01 os-controller-02 os-controller-03 --force
```
{% endcode %}

3. Set pcs cluster property

{% code overflow="wrap" %}
```html
pcs property set pe-warn-series-max=1000 pe-input-series-max=1000 pe-error-series-max=1000 cluster-recheck-interval=5min
```
{% endcode %}

4. Create pcs resource vip and haproxy

{% code overflow="wrap" %}
```html
pcs resource create internal_vip ocf:heartbeat:IPaddr2 ip="10.10.10.100" cidr_netmask="24" op monitor interval="30s"

pcs resource create public_vip ocf:heartbeat:IPaddr2 ip="202.10.10.100" cidr_netmask="24" op monitor interval="30s"

pcs resource create lb-haproxy systemd:haproxy op monitor interval="30s"
```
{% endcode %}

5. Define ordering and colocation constraints

{% code overflow="wrap" %}
```html
pcs constraint order start internal_vip then public_vip 

pcs constraint colocation add public_vip with internal_vip INFINITY

pcs constraint colocation add lb-haproxy with internal_vip INFINITY
```
{% endcode %}

6. Show cluster status

```
pcs status
```

7. Restart haproxy from pcs

```
pcs resource restart lb-haproxy
```



