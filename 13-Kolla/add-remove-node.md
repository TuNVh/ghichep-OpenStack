### Adding new controllers
```
kolla-ansible -i ./multinode bootstrap-servers --limit compute2
kolla-ansible -i ./multinode pull --limit compute2
kolla-ansible -i ./multinode deploy --limit compute2
```

https://docs.openstack.org/kolla-ansible/latest/user/adding-and-removing-hosts.html
