
#clone_repo

```shell
git clone --recurse-submodules -j4 https://github.com/rackerlabs/genestack /opt/genestack
```

#setup_local_deploy_node
``` shell
export GENESTACK_PRODUCT=openstack-flex
/opt/genestack/bootstrap.sh
```

#move_inventory_file_into_place

copy inventory file

#fixup_fqdn
```shell
source /opt/genestack/scripts/genestack.rc

ansible -m shell -a 'hostnamectl set-hostname {{ inventory_hostname }}' --become  --user tim --extra-vars='ansible_become_pass=tim' all
```

#setup_hosts_for_kubes
```shell
cd /opt/genestack/ansible/playbooks
ansible-playbook  --user tim --extra-vars='ansible_become_pass=tim' --become host-setup.yml
```

#setup_kubes
``` shell
cd /opt/genestack/submodules/kubespray

ansible-playbook  --user tim --extra-vars='ansible_become_pass=tim' --become cluster.yml 
```

#install_ovn

nano /etc/genestack/helm-configs/kube-ovn/kube-ovn-helm-overrides.yaml

```
networking:
  IFACE: "eth1"
  vlan:
    VLAN_INTERFACE_NAME: "eth1"
```

#label_ovn_nodes

```shell
kubectl label node -l beta.kubernetes.io/os=linux kubernetes.io/os=linux
kubectl label node -l node-role.kubernetes.io/control-plane kube-ovn/role=master
kubectl label node -l ovn.kubernetes.io/ovs_dp_type!=userspace ovn.kubernetes.io/ovs_dp_type=kernel
```

#install_ovn 
```shell
/opt/genestack/bin/install-kube-ovn.sh
```

#validate
``` shell
kubectl get nodes
```

_________
#optional_install_k9s

``` shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

``` shell
brew install derailed/k9s/k9s
```


-----
#label_nodes_for_kubes

``` shell
kubectl label node $(kubectl get nodes | awk '/controller/ {print $1}') openstack-control-plane=enabled

kubectl label node $(kubectl get nodes | awk '/compute/ {print $1}') openstack-compute-node=enabled

kubectl label node $(kubectl get nodes | awk '/network/ {print $1}') openstack-network-node=enabled

kubectl label node $(kubectl get nodes | awk '/storage/ {print $1}') openstack-storage-node=enabled

kubectl label node $(kubectl get nodes | awk '/compute/ {print $1}') openstack-network-node=enabled

kubectl label node $(kubectl get nodes | awk '/controller/ {print $1}')  node-role.kubernetes.io/worker=worker

kubectl label node $(kubectl get nodes | awk '/network/ {print $1}')  node-role.kubernetes.io/worker=worker

kubectl label node $(kubectl get nodes | awk '/ceph/ {print $1}') role=storage-node
```

#install_prometheus
``` shell
/opt/genestack/bin/install-prometheus.sh
```

#compile_openstack_helm
``` shell
cd /opt/genestack/submodules/openstack-helm &&
make all

cd /opt/genestack/submodules/openstack-helm-infra &&
make all
```

#install_rook_for_CSI_storage

``` shell
kubectl apply -k /etc/genestack/kustomize/rook-operator/

kubectl -n rook-ceph set image deploy/rook-ceph-operator rook-ceph-operator=rook/ceph:v1.13.7
```


> [!Wait for operator] Wait for the operator to become ready
> kubectl -n rook-ceph get pods -w


#install_rook_and_configure_storage

``` shell
kubectl apply -k /etc/genestack/kustomize/rook-cluster/overlay
```

#Monitor_build
```shell
kubectl --namespace rook-ceph get cephclusters.ceph.rook.io
```

#apply_storage_policys_for_cluster

```shell
kubectl apply -k /etc/genestack/kustomize/rook-defaults
```


---------
#generate_openstack_secrets_and_load
```shell
kubectl apply -k /etc/genestack/kustomize/openstack

/opt/genestack/bin/create-secrets.sh

kubectl create -f /etc/genestack/kubesecrets.yaml
```

#install_metallb

``` shell
kubectl apply -f /etc/genestack/manifests/metallb/metallb-namespace.yaml

/opt/genestack/bin/install-metallb.sh
```


#edit_file_and_update_vip 
/etc/genestack/manifests/metallb/metallb-openstack-service-lb.yml 

VIP=10.1.1.10

#apply_metallb_vip

```shell
kubectl apply -f /etc/genestack/manifests/metallb/metallb-openstack-service-lb.yml
```


--------
#install_gateway_api



```
kubectl create ns nginx-gateway

kubectl apply -f /opt/genestack/manifests/nginx-gateway/nginx-gateway-namespace.yaml

kubectl kustomize "https://github.com/nginxinc/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.4.0" | kubectl apply -f -

