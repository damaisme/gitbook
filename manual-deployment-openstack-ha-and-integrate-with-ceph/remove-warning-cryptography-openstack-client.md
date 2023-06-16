# Remove warning cryptography openstack client

<figure><img src="../.gitbook/assets/openstack client warning.png" alt=""><figcaption></figcaption></figure>

Resolution: Downgrade cryptography python package < 3.4

```bash
apt install python3-pip
pip install cryptography==3.3.2
```
