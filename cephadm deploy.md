#install_cephadm

test-ceph1  10.1.1.122
test-ceph2 10.1.1.121
test-ceph3 10.1.1.123


```shell
curl --silent --remote-name -- https://raw.githubusercontent.com/ceph/ceph/quincy/src/cephadm/cephadm && chmod +x cephadm
```

#install_deps 

```shell
apt-get update ; apt-get install docker.io lvm2
```

#bootstrap_first_monitor
```shell
./cephadm bootstrap --mon-ip $IP
```


#copy_over_ssh_keys_and_fixup_sshd

``` shell
cat /etc/ceph/ceph.pub
```


#add_hosts_to_ceph_orch

```shell
./cephadm shell

ceph orch host add test-ceph2 10.1.1.121

ceph orch host add test-ceph3 10.1.1.123
```

#consume_all_block_devs
```shell
ceph orch apply osd --all-available-devices
```

#show_devs
``` shell
ceph orch device ls
```

#show_df
```shell
ceph osd df
rados df
```

#create_pools
```shell
ceph osd pool create general 32
ceph osd pool create general-multi-attach-data 32
ceph osd pool create general-multi-attach-metadata 32
rbd pool init general
ceph fs new general-multi-attach general-multi-attach-metadata general-multi-attach-data
```

#label_nodes
```shell
ceph orch host label add test-ceph1 mds
ceph orch host label add test-ceph2 mds
ceph orch host label add test-ceph3 mds
ceph orch host label add test-ceph1 rgw
ceph orch host label add test-ceph2 nfs
ceph orch apply mds myfs label:mds
```

#rgw 
```json
service_type: rgw
service_id: us-central-1
placement:
  label: rgw
  count_per_host: 2
spec:
  rgw_realm: default
  rgw_zone: default
  rgw_frontend_type: "beast"
  rgw_frontend_port: 8081
  rgw_frontend_extra_args:
  - "tcp_nodelay=1"
  - "max_header_size=65536"
```

#rgw_haproxy
```json
service_type: ingress.rgw-inside
service_id: rgw.us-central-1
placement:
  hosts:
    - test-ceph2
    - test-ceph3
spec:
  backend_service: rgw.us-central-1
  virtual_ip: 10.1.1.100/24
  frontend_port: 8080
  monitor_port: 8090
```


#nfs_ganesha
```json
service_type: nfs
service_id: mynfs
service_name: general-multi-attach.myfs
placement:
  count: 1
  label: nfs
spec:
  port: 2049
  virtual_ip: 10.1.1.101
```

#nfs_haproxy
```json
service_type: ingress
service_id: nfs.mynfs
service_name: ingress.nfs.mynfs
placement:
  count: 4
  label: nfs
spec:
  backend_service: nfs.mynfs
  first_virtual_router_id: 50
  monitor_port: 9010
  virtual_ip: 10.1.1.101/24
  keepalive_only: true
```


#apply_services
```shell
ceph orch apply -i $file
```


#upgrade
```shell
ceph orch upgrade start --ceph-version 18.2.4
```
