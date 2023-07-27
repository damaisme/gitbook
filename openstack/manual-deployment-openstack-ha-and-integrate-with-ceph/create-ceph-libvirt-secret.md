# Create ceph libvirt secret

## Execute on Compute Nodes

1. Generate random uuid

```
$ uuid
9de5fa70-0706-11ee-a8cf-d326bea0ef5d
```

2. Create secret file

```
cat << EOF > /tmp/secret.xml
<secret ephemeral='no' private='no'>
  <uuid>9de5fa70-0706-11ee-a8cf-d326bea0ef5d</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF
```

3. Define virsh secret

```
virsh secret-define --file /tmp/secret.xml
```

4. Set secret value with cinder keyring

{% code overflow="wrap" %}
```bash
virsh secret-set-value --secret 9de5fa70-0706-11ee-a8cf-d326bea0ef5d --base64 $(cat /etc/ceph/ceph.client.cinder.keyring  | grep key | awk '{print $3}')
```
{% endcode %}
