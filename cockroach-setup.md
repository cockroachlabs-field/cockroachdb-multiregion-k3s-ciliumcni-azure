# Deploy CockroachDB to Kubernetes Clusters

- Download and Configure the Scripts to deploy cockroachdb
 1. Create a directory and download the required script and configuration files into it:   

```bash
mkdir multiregion
```

```bash
cd multiregion
```

```bash
curl -OOOOOOOOO \
https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/{README.md,client-secure.yaml,cluster-init-secure.yaml,cockroachdb-statefulset-secure.yaml,dns-lb.yaml,example-app-secure.yaml,external-name-svc.yaml,setup.py,teardown.py}
```

2. Retrieve the `kubectl` "contexts" for your clusters: 

```bash
kubectl config get-contexts
```

At the top of the `setup.py` script, fill in the `contexts` map with the zones of your clusters and their "context" names, e.g.:

> `contexts = { 'eastus': 'crdb-k3s-eastus', 'westus': 'crdb-k3s-westus', 'northeurpoe': 'crdb-k3s-northeurope',}`

3. In the `setup.py` script, fill in the `regions` map with the zones and corresponding regions of your clusters, for example:

> `regions = { 'eastus': 'eastus', 'westus': 'westus', 'northeurope': 'northeurope',}`

Setting `regions` is optional, but recommended, because it improves CockroachDB's ability to diversify data placement if you use more than one zone in the same region. If you aren't specifying regions, just leave the map empty.

4. If you haven't already, [install CockroachDB locally and add it to your `PATH`](https://www.cockroachlabs.com/docs/v20.1/install-cockroachdb). The `cockroach` binary will be used to generate certificates.

If the `cockroach` binary is not on your `PATH`, in the `setup.py` script, set the `cockroach_path` variable to the path to the binary.

5. Optionally, to optimize your deployment for better performance, review [CockroachDB Performance on Kubernetes](https://www.cockroachlabs.com/docs/v20.1/kubernetes-performance) and make the desired modifications to the `cockroachdb-statefulset-secure.yaml` file.
6. Run the `setup.py` script: 

```bash
python setup.py
```

As the script creates various resources and creates and initializes the CockroachDB cluster, you'll see a lot of output, eventually ending with `job "cluster-init-secure" created`.

7. Configure Core DNS
            
Each Kubernetes cluster has a [CoreDNS](https://coredns.io/) service that responds to DNS requests for pods in its region. CoreDNS can also forward DNS requests to pods in other regions.

To enable traffic forwarding to CockroachDB pods in all 3 regions, you need to [modify the ConfigMap](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#coredns-configmap-options) for the CoreDNS Corefile in each region.

1. Download and open our ConfigMap template `[configmap.yaml](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/multiregion/eks/configmap.yaml)`: 

```bash
curl -O https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/eks/configmap.yaml
```

2. After [obtaining the IP addresses of the ingress load balancers in all 3 regions aks clusters, you can use this information to define a separate ConfigMap for each region. Each unique ConfigMap lists the forwarding addresses for the pods in the 2 other regions

For each region, modify `configmap.yaml` by replacing:

- `region2` and `region3` with the namespaces in which the CockroachDB pods will run in the other 2 regions.
- `ip1`, `ip2`, and `ip3` with the IP addresses of the Network Load Balancers in the region, which you looked up in the previous step.

You will end up with 2 different ConfigMaps. Give each ConfigMap a unique filename like `configmap-1.yaml`. An example of which can be found in this repository

3. For each region, first back up the existing ConfigMap:  

```bash
kubectl -n kube-system get configmap coredns -o yaml > <configmap-backup-name>
```

Then apply the new ConfigMap:

```bash
kubectl apply -f <configmap-name> --context <cluster-context>
```

4. For each region, check that your CoreDNS settings were applied: 

```bash
kubectl get -n kube-system cm/coredns --export -o yaml --context <cluster-context>
```

8. Confirm that the CockroachDB pods in each cluster say `1/1` in the `READY` column - This could take a couple of minutes to propagate, indicating that they've successfully joined the cluster:    

```bash
kubectl get pods --selector app=cockroachdb --all-namespaces --context $clus1
```

> `NAMESPACE NAME READY STATUS RESTARTS AGE
us-east cockroachdb-0 1/1 Running 0 14m
us-east cockroachdb-1 1/1 Running 0 14m
us-east cockroachdb-2 1/1 Running 0 14m`


```bash
kubectl get pods --selector app=cockroachdb --all-namespaces --context $clus2
```

> `NAMESPACE NAME READY STATUS RESTARTS AGE
us-west cockroachdb-0 1/1 Running 0 14m
us-west cockroachdb-1 1/1 Running 0 14m
us-west cockroachdb-2 1/1 Running 0 14m`


9. Create secure clients

```bash
kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/client-secure.yaml --namespace $loc1
```

```bash
kubectl exec -it cockroachdb-client-secure -n $loc1 -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```

10. Port forward the admin ui

```bash
kubectl port-forward cockroachdb-0 8080 --context $clus1 --namespace $loc1
```
You will then be able to access the Admin UI via your browser. http://localhost:8080

[Back](README.md)