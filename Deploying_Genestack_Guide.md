# Deploying Genstack test lab:

### RPC lab account: 


URL (https://mycloud.rackspace.com/cloud/776051/servers#)


### 5 vms: 3 Ctrl and 2 comp

Flavor: 15GB Compute v1
Storage Volumes: SATA 75GB X 3 for Controllers.
Add Two addtional networks to instance on creation under network and sec_groups.

### 1. copy ssh-key across nodes.

```
# ssh-copy-id
  or manually copy to authorized_keys file.
# vim .ssh/authorized_keys
```

### 2. Clone genestack repo and boot strap only on deployment node.

```
 # git clone --recurse-submodules -j4 https://github.com/rackerlabs/genestack /opt/genestack

 # /opt/genestack/bootstrap.sh
```

### 3. Modify copy sample inventory file created to create own with newly created vms names and IP's from new network created on VM deployment. Modify kube_ovn_iface to match VM's interface.

```
 # cat  /etc/genestack/inventory/inventory.yaml (example path)

 EXAMPLE portion.

 all:
  hosts:
    leo-gs-ctrl01:
      ansible_host: <IP>
    leo-gs-ctrl02:
      ansible_host: <IP>
    leo-gs-ctrl03:
      ansible_host: <IP>
    leo-gs-comp01:
      ansible_host: <IP>
    leo-gs-comp02:
      ansible_host: <IP>
  children:
    k8s_cluster:
      vars:
        cluster_name: cluster.local  # If cluster_name is modified then cluster_domain_suffix will need to be modified for all the helm charts and for infrastructure operator configs too
        kube_ovn_iface: eth2  # see the netplan snippet in etc/netplan/default-DHCP.yaml for more info.
        kube_ovn_default_interface_name: eth2  # see the netplan snippet in etc/netplan/default-DHCP.yaml for more info.
        kube_ovn_central_hosts: "{{ groups['ovn_network_nodes'] }}"
      children:
        kube_control_plane:  # all k8s control plane nodes need to be in this group
          hosts:
            leo-gs-ctrl01: null
            leo-gs-ctrl02: null
            leo-gs-ctrl03: null
        etcd:  # all etcd nodes need to be in this group
          hosts:
            leo-gs-ctrl01: null
            leo-gs-ctrl02: null
            leo-gs-ctrl03: null
        kube_node:  # all k8s enabled nodes need to be in this group
          hosts:
            leo-gs-ctrl03: null
            leo-gs-comp02: null
            leo-gs-comp01: null
            leo-gs-ctrl01: null
            leo-gs-ctrl02: null
        openstack_control_plane:  # nodes used for nova compute labeled as openstack-control-plane=enabled
          hosts:
            leo-gs-ctrl01: null
            leo-gs-ctrl02: null
            leo-gs-ctrl03: null
        ovn_network_nodes:  # nodes used for nova compute labeled as openstack-network-node=enabled
          hosts:
            leo-gs-ctrl03: null
            leo-gs-ctrl01: null
            leo-gs-ctrl02: null
            leo-gs-comp01: null
            leo-gs-comp02: null
        storage_nodes:
          children:
            ceph_storage_nodes:  # nodes used for ceph storage labeled as role=storage-node
              hosts:
                leo-gs-ctrl01: null
                leo-gs-ctrl02: null
                leo-gs-ctrl03: null
            cinder_storage_nodes:  # nodes used for cinder storage labeled as openstack-storage-node=enabled
              hosts:
                leo-gs-ctrl03: null
                leo-gs-ctrl01: null
                leo-gs-ctrl02: null
        openstack_compute_nodes:  # nodes used for nova compute labeled as openstack-compute-node=enabled
          hosts:
            leo-gs-comp01: null
            leo-gs-comp02: null

```

### 4. Ensure systems have a proper FQDN Hostname

```
  # source /opt/genestack/scripts/genestack.rc

  # ansible -m shell -a 'hostnamectl set-hostname {{ inventory_hostname }}' --become all

  # ansible -m shell -a "grep 127.0.0.1 /etc/hosts | grep -q {{ inventory_hostname }} || sed -i 's/^127.0.0.1.*/127.0.0.1 {{ inventory_hostname }} localhost.localdomain localhost/' /etc/hosts" --become all
```

### 5. Update all nodes.

```
  # ansible -m shell -a 'apt-get update'  --become all
  # ansible -m shell -a 'apt-get upgrade -y ' --become all
  # ansible -m shell -a 'reboot now' --become all
```
# (note: Do not use kernel 5.15.0-119)


### 6. Prepare hosts for Kube installation.

```
  # source /opt/genestack/scripts/genestack.rc
  # cd /opt/genestack/ansible/playbooks
  # ansible-playbook host-setup.yml -i /etc/genestack/inventory/inventory.yaml
```

### 7. Run cluster deployment.

```
  # cd /opt/genestack/submodules/kubespray

  # ansible-playbook cluster.yml -i /etc/genestack/inventory/inventory.yaml

```

### 8. Label kube nodes to respective roles.


```
  # kubectl get nodes

  # Label the storage nodes - optional and only used when deploying ceph for K8S infrastructure shared storage
  kubectl label node $(kubectl get nodes | awk '/ctrl/ {print $1}') role=storage-node

  # Label the openstack controllers
  kubectl label node $(kubectl get nodes | awk '/ctrl/ {print $1}') openstack-control-plane=enabled
  
  # Label the openstack compute nodes
  kubectl label node $(kubectl get nodes | awk '/comp/ {print $1}') openstack-compute-node=enabled
  
  # Label the openstack network nodes
  kubectl label node $(kubectl get nodes | awk '/ctrl/ {print $1}') openstack-network-node=enabled
  
  # Label the openstack storage nodes
  kubectl label node $(kubectl get nodes | awk '/ctrl/ {print $1}') openstack-storage-node=enabled
  
  # With OVN we need the compute nodes to be "network" nodes as well. While they will be configured for networking, they wont be gateways.
  kubectl label node $(kubectl get nodes | awk '/ctrl|comp/ {print $1}') openstack-network-node=enabled
  
  # Label all workers - Recommended and used when deploying Kubernetes specific services
  kubectl label node $(kubectl get nodes | awk '/ctrl|comp/ {print $1}')  node-role.kubernetes.io/worker=worker

```
### 9. Vailidater lables are set.

```
    # kubectl get nodes -o json | jq '[.items[] | {"NAME": .metadata.name, "LABELS": .metadata.labels}]'
```


### 10. Deploy K8s Dashboard RBAC.


  ```
    # kubectl apply -k /etc/genestack/kustomize/k8s-dashboard
    
    Retrieve token
    
    # kubectl get secret admin-user -n kube-system -o jsonpath={".data.token"} | base64 -d  
  ```

### 11. Remove any taints from controllers.
  
  ```
    # kubectl taint nodes $(kubectl get nodes -o 'jsonpath={.items[*].metadata.name}') node-role.kubernetes.io/control-plane:NoSchedule-

  ```


### 12. Install Kube-convert to assist with any upgrades.

   ```
    # curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl-convert"
    # sudo install -o root -g root -m 0755 kubectl-convert /usr/local/bin/kubectl-convert
   ```

### 13. Install Prometheus.

 
   URL (https://docs.rackspacecloud.com/prometheus/#install-the-prometheus-stack) 


    copy and paste bash script from url then run on master VM.

    Confirm deployment:

     ```
     # kubectl --namespace prometheus get pods -l "release=kube-prometheus-stack"
     ```

### 14. Install Helm/update charts   
     
     ```
     # cd /opt/genestack/submodules/openstack-helm &&
     make all

     # cd /opt/genestack/submodules/openstack-helm-infra &&
     make all
     ```

### 15. Install rook-ceph in-cluster.

    Deploy Operator

       ```   
       # kubectl apply -k /etc/genestack/kustomize/rook-operator/
       # kubectl -n rook-ceph set image deploy/rook-ceph-operator rook-ceph-operator=rook/ceph:v1.13.7
       ```

### 16. Set device filter for ceph.

    ```
    # vim /opt/genestack/base-kustomize/rook-cluster/rook-cluster.yaml

       Edit line
        # deviceFilter: "xvdb"
    ```

### 17. Deploy Rook-Ceph Cluster.
 
       ```
       # kubectl apply -k /opt/genestack/base-kustomize/rook-cluster/
       ```

### 18. Create storage clases
   
       ```
       # kubectl apply -k /opt/genestack/base-kustomize/rook-defaults    
       ```

### 19. Validate cluster or log into ceph-tools pod to verify cluster health.

       ```
       # kubectl --namespace rook-ceph get cephclusters.ceph.rook.io

       # kubectl exec --stdin --tty -n rook-ceph rook-ceph-tools-(pod_ID) -- /bin/bash
       ```

### 20. Installing Openstack.


        Create namespace and secrets
         
         ```
         # kubectl apply -k /opt/genestack/base-kustomize/openstack
         # /opt/genestack/bin/create-secrets.sh
         # kubectl create -f /etc/genestack/kubesecrets.yaml -n openstack
         ```


### 21. Install Gateway API for cluster.

        Controller Selection
        ```
        # kubectl create ns nginx-gateway
        ```
        Install the Gateway API Resource from Kubernetes
        
        ```
        # kubectl kustomize "https://github.com/nginxinc/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.4.0" | kubectl apply -f -
        ```

        Install the NGINX Gateway Fabric controller
        ```
        # cd /opt/genestack/submodules/nginx-gateway-fabric/charts
        # helm upgrade --install nginx-gateway-fabric ./nginx-gateway-fabric \
            --namespace=nginx-gateway \
            -f /etc/genestack/helm-configs/nginx-gateway-fabric/helm-overrides.yaml
        ```

        Once deployed ensure a system rollout has been completed for Cert Manager.
        
        ```
        # kubectl rollout restart deployment cert-manager --namespace cert-manager
        ```

        Create the shared gateway resource
        
        ```
        # kubectl kustomize /etc/genestack/kustomize/gateway/nginx-gateway-fabric | kubectl apply -f -
        ```
        
   Apply the Let's Encrypt Cluster Issuer (copy pasta)
        
```
        -------------------
read -p "Enter a valid email address for use with ACME: " ACME_EMAIL; \
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ${ACME_EMAIL}
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
            - group: gateway.networking.k8s.io
              kind: Gateway
              name: flex-gateway
              namespace: nginx-gateway
EOF
        -------------------
```
Patch Gateway with valid listeners (copy pasta) (NOTE: you can make up a test domain)

       ```
       mkdir -p /etc/genestack/gateway-api/listeners
       for listener in $(ls -1 /opt/genestack/etc/gateway-api/listeners); do
           sed 's/your.domain.tld/<YOUR_DOMAIN_HERE>/g' /opt/genestack/etc/gateway-api/listeners/$listener > /etc/genestack/gateway-api/listeners/$listener
       done
       ```

       ```
       # kubectl patch -n nginx-gateway gateway flex-gateway \
              --type='json' \
              --patch="$(jq -s 'flatten | .' /etc/genestack/gateway-api/listeners/*)"
       ```


Apply Related Gateway routes


       ```

       mkdir -p /etc/genestack/gateway-api/routes
       for route in $(ls -1 /opt/genestack/etc/gateway-api/routes); do
           sed 's/your.domain.tld/<YOUR_DOMAIN>/g' /opt/genestack/etc/gateway-api/routes/$route > /etc/genestack/gateway-api/routes/$route
       done

       ```
       ```
       # kubectl apply -f /etc/genestack/gateway-api/routes
       ```

Patch Gateway with Let's Encrypt Cluster Issuer

       ```
       # kubectl patch --namespace nginx-gateway \
              --type merge \
              --patch-file /etc/genestack/gateway-api/gateway-letsencrypt.yaml \
              gateway flex-gateway
       ```
       
Implementation with Prometheus UI

       ```
       mkdir -p /etc/genestack/gateway-api
       sed 's/your.domain.tld/<YOUR_DOMAIN>/g' /opt/genestack/etc/gateway-api/gateway-prometheus.yaml > /etc/genestack/gateway-api/gateway-prometheus.yaml
       ```
       ```
       # kubectl apply -f /etc/genestack/gateway-api/gateway-prometheus.yaml
       ```

### 22. Install Rabbitmq Cluster and validate

       ```
       # kubectl apply -k /opt/genestack/base-kustomize/rabbitmq-operator

       # kubectl apply -k /opt/genestack/base-kustomize/rabbitmq-topology-operator

       # kubectl apply -k /opt/genestack/base-kustomize/rabbitmq-cluster/base

       # kubectl --namespace openstack get rabbitmqclusters.rabbitmq.com -w
       ```


###  23. Install mariadb Operator.
    
set cluster name

       ```
       # cluster_name=`kubectl config view --minify -o jsonpath='{.clusters[0].name}'`
       ```
       ```
       # echo $cluster_name
       ```
      
       URL (https://docs.rackspacecloud.com/infrastructure-mariadb/#deploy-the-mariadb-operator) 
       NOTE:(run script on master node to deploy operator)
       

Validate operator is up before running deploy cluster.
       
       ```
       # kubectl --namespace mariadb-system get pods -w
       ```

###  24. Install MariaDB cluster.
       
       ```
       # kubectl --namespace openstack apply -k /opt/genestack/base-kustomize/mariadb-cluster/base
       ```

Verify pods are up
       ```
       # kubectl get pods -A | grep maria
       ```
Verify readiness with the following command
       ```
       # kubectl --namespace openstack get mariadbs -w
       ```


### 25. Install memached.

PULL SCRIPT AND RUN FROM URL (https://docs.rackspacecloud.com/infrastructure-memcached/#deploy-the-memcached-cluster)
      
Verify readiness with the following command.
       
       ```
       # kubectl --namespace openstack get horizontalpodautoscaler.autoscaling memcached -w
       ```

### 26. Install libvirt.
       
       Run script at following URL (https://docs.rackspacecloud.com/infrastructure-libvirt/#run-the-package-deployment)
       
       #### NOTE: IF YOU HIT THE FOLLOWING ERROR MESSAGE.
          ```
          (Error: An error occurred while checking for chart dependencies. You may need to run `helm dependency build` to fetch missing dependencies: found in Chart.yaml, but missing in charts/ directory: helm-toolkit)
          ```
       
#### WORK AROUND:
       ```
       # helm dependency list /opt/genestack/submodules/openstack-helm-infra/libvirt/
   
       # helm dependency build /opt/genestack/submodules/openstack-helm-infra/libvirt/

       # cd /opt/genestack/submodules/openstack-helm-infra/libvirt/

       # helm dependency update .

       # Then re-run install libvirt script from URL
       ```

validate functionality on your compute hosts with virsh

  
       # kubectl exec -it $(kubectl get pods -l application=libvirt -o=jsonpath='{.items[0].metadata.name}' -n openstack) -n openstack -- virsh list
       


### 27. Deploy Openstack services

Deploy Keystone
       
Run script from URL (https://docs.rackspacecloud.com/openstack-keystone/#run-the-package-deployment)

Deploy the openstack admin client pod (optional)
         ```
         # kubectl --namespace openstack apply -f /etc/genestack/manifests/utils/utils-openstack-client-admin.yaml
   
         # kubectl --namespace openstack exec -ti openstack-admin-client -- openstack user list
         ```

### 28. Deploy Glance.

Run script from URL locally (https://docs.rackspacecloud.com/openstack-glance/#run-the-package-deployment)

validate

        # kubectl --namespace openstack exec -ti openstack-admin-client -- openstack image list
        
### 29. Deploying Cinder.
        

Run script from URL (https://docs.rackspacecloud.com/openstack-cinder/#run-the-package-deployment)

        
        # kubectl --namespace openstack exec -ti openstack-admin-client -- openstack volume list
        

### 30. Install compute kit (Nova,Neutron,Placement)

Deploy placement:

Run script from URL locally (https://docs.rackspacecloud.com/openstack-compute-kit-placement/)


Deploy Nova:

Run script from URL locally (https://docs.rackspacecloud.com/openstack-compute-kit-nova/)
        

Deploy Neutron:

Run script from URL locally (https://docs.rackspacecloud.com/openstack-compute-kit-neutron/)
        

### 31. Verify Openstack environment.
        
        ```
        # kubectl exec --stdin --tty -n openstack openstack-admin-client -- /bin/bash
        ```
        ```
        root@openstack-admin-client:/# neutron agent-list
        ```

        ```
        root@openstack-admin-client:/# nova service-list
        ```
