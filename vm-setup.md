# Adding Virtual Machines in Region 3 to the Cluster

## Tell your cluster about external workloads

To allow an external workload to join your cluster, the cluster must be informed about each such workload. This is done by creating a CiliumExternalWorkload (CEW) resource for each external workload. CEW resource specifies the name and identity labels (including namespace) for the workload. The name must be the hostname of the external workload, as returned by the hostname command run in the external workload.

```
cilium clustermesh vm create crdb-$loc3-node1 -n eastus --ipv4-alloc-cidr 10.192.1.0/30
cilium clustermesh vm create crdb-$loc3-node1 -n westus --ipv4-alloc-cidr 10.191.1.0/30
cilium clustermesh vm create crdb-$loc3-node3 -n eastus --ipv4-alloc-cidr 10.190.1.0/30
```
To see the list of existing CEW resources, run:

```
cilium clustermesh vm status
```

At this point the IP: in the status for `crdb-northeurope-node1` is `N/A` to inform that the VM has not yet joined the cluster.

Run the external workload install command on your k8s cluster. This extracts the TLS certificates and other access information from the cluster installation and writes out an installation script to be used in the external workloads to install Cilium and connect it to your k8s cluster:

```
cilium clustermesh vm install install-external-workload.sh
```
Log in to the external workload. First make sure the hostname matches the name used in the CiliumExternalWorkload resource:

```
ssh ubuntu@
hostanme
curl https://releases.rancher.com/install-docker/20.10.sh | sh
sudo usermod -aG docker $USER
```

```
vim ./install-external-workload.sh
```
```
./install-external-workload.sh
```

Verify basic connectivity
Next you can check the status of the Cilium agent in your external workload:

```
cilium status
```

You should see something like:
```
KVStore:     Ok   etcd: 1/1 connected, lease-ID=7c02748328e75f57, lock lease-ID=7c02748328e75f59, has-quorum=true: https://clustermesh-apiserver.cilium.io:32379 - 3.4.13 (Leader)
Kubernetes:  Disabled
...
```







Now you are ready to move to the next step. [Pod Network test](network-test.md)

[Back](README.md)