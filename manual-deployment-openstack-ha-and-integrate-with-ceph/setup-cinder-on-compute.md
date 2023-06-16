# Setup Cinder on Compute

1. Install cinder packages

```bash
apt install -y cinder-volume
```

2. Modify cinder configuration

```bash
vi /etc/cinder/cinder.conf

[DEFAULT]
debug = False
use_forwarded_for = true
use_stderr = False
my_ip = 10.10.10.X
osapi_volume_workers = 5
volume_name_template = volume-%s
glance_api_servers = http://10.10.10.100:9292
glance_num_retries = 3
glance_api_version = 2
glance_ca_certificates_file =
os_region_name = java
enabled_backends = ceph
osapi_volume_listen = 10.10.10.X
osapi_volume_listen_port = 8776
api_paste_config = /etc/cinder/api-paste.ini
auth_strategy = keystone
transport_url = rabbit://openstack:rabbit!dama@10.10.10.11:5672,openstack:rabbit!dama@10.10.10.12:5672,openstack:rabbit!dama@10.10.10.13:5672//

[oslo_messaging_notifications]
transport_url = rabbit://openstack:rabbit!dama@10.10.10.11:5672,openstack:rabbit!dama@10.10.10.12:5672,openstack:rabbit!dama@10.10.10.13:5672//
driver = noop

[oslo_middleware]
enable_proxy_headers_parsing = True

[nova]
interface = internal
auth_url = http://internal.java.dama.id:5000/v3
auth_type = password
project_domain_id = default
user_domain_id = default
region_name = gti
project_name = service
username = nova
password = nova!dama
cafile =

[database]
connection = mysql+pymysql://cinder:cinder!dama@10.10.10.100/cinder
max_retries = -1

[keystone_authtoken]
www_authenticate_uri = http://internal.java.dama.id:5000
auth_url = http://internal.java.dama.id:5000/v3
memcached_servers = 10.10.10.11:11211,10.10.10.12:11211,10.10.10.13:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cinder
password = cinder!dama
cafile =

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = 9de5fa70-0706-11ee-a8cf-d326bea0ef5d
report_discard_supported = True
image_upload_use_cinder_backend = True

[privsep_entrypoint]
helper_command = sudo cinder-rootwrap /etc/cinder/rootwrap.conf privsep-helper --config-file /etc/cinder/cinder.conf

[coordination]
```

3. Restart and enable service cinder-volume

```
systemctl restart cinder-volume
systemctl enable cinder-volume
systemctl status cinder-volume
```

