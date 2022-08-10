# Cấu hình openstack trong kolla
```
network:
  ethernets:
    ens192:
      addresses:
      - 172.10.10.150/24
    ens160:
      dhcp4: no
  bridges:
    br-ex:
      addresses:
      - 10.2.65.150/24
      gateway4: 10.2.65.1
      nameservers:
        addresses:
        - 8.8.8.8
      interfaces:
        - ens160
  version: 2

```
File cấu hình các service trong kolla là `/etc/kolla/globals.yaml`

```
workaround_ansible_issue_8743: yes
kolla_base_distro: "ubuntu"
kolla_internal_vip_address: "172.10.10.135"
network_interface: "ens192"
neutron_external_interface: "ens160"
neutron_plugin_agent: "openvswitch"
keepalived_virtual_router_id: "77"
kolla_enable_tls_external: "{{ kolla_enable_tls_internal if kolla_same_external_internal_vip | bool else 'no' }}"
enable_openstack_core: "yes"
enable_haproxy: "yes"
enable_keepalived: "{{ enable_haproxy | bool }}"
enable_keystone: "{{ enable_openstack_core | bool }}"
enable_mariadb: "yes"
enable_memcached: "yes"
enable_neutron: "{{ enable_openstack_core | bool }}"
enable_nova: "{{ enable_openstack_core | bool }}"
enable_rabbitmq: "{{ 'yes' if om_rpc_transport == 'rabbit' or om_notify_transport == 'rabbit' else 'no' }}"
enable_cinder: "yes"
enable_cinder_backup: "no"
enable_cinder_backend_lvm: "yes"
enable_fluentd: "yes"
enable_heat: "{{ enable_openstack_core | bool }}"
enable_horizon: "{{ enable_openstack_core | bool }}"
enable_horizon_heat: "{{ enable_heat | bool }}"
enable_horizon_magnum: "{{ enable_magnum | bool }}"
enable_magnum: "yes"
enable_neutron_provider_networks: "yes"
enable_nova_ssh: "yes"
cinder_volume_group: "cinder-volumes"
nova_compute_virt_type: "qemu"

```

Cấu hình multinode
```
[control]
server-01 ansible_user=root ansible_password=1 ansible_become=true
[network]
server-01 ansible_user=root ansible_password=1 ansible_become=true
[compute]
server-02 ansible_user=root ansible_password=1 ansible_become=true
server-03 ansible_user=root ansible_password=1 ansible_become=true
[monitoring]
server-01 ansible_user=root ansible_password=1 ansible_become=true
[storage]
server-02


```
## install environment
```
sudo apt update
sudo apt install python3-dev libffi-dev gcc libssl-dev
sudo apt install python3-venv
python3 -m venv /path/to/venv
source /path/to/venv/bin/activate
pip install -U pip
pip install 'ansible>=4,<6'

sudo apt install ansible
pip install git+https://opendev.org/openstack/kolla-ansible@master

sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

cp -r /path/to/venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp /path/to/venv/share/kolla-ansible/ansible/inventory/* .

kolla-ansible install-deps

cấu hình trong /etc/ansible/ansible.cfg:
[defaults]
host_key_checking=False
pipelining=True
forks=100

ansible -i multinode all -m ping
kolla-genpwd


kolla-ansible -i ./multinode bootstrap-servers
kolla-ansible -i ./multinode prechecks
kolla-ansible -i ./multinode deploy

cấu hình CLI
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/master

/path/to/venv/share/kolla-ansible/init-runonce

```


## download images
```
$ wget https://download.fedoraproject.org/pub/alt/atomic/stable/Fedora-Atomic-27-20180419.0/CloudImages/x86_64/images/Fedora-Atomic-27-20180419.0.x86_64.qcow2

## create image
$ openstack image create \
                      --disk-format=qcow2 \
                      --container-format=bare \
                      --file=Fedora-Atomic-27-20180419.0.x86_64.qcow2\
                      --property os_distro='fedora-atomic' \
                      fedora-atomic-latest
```
### Template Docker Swarm cluster
```
openstack coe cluster template create swarm-cluster-template \
                     --image fedora-atomic-latest \
                     --external-network public \
                     --dns-nameserver 8.8.8.8 \
                     --master-flavor m1.small \
                     --flavor m1.small \
                     --coe swarm

### create Docker Swarm cluster
openstack coe cluster create swarm-cluster \
                        --cluster-template swarm-cluster-template \
                        --master-count 1 \
                        --node-count 1 \
                        --keypair mykey
                        
openstack coe cluster list
openstack coe cluster show swarm-cluster
```
### Template k8s cluster
```
openstack coe cluster template create kubernetes-cluster-template \
                     --image fedora-atomic-latest \
                     --external-network public \
                     --dns-nameserver 8.8.8.8 \
                     --master-flavor m1.small \
                     --flavor m1.small \
                     --coe kubernetes
###  create k8s cluster
openstack coe cluster create kubernetes-cluster \
                        --cluster-template kubernetes-cluster-template \
                        --master-count 1 \
                        --node-count 1 \
                        --keypair mykey
                        
openstack coe cluster list
openstack coe cluster show kubernetes-cluster
=> Add the credentials of the above cluster to your environment:
$ mkdir -p ~/clusters/kubernetes-cluster
$ $(openstack coe cluster config kubernetes-cluster --dir ~/clusters/kubernetes-cluster)
==> The above command will save the authentication artifacts in the directory ~/clusters/kubernetes-cluster and it will export the KUBECONFIG environment variable:
export KUBECONFIG=/home/user/clusters/kubernetes-cluster/config
```

