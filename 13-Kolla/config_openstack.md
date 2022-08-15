# Cấu hình openstack trong kolla
```
network:
  ethernets:
    ens192:
      addresses:
      - 172.10.10.150/24
    ens160:
      addresses:
      - 10.2.65.150/24
      gateway4: 10.2.65.1
      nameservers:
        addresses:
        - 8.8.8.8
  version: 2
```
## add route
```
ip route del 0.0.0.0/24
ip route del default via 10.2.65.1

ip route add 10.2.65.1 via br-ex
ip route add default via 10.2.65.1
```
Cấu hình cinder
```
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb

## edit /etc/lvm.conf
devices {
...
filter = [ "a/sdb/", "r/.*/"]
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
kolla-ansible post-deploy
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
                     
pip3 install python-magnumclient                         
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
- https://docs.openstack.org/kolla-ansible/latest/user/operating-kolla.html
