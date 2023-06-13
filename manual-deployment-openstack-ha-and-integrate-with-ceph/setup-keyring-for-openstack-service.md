---
description: >-
  In OpenStack, keyrings are used to authenticate OpenStack components that
  interact with Ceph. Cinder and Glance require keyrings to access and manage
  block and image storage in the Ceph cluster.
---

# Setup keyring for openstack service

1. Create osd pool

```
sudo ceph osd pool create volumes
sudo ceph osd pool create images
sudo ceph osd pool create backups
sudo ceph osd pool create vms
```

2. Initialize the pool for use by RBD

```
sudo rbd pool init volumes
sudo rbd pool init images
sudo rbd pool init backups
sudo rbd pool init vms
```

3. Create ceph user and keyring

{% code overflow="wrap" %}
```
sudo ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images' mgr 'profile rbd pool=images'
sudo ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' mgr 'profile rbd pool=volumes, profile rbd pool=vms'
sudo ceph auth get-or-create client.cinder-backup mon 'profile rbd' osd 'profile rbd pool=backups' mgr 'profile rbd pool=backups'
```
{% endcode %}

4. Distribute keyring to all nodes

{% code overflow="wrap" %}
```
for x in {01..03}
do
sudo ceph auth get-or-create client.glance | ssh os-controller-$x "sudo tee /etc/ceph/ceph.client.glance.keyring"
sudo ceph auth get-or-create client.cinder | ssh os-controller-$x "sudo tee /etc/ceph/ceph.client.cinder.keyring"
sudo ceph auth get-or-create client.cinder-backup | ssh os-controller-$x "sudo tee /etc/ceph/ceph.client.cinder-backup.keyring"

sudo ceph auth get-or-create client.glance | ssh os-compute-$x "sudo tee /etc/ceph/ceph.client.glance.keyring"
sudo ceph auth get-or-create client.cinder | ssh os-compute-$x "sudo tee /etc/ceph/ceph.client.cinder.keyring"
sudo ceph auth get-or-create client.cinder-backup | ssh os-compute-$x "sudo tee /etc/ceph/ceph.client.cinder-backup.keyring"
done
```
{% endcode %}

