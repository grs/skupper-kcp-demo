# skupper-kcp-demo

Non-transparent multi-cluster with skupper and kcp

https://github.com/grs/skupper-kcp-demo/blob/main/skupper-kcp-demo.mp4?raw=true

## Using Skupper to communicate between workloads placed in different locations with KCP

1. Configure clusters to run workloads on. These must be able to
   connect to the KCP server. At least one must have a working
   loadbalancer configuration.

   The example commands below assume two clusters named cluster-1 and
   cluster-2, with kubeconfigs named cluster-1.kubeconfig and
   cluster-2.kubeconfig respectively.

2. Install site-controller-v2 on each of these clusters, along with
   the required CRDs for Site, IngressBinding and EgressBinding.

   E.g.

```
kubectl --kubeconfig cluster-1.kubeconfig apply -f skupper-crds.yaml
kubectl --kubeconfig cluster-1.kubeconfig apply -f skupper-site-controller.yaml
```

   and

```
kubectl --kubeconfig cluster-2.kubeconfig apply -f skupper-crds.yaml
kubectl --kubeconfig cluster-2.kubeconfig apply -f skupper-site-controller.yaml
```

3. Start KCP server (the following example assume a kubeconfig for KCP
   named .kcp/admin.kubeconfig).

```
kcp start
```

4. Create workspace in KCP.

```
KUBECONFIG=.kcp/admin.kubeconfig kubectl workspaces create skupper-demo --enter
```

5. Set up syncers for each cluster to that workspace.

```
KUBECONFIG=.kcp/admin.kubeconfig kubectl kcp workload sync cluster-1 --syncer-image ghcr.io/kcp-dev/kcp/syncer:release-0.7 --resources=services,sites.skupper.io,requiredservices.skupper.io,providedservices.skupper.io -o - | kubectl --kubeconfig ./cluster-1.kubeconfig apply -f -
```

and

```
KUBECONFIG=.kcp/admin.kubeconfig kubectl kcp workload sync cluster-2 --syncer-image ghcr.io/kcp-dev/kcp/syncer:release-0.7 --resources=services,sites.skupper.io,requiredservices.skupper.io,providedservices.skupper.io -o - | kubectl --kubeconfig ./cluster-2.kubeconfig apply -f -
```

6. Set up locations and placements in KCP workspace.

   a. label the synctarget CRs, e.g.

```
kubectl --kubeconfig ./.kcp/admin.kubeconfig  label synctarget/cluster-1 category=one
kubectl --kubeconfig ./.kcp/admin.kubeconfig  label synctarget/cluster-2 category=two
```

   b. create new locations and placements, e.g.

```
kubectl --kubeconfig ./.kcp/admin.kubeconfig apply -f locations.yaml
kubectl --kubeconfig ./.kcp/admin.kubeconfig apply -f placements.yaml
```
   c. delete the default location and placement, e.g.

```
kubectl --kubeconfig ./.kcp/admin.kubeconfig delete location default
kubectl --kubeconfig ./.kcp/admin.kubeconfig delete placement default
```

7. Install network-controller in KCP workspace.

   E.g.
```
kubectl --kubeconfig ./.kcp/admin.kubeconfig apply -f skupper-network-controller.yaml
```

8. Deploy application in KCP across namespaces that are labelled for
   particular locations.

   E.g. to split the bookinfo demo between two namespaces, each placed
   on a different cluster, the details and reviews services on
   cluster-1, the productpage and ratings service on cluster-2:

```
kubectl --kubeconfig ./.kcp/admin.kubeconfig apply -f namespaces.yaml
kubectl --kubeconfig ./.kcp/admin.kubeconfig apply -f bookinfo_one.yaml -n one
kubectl --kubeconfig ./.kcp/admin.kubeconfig apply -f bookinfo_two.yaml -n two
```

9. Create RequiredService and ProvidedService resources as needed to
   expose services from one location to another.

   E.g. continuing with the bookinfo example, ensure that the ratings
   service in cluster-2 can be seen by the reviews service on
   cluster-1, and that the details and reviews service on cluster-1
   can be seen by the productpage service on cluster-2:

```
kubectl --kubeconfig ./.kcp/admin.kubeconfig apply -f bookinfo_one_skupper.yaml -n one
kubectl --kubeconfig ./.kcp/admin.kubeconfig apply -f bookinfo_two_skupper.yaml -n two
```