echo "---" | tee /etc/genestack/helm-configs/nginx-gateway-fabric/helm-overrides.yaml

pushd /opt/genestack/submodules/nginx-gateway-fabric/charts || exit 1
helm upgrade --install nginx-gateway-fabric ./nginx-gateway-fabric \
    --namespace=nginx-gateway \
    --create-namespace \
    -f /opt/genestack/base-helm-configs/nginx-gateway-fabric/helm-overrides.yaml \
    -f /etc/genestack/helm-configs/nginx-gateway-fabric/helm-overrides.yaml \
    --post-renderer /etc/genestack/kustomize/kustomize.sh \
    --post-renderer-args gateway/overlay
popd || exit 1

kubectl rollout restart deployment cert-manager --namespace cert-manager

kubectl kustomize /etc/genestack/kustomize/gateway/nginx-gateway-fabric | kubectl apply -f -
```

#patch_listeners_and_routes


> [!note] Edit FQDN for your use case
> sed 's/your.domain.tld/YOUR_DOMAIN_GOES_HERE/g' 


```shell
mkdir -p /etc/genestack/gateway-api/listeners
for listener in $(ls -1 /opt/genestack/etc/gateway-api/listeners); do
    sed 's/your.domain.tld/underworld.local/g' /opt/genestack/etc/gateway-api/listeners/$listener > /etc/genestack/gateway-api/listeners/$listener
done

kubectl patch -n nginx-gateway gateway flex-gateway \
              --type='json' \
              --patch="$(jq -s 'flatten | .' /etc/genestack/gateway-api/listeners/*)"

mkdir -p /etc/genestack/gateway-api/routes
for route in $(ls -1 /opt/genestack/etc/gateway-api/routes); do
    sed 's/your.domain.tld/underworld.local/g' /opt/genestack/etc/gateway-api/routes/$route > /etc/genestack/gateway-api/routes/$route
done
```

#apply_routes_and_listeners

```shell
kubectl apply -f /etc/genestack/gateway-api/routes
```

-------
#mariadb

```shell
cluster_name=`kubectl config view --minify -o jsonpath='{.clusters[0].name}'`
echo $cluster_name

/opt/genestack/bin/install-mariadb-operator.sh
```

#wait_for_webhook
```shell

kubectl --namespace mariadb-system get pods -w
```

#install_mariadb
```shell

kubectl --namespace openstack apply -k /etc/genestack/kustomize/mariadb-cluster/overlay
```

#verify_healthy_cluster
```shell
kubectl --namespace openstack get mariadbs -w
```


______
#rabbitmq

```shell
kubectl apply -k /etc/genestack/kustomize/rabbitmq-operator

kubectl apply -k /etc/genestack/kustomize/rabbitmq-topology-operator

kubectl apply -k /etc/genestack/kustomize/rabbitmq-cluster/overlay
```

#validate

```shell
kubectl --namespace openstack get rabbitmqclusters.rabbitmq.com -w
```

_____
#memcache

```shell
/opt/genestack/bin/install-memcached.sh

kubectl --namespace openstack get horizontalpodautoscaler.autoscaling memcached -w
```

_____
#libvirt
```shell
/opt/genestack/bin/install-libvirt.sh
```

____
#ovn_for_openstack


> [!NOTE] Edit eth2 with interface of provider network
>ovn.openstack.org/ports='br-ex:eth2

```shell

export ALL_NODES=$(kubectl get nodes -l 'openstack-network-node=enabled' -o 'jsonpath={.items[*].metadata.name}')

kubectl annotate \
         nodes \
         ${ALL_NODES} \
         ovn.openstack.org/int_bridge='br-int'

kubectl annotate \
        nodes \
        ${ALL_NODES} \
        ovn.openstack.org/bridges='br-ex'

kubectl annotate \
        nodes \
        ${ALL_NODES} \
        ovn.openstack.org/ports='br-ex:eth2'
    
    
kubectl annotate \
        nodes \
        ${ALL_NODES} \
        ovn.openstack.org/mappings='physnet1:br-ex'
kubectl annotate \
        nodes \
        ${ALL_NODES} \
        ovn.openstack.org/availability_zones='az1'

kubectl annotate \
        nodes \
        ${ALL_NODES} \
        ovn.openstack.org/gateway='enabled'


kubectl apply -k /etc/genestack/kustomize/ovn
```

___

#fluentbit
``` shell
/opt/genestack/bin/install-fluentbit.sh
```

____
#openstack 

```shell
/opt/genestack/bin/setup-openstack.sh
```

#install_client_admin_pod

```shell
kubectl --namespace openstack apply -f /etc/genestack/manifests/utils/utils-openstack-client-admin.yaml
```

#validate_keystone

```shell
kubectl --namespace openstack exec -ti openstack-admin-client -- openstack user list
```

