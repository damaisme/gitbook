# Setup Rabbitmq Cluster

## Install Rabbitmq (Exec on all controller nodes)

1. Install rabbitmq

```
apt install rabbitmq-server -y
```

2. Change rabbitmq listen address

```
sed -i "s/\#NODE_IP_ADDRESS=127.0.0.1/NODE_IP_ADDRESS=$(ip -o -4 addr show dev ens4 | awk '{split($4,a,"/"); print a[1]}')/g"  /etc/rabbitmq/rabbitmq-env.conf
```

3. Start and enable rabbitmq service

```
systemctl enable --now rabbitmq-server
```



## Distribute Erlang Cookie (Exec on controller-01)

1. Distribute erlang cookie to other controller nodes

```
scp /var/lib/rabbitmq/.erlang.cookie os-controller-02:/var/lib/rabbitmq/
scp /var/lib/rabbitmq/.erlang.cookie os-controller-03:/var/lib/rabbitmq/
```



## Join Cluster (Exec on controller-02 and controlloer-03

1. join rabbitmq controller-02 and controller-03 to controller-01

```
systemctl restart rabbitmq-server
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@os-controller-01
rabbitmqctl start_app
```



## Set HA Mode and Create User for Openstack (Exec on controller-01)

1. Set HA mode

```
rabbitmqctl set_policy ha-all '^(?!amq\.).*' '{"ha-mode": "all"}'
```

2. Check cluster status

```
rabbitmqctl cluster_status
```

3. Create openstack user

```
rabbitmqctl add_user openstack 'rabbit!dama'

rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```





