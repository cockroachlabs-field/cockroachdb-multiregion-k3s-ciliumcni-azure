# Kubernetes Setup

## Build Steps

The first step is to install a K3s master node making sure to disable support for the default CNI plugin and the built-in network policy enforcer.
To do this we will be using [k3sup](https://github.com/alexellis/k3sup) (said 'ketchup'). [k3sup](https://github.com/alexellis/k3sup) is a light-weight utility to get from zero to KUBECONFIG with k3s on any local or remote VM. All you need is ssh access and the k3sup binary to get kubectl access immediately.


1. The first step is to install [k3sup](https://github.com/alexellis/k3sup) on the the host workstation you will be using to configure the demo enviroment. This could be your workstation or a dedicated builder machine.

```
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
k3sup --help
```
2. Next we need to deploy [k3s](https://github.com/k3s-io/k3s) on to our first virtual machine in region one. Use the [k3sup](https://github.com/alexellis/k3sup) command below to install the first node. Be sure to replace (Public IP) with the public IP of the node.
Retrieve the Public IP address of the first node in region one
```
MASTERR1=$(az vm show -d -g $rg  -n crdb-$loc1-node1 --query publicIps -o tsv)
```
Now use [k3sup](https://github.com/alexellis/k3sup) to create the first Kubernetes cluster.
```
k3sup install \
  --ip=$MASTERR1 \
  --user=ubuntu \
  --sudo \
  --cluster \
  --k3s-channel stable \
  --k3s-extra-args '--flannel-backend none --disable-network-policy' \
  --merge \
  --local-path $HOME/.kube/config \
  --context=$clus1
```

3. Next you can add the agent nodes to the [k3s](https://github.com/k3s-io/k3s) This will be where are workloads are ran from. In this example we are going to add three agents.
Obtain the Public IP address of the second node.
```
AGENT1R1=$(az vm show -d -g $rg  -n crdb-$loc1-node2 --query publicIps -o tsv)
```
Now use [k3sup](https://github.com/alexellis/k3sup) to add the node to the existing cluster.
```
k3sup join \
  --ip $AGENT1R1 \
  --user ubuntu \
  --sudo \
  --k3s-channel stable \
  --server \
  --server-ip $MASTERR1 \
  --server-user ubuntu \
  --sudo \
  --k3s-extra-args '--flannel-backend=none --disable-network-policy'
```
Repeat this for the third node.

Obtain the Public IP address of the third node.
```
AGENT2R1=$(az vm show -d -g $rg  -n crdb-$loc1-node3 --query publicIps -o tsv)
```
Now use [k3sup](https://github.com/alexellis/k3sup) to add the node to the existing cluster.
```
k3sup join \
  --ip $AGENT2R1 \
  --user ubuntu \
  --sudo \
  --k3s-channel stable \
  --server \
  --server-ip $MASTERR1 \
  --server-user ubuntu \
  --sudo \
  --k3s-extra-args '--flannel-backend=none --disable-network-policy'
```

4. So now we have a single cluster in one region we can now move on to the deployment of [k3s](https://github.com/k3s-io/k3s) to our second region. The command below is similar to the command is step two but we need to ensure we use the Public IP of the first node in our second region. 
Deploy Second k3s cluster
Retrieve the Public IP address of the first node in region two
```
MASTERR2=$(az vm show -d -g $rg  -n crdb-$loc2-node1 --query publicIps -o tsv)
```
Now use [k3sup](https://github.com/alexellis/k3sup) to create the second Kubernetes cluster.
```
k3sup install \
  --ip=$MASTERR2 \
  --user=ubuntu \
  --sudo \
  --cluster \
  --k3s-channel=stable \
  --k3s-extra-args '--flannel-backend none --disable-network-policy' \
  --merge \
  --local-path $HOME/.kube/config \
  --context=$clus2
```
Next you can add the agent nodes to the [k3s](https://github.com/k3s-io/k3s) This will be where are workloads are ran from. In this example we are going to add two agents.
Obtain the Public IP address of the second node.
```
AGENT1R2=$(az vm show -d -g $rg  -n crdb-$loc2-node2 --query publicIps -o tsv)
```
Now use [k3sup](https://github.com/alexellis/k3sup) to add the node to the existing cluster.
```
k3sup join \
  --ip $AGENT1R2 \
  --user ubuntu \
  --sudo \
  --k3s-channel stable \
  --server \
  --server-ip $MASTERR2 \
  --server-user ubuntu \
  --sudo \
  --k3s-extra-args '--flannel-backend=none --disable-network-policy'
```
Repeat this for the third node.

Obtain the Public IP address of the third node.
```
AGENT2R2=$(az vm show -d -g $rg  -n crdb-$loc2-node3 --query publicIps -o tsv)
```
Now use [k3sup](https://github.com/alexellis/k3sup) to add the node to the existing cluster.
```
k3sup join \
  --ip $AGENT2R2 \
  --user ubuntu \
  --sudo \
  --k3s-channel stable \
  --server \
  --server-ip $MASTERR2 \
  --server-user ubuntu \
  --sudo \
  --k3s-extra-args '--flannel-backend=none --disable-network-policy'
```

5. Install the latest version of the Cilium CLI. The Cilium CLI can be used to install Cilium, inspect the state of a Cilium installation, and enable/disable various features (e.g. clustermesh, Hubble).
```
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
```

6. Now that we have installed the Cilium CLI we can use it to install the Cilium CNI on to our first [k3s](https://github.com/k3s-io/k3s) cluster. First we ensure we are using the correct context.  
```
kubectl config use-context $clus1
cilium install --config cluster-pool-ipv4-cidr=10.10.0.0/16 --cluster-name=$clus1 --cluster-id=1
```
7. We need to no repeat this process for our second [k3s](https://github.com/k3s-io/k3s) cluster. If you are planning to run Hubble Relay across clusters, it is best to share a certificate authority (CA) between the clusters as it will enable mTLS across clusters to just work. We can do this by simply propagate the Kubernetes secret containing the CA from one cluster to the other.

```
kubectl config use-context $clus1
kubectl get secret cilium-ca -n kube-system -o yaml > cilium-ca.yaml
kubectl config use-context $clus2
kubectl create -f cilium-ca.yaml
```

8. Perform the install of the Cilium CNI on the second k3s](https://github.com/k3s-io/k3s) cluster.

>Note: Pay attention here to the different cidr that has been used form the first cluster. Cluster must not have overlappping address space for the Pod network

```
cilium install --config cluster-pool-ipv4-cidr=10.11.0.0/16 --cluster-name=$clus2 --cluster-id=2 
```

9. Cilium Cluster Mesh is a way to build a mesh of Kubernetes clusters. By connecting clusters together you enable pod-to-pod connectivity across all clusters, define global services to load-balance between clusters and enforce security policies to restrict access. To enable this feature run the two commands below for your host workstation.

```
cilium clustermesh enable --context $clus1 --service-type LoadBalancer
cilium clustermesh enable --context $clus2 --service-type LoadBalancer
```
You are able to check the progress by using the commands below.

```
cilium clustermesh status --context $clus1 --wait
cilium clustermesh status --context $clus2 --wait
```

10. Once the mesh is established you can connect the two clusters together.

```
cilium clustermesh connect --context $clus1 --destination-context $clus2
```
Now you are ready to move to the next step. [Region Three Virtual Machine setup.](vm-setup.md)

[Back](README.md)
