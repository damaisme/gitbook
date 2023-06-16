# Operational Test

#### Create External Network

```html
# Create external network with type flat
openstack network create --share --external \
  --provider-physical-network physpro1 \
  --provider-network-type flat provider1-net

openstack network create --share --external \
  --provider-physical-network physpro2 \
  --provider-network-type flat provider2-net

openstack subnet create --network provider1-net \
  --gateway 50.50.50.1 --no-dhcp \
  --subnet-range 50.50.50.0/24 provider1-subnet

openstack subnet create --network provider2-net \
  --gateway 60.60.60.1 --no-dhcp \
  --subnet-range 60.60.60.0/24 provider2-subnet
```

#### Create Internal Network

```html
openstack network create test-internal-net

openstack subnet create --network test-internal-net \
  --allocation-pool start=192.168.10.10,end=192.168.10.254 \
  --dns-nameserver 8.8.8.8 --gateway 192.168.10.1 \
  --subnet-range 192.168.10.0/24 test-internal-subnet
```

#### Create Router

```html
openstack router create router-test

openstack router set --external-gateway provider1-net router-test

openstack router add subnet router-test test-internal-subnet

openstack router show router-test
```

#### Create Image

```html
wget <https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img>

openstack image create --disk-format qcow2 --container-format bare \
  --public --file ./bionic-server-cloudimg-amd64.img ubuntu-bionic
```

#### Create flavor

```html
openstack flavor create --ram 2048 --disk 10 --vcpus 2 --public ram2-cpu2
```

#### Create keypair

```html
openstack keypair create --public-key ~/.ssh/id_rsa.pub controller-key
```

#### Create Security Group

```html
openstack security group create allow-all-traffic --description 'Allow All Ingress Traffic'
openstack security group rule create --protocol icmp allow-all-traffic
openstack security group rule create --protocol tcp  allow-all-traffic
openstack security group rule create --protocol udp  allow-all-traffic
```

#### Create Instance

```html
openstack server create --flavor ram2-cpu2 \
  --image ubuntu-bionic \
  --key-name controller-key \
  --security-group allow-all-traffic \
  --network test-internal-net \
  ubuntu-test
```

#### Create Floating IP

```html
openstack floating ip create provider1-net --floating-ip-address 50.50.50.10
openstack server add floating ip ubuntu-test 50.50.50.10
```

**List Server**

```
openstack server list
```



**Test SSH to Instance**

```
ssh 50.50.50.10 -l ubuntu
```
