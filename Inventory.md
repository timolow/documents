```

all:
  hosts:
    genestack-controller1.lab.underworld.local:
      ansible_host: 10.1.1.200
    genestack-controller2.lab.underworld.local:
      ansible_host: 10.1.1.201
    genestack-controller3.lab.underworld.local:
      ansible_host: 10.1.1.202
    genestack-network1.lab.underworld.local:
      ansible_host: 10.1.1.203
    genestack-network2.lab.underworld.local:
      ansible_host: 10.1.1.204
    genestack-network3.lab.underworld.local:
      ansible_host: 10.1.1.205
    genestack-storage1.lab.underworld.local:
      ansible_host: 10.1.1.206
    genestack-ceph1.lab.underworld.local:
      ansible_host: 10.1.1.209
    genestack-ceph2.lab.underworld.local:
      ansible_host: 10.1.1.210
    genestack-ceph3.lab.underworld.local:
      ansible_host: 10.1.1.211
    genestack-compute1.lab.underworld.local:
      ansible_host: 10.1.1.212
    

  children:
    k8s_cluster:
      vars:
        kube_ovn_iface: eth1  # see the netplan snippet in etc/netplan/default-DHCP.yaml for more info.
        kube_ovn_default_interface_name: eth1  # see the netplan snippet in etc/netplan/default-DHCP.yaml for more info.
        kube_ovn_central_hosts: "{{ groups['ovn_network_nodes'] }}"
      children:
        kube_control_plane:  # all k8s control plane nodes need to be in this group
          hosts:
            genestack-controller1.lab.underworld.local: null
            genestack-controller2.lab.underworld.local: null
            genestack-controller3.lab.underworld.local: null
        etcd:  # all etcd nodes need to be in this group
          hosts:
            genestack-controller1.lab.underworld.local: null
            genestack-controller2.lab.underworld.local: null
            genestack-controller3.lab.underworld.local: null
        kube_node:  # all k8s enabled nodes need to be in this group
          hosts:
            genestack-controller1.lab.underworld.local: null
            genestack-controller2.lab.underworld.local: null
            genestack-controller3.lab.underworld.local: null
            genestack-storage1.lab.underworld.local: null
            genestack-network1.lab.underworld.local: null
            genestack-network2.lab.underworld.local: null
            genestack-network3.lab.underworld.local: null
            genestack-ceph1.lab.underworld.local: null
            genestack-ceph2.lab.underworld.local: null
            genestack-ceph3.lab.underworld.local: null
            genestack-compute1.lab.underworld.local: null


        openstack_control_plane:  # nodes used for nova compute labeled as openstack-control-plane=enabled
          hosts:
            genestack-controller1.lab.underworld.local: null
            genestack-controller2.lab.underworld.local: null
            genestack-controller3.lab.underworld.local: null
        ovn_network_nodes:  # nodes used for nova compute labeled as openstack-network-node=enabled
          hosts:
            genestack-network1.lab.underworld.local: null
            genestack-network2.lab.underworld.local: null
            genestack-network3.lab.underworld.local: null
        storage_nodes:
          children:
            ceph_storage_nodes:  # nodes used for ceph storage labeled as role=storage-node
              hosts:
                genestack-ceph1.lab.underworld.local: null
                genestack-ceph2.lab.underworld.local: null
                genestack-ceph3.lab.underworld.local: null
            cinder_storage_nodes:  # nodes used for cinder storage labeled as openstack-storage-node=enabled
              hosts:
                genestack-storage1.lab.underworld.local: null
        nova_compute_nodes:  # nodes used for nova compute labeled as openstack-compute-node=enabled
          hosts:
            genestack-compute1.lab.underworld.local: null
```