link tham khảo: 
- https://docs.openstack.org/magnum/ussuri/install/launch-instance.html#upload-the-images-required-for-your-clusters-to-the-image-service
- https://computingforgeeks.com/install-fedora-coreos-fcos-on-kvm-openstack/






Dựa vào file cấu hình này mà ansible sẽ tạo ra file config đối với các service.

Chúng ta sẽ tìm hiểu về nội dung cấu hình trong file này.


## 1. Cấu hình kolla
```sh
###################
# Kolla options
###################

# Valid options are [ centos, oraclelinux, ubuntu ]
#- kolla_base_distro: "centos"`

# Valid options are [ binary, source ]
#kolla_install_type: "binary"

# Valid option is Docker repository tag
#openstack_release: ""

# This should be a VIP, an unused IP on your network that will float between
# the hosts running keepalived for high-availability. When running an All-In-One
# without haproxy and keepalived, this should be the first IP on your
# 'network_interface' as set in the Networking section below.
- `kolla_internal_vip_address: "10.10.10.254"
```

- `kolla_base_distro: "centos"`: Image các service trong openstack dựa trên nền distro nào?
- `kolla_install_type: "binary"`: Image các service có dưới 2 dạng là binary và source. Tham khảo tại repo chứa image trên docker. https://hub.docker.com/u/kolla/
- `openstack_release`: Phiên bản image của các service. Mặc định nó sẽ lấy phiên bản mới nhất. Phiên bản mới nhất hiện tại là 4.0.0.
- `kolla_internal_vip_address: "10.10.10.254"`: Địa chỉ VIP chạy HA proxy và keepalived.

## 2. Cấu hình docker.

```sh
####################
# Docker options
####################

#docker_registry: "172.16.0.10:4000"
#docker_namespace: "companyname"
#docker_registry_username: "sam"
#docker_registry_password: "correcthorsebatterystaple"
```

- Cấu hình repository là nơi chứa image của các service openstack. Nếu không cấu hình, mặc định docker sẽ pull các image từ docker hub. https://hub.docker.com/u/kolla/

## 3. Cấu hình Network
```sh
###############################
# Neutron - Networking Options
###############################

#network_interface: "eth0"
#`neutron_external_interface: "eth1"
# Valid options are [ openvswitch, linuxbridge ]
#neutron_plugin_agent: "openvswitch"
```

- `network_interface`: interface kết nối các service trên các node.
- `neutron_external_interface`: interface kết nối ra mạng external.
- `neutron_plugin_agent`: Sử dụng giải pháp network nào? Có 2 lựa chọn là openvswitch hoặc linuxbridge.

## 4. Cấu hình OpenStack
```sh
####################
# OpenStack options
####################

# Valid options are [ novnc, spice ]
#nova_console: "novnc"
# OpenStack services can be enabled or disabled with these options
#enable_aodh: "no"
#enable_barbican: "no"
#enable_ceilometer: "no"
#enable_central_logging: "no"
#enable_ceph: "no"
#enable_ceph_rgw: "no"
#enable_chrony: "no"
#enable_cinder: "no"
#enable_cinder_backend_hnas_iscsi: "no"
#enable_cinder_backend_hnas_nfs: "no"
#enable_cinder_backend_iscsi: "no"
#enable_cinder_backend_lvm: "no"
#enable_cinder_backend_nfs: "no"
#enable_cloudkitty: "no"
#enable_collectd: "no"
#enable_congress: "no"
#enable_designate: "no"
#enable_destroy_images: "no"
#enable_etcd: "no"
#enable_freezer: "no"
#enable_gnocchi: "no"
#enable_grafana: "no"
#enable_heat: "yes"
#enable_horizon: "yes"
```

- `nova_console: "novnc": Console đến nova bằng phương pháp nào?
- `enable_horizon: "yes"``: Cấu hình sẽ cài đặt các service nào. (yes or no).

### 4.1 Cấu hình keystone
```sh
##############################
# Keystone - Identity Options
##############################

# Valid options are [ uuid, fernet ]
#`keystone_token_provider: 'uuid'`
#fernet_token_expiry: 86400
```

- `keystone_token_provider: 'uuid'`: Sử dụng dạng token nào.
- `fernet_token_expiry: 86400`: Thời gian hết hạn của token fernet.

## 4.2 Cấu hình glance
```sh
#########################
# Glance - Image Options
#########################
# Configure image backend.
#glance_backend_file: "yes"
#glance_backend_ceph: "no"
```

- Cấu hình backend chứa image là file hay ceph?

## 4.3 Cấu hình nova

```sh
#########################
# Nova - Compute Options
#########################
#nova_backend_ceph: "{{ enable_ceph }}"
```

Cấu hình nova có sử dụng ceph không?

## 4.4 Cấu hình cinder
```sh
#################################
# Cinder - Block Storage Options
#################################
# Enable / disable Cinder backends
#cinder_backend_ceph: "{{ enable_ceph }}"
#cinder_volume_group: "cinder-volumes"
#cinder_backup_driver: "nfs"
#cinder_backup_share: ""
#cinder_backup_mount_options_nfs: ""
```

Cấu hình cinder có sử dụng ceph?

#### Ngoài ra còn cấu hình các dịch vụ khác. Các bạn tham khảo file mẫu `globals.yaml` trong thư mục `file_config`.
