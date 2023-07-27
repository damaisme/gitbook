# Setup Galera Cluster

## Preparation (Execute on all controller nodes)

1. Install required package

{% code overflow="wrap" %}
```
apt install -y apt-transport-https software-properties-common python3-mysqldb rsync python3-pymysql
```
{% endcode %}

2. Install mariadb

```
apt install -y mariadb-server mariadb-client
```

3. Enable mariadb service

```
systemctl enable --now mariadb
```

4. Secure mariadb installation and set root password

```
mysql_secure_installation

Enter current password for root (enter for none): 

Change the root password? [Y/n] Y

Set root password? [Y/n] Y
New password: mysql!dama
Re-enter new password: mysql!dama

Remove anonymous users? [Y/n] Y

Disallow root login remotely? [Y/n] Y

Remove test database and access to it? [Y/n] Y

Reload privilege tables now? [Y/n] Y
```

5. Change mariadb listen address

{% code overflow="wrap" %}
```
sed -i "s/127.0.0.1/$(ip -4 addr show ens4 | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -1)/g" /etc/mysql/mariadb.conf.d/50-server.cnf
```
{% endcode %}

6. Create galera config

{% code overflow="wrap" %}
```
cat << EOF > /etc/mysql/mariadb.conf.d/50-galera.cnf
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=$(ip -4 addr show ens4 | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -1)

# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

wsrep_cluster_name="os_wsrep_cluster"
wsrep_cluster_address="gcomm://10.10.10.11,10.10.10.12,10.10.10.13"

wsrep_sst_method=rsync

wsrep_node_address="$(ip -4 addr show ens4 | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -1)"
wsrep_node_name="$(hostname)"
EOF
```
{% endcode %}



## Initiate Galera Cluster (Execute on controller-01)

1. Stop mariadb service

```
systemctl stop mariadb
```

2. Initiate cluster

```
galera_new_cluster
```

3. Check cluster status

```
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"

output:
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 1     |
+--------------------+-------+
```



## Join other nodes to the cluster (Exec on controller-01 and controller-02)

1. Restart mariadb service

```
systemctl restart mariadb
```

2. Check cluster status

```
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"

output:
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
```





