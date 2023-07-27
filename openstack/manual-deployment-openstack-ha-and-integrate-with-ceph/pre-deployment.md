# Pre-Deployment

## Exec on all nodes

1. Set hosts file

```
vi /etc/hosts

202.10.10.100 public.java.dama.ink
10.10.10.100 internal.java.dama.ink
10.10.10.100 admin.java.dama.ink

10.10.10.11 os-controller-01
10.10.10.12 os-controller-02
10.10.10.13 os-controller-03
10.10.10.21 os-compute-01
10.10.10.22 os-compute-02
10.10.10.23 os-compute-03
```

2. Verify the connection between nodes

```
ping -c 3 os-controller-01
ping -c 3 os-controller-02
ping -c 3 os-controller-03
ping -c 3 os-compute-01
ping -c 3 os-compute-02
ping -c 3 os-compute-03
```

3. Adding to the Focal-Yoga Ubuntu Cloud Archive (UCA) repository

```
add-apt-repository cloud-archive:yoga
apt update && apt upgrade -y 
```

4. Installing chrony (ntp)

```
# Installing chrony
apt install -y chrony

# Start and enable chrony service 
systemctl enable --now chrony
systemctl status chrony
```

5. Set up passwordless ssh

```
sed -i -r 's/#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
systemctl restart sshd

ssh-keygen

ssh-copy-id os-controller-01
ssh-copy-id os-controller-02
ssh-copy-id os-controller-03
ssh-copy-id os-compute-01
ssh-copy-id os-compute-02
ssh-copy-id os-compute-03
```
