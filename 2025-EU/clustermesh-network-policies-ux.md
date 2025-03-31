# ClusterMesh and NetworkPolicies design improvements

## Problems

### 1: The same network policy don't mean the same thing in a normal vs meshed cluster

A few problematic examples:
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "to-prod-from-control-plane-nodes"
spec:
  endpointSelector:
    matchLabels:
      env: prod
  ingress:
    - fromNodes:
      - matchLabels:
          node-role.kubernetes.io/control-plane: ""
```

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: hubble-ui
  namespace: kube-system
spec:
  endpointSelector:
    matchLabels:
      io.cilium.k8s.policy.serviceaccount: hubble-ui
  egress:
  - toEntities:
      - kube-apiserver
  - toEndpoints:
      - matchLabels:
          io.cilium.k8s.policy.serviceaccount: hubble-relay
```

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: core-dns-ingress
spec:
  endpointSelector:
    matchLabels:
      io.cilium.k8s.policy.serviceaccount: coredns
      k8s:io.kubernetes.pod.namespace: kube-system
  ingress:
  - fromEndpoints:
    - {}
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
```

Any network policy part of a some third party deployment (helm chart or any manifest in general), that users would often apply as is. For instance here is one in ArgoCD:
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: argocd-redis-network-policy
spec:
  podSelector:
    matchLabels:
    app.kubernetes.io/name: argocd-redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: argocd-server
    - podSelector:
        matchLabels:
      	   app.kubernetes.io/name: argocd-repo-server
    - podSelector:
        matchLabels:
      	   app.kubernetes.io/name: argocd-application-controller
    ports:
    - protocol: TCP
      port: 6379
```

Also Entities are not really Cluster Mesh aware or at least might not necessarily do very intuitive things:
- Remote-nodes target the whole mesh
- There is a cluster entity but not clustermesh entity

### 2: You can't self select your cluster name

A deployment mechanism to a meshed cluster should be always aware of your own cluster name (if the goal is to restrict traffic to the local cluster only). For instance:
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "to-prod-from-control-plane-nodes"
spec:
  endpointSelector:
    matchLabels:
      env: prod
  ingress:
  - fromNodes:
    - matchLabels:
        node-role.kubernetes.io/control-plane: ""
        io.cilium.k8s.policy.cluster: cluster1
```

### 3: You have to always know a cluster name to allow traffic to it

Pet vs cattle problem applied to cluster and network policy. Having some kind of cluster labeling would most likely be useful similarly to namespace labels.

(Also could be useful for cluster affinity https://github.com/cilium/cilium/issues/29200)


## Possible solutions

### Cluster labels

SIG-multicluster created the About API (https://multicluster.sigs.k8s.io/concepts/about-api/) with a ClusterProperty resource that I believe would be perfect for this I believe!

We should probably make this opt-in (and not enabled if the CRD are installed) so that the user is aware that Cilium will do with some warning about label churn + linking to the doc to filter labels like we do for Nodes.

### A mode to restrict netpols to the local cluster

Antrea has a special "scope" field to do this: https://antrea.io/docs/v2.3.0/docs/multicluster/user-guide/#multi-cluster-networkpolicy (might not be super fit with the Cilium labels system though?)
- Could essentially be similar to what we do for Namespace and namespaced policies today.
- Unless that we attempt to select a specific cluster we essentially assume that we are only selecting for the local cluster somehow.

There are multiple problems/counterarguments for this:
- This is a breaking change
  - Could make this opt-in at first at least
- The MCS-API KEP says the following:
  - “Within a clusterset,namespace sameness applies and all namespaces with a given name are considered to be the same namespace”
- The above principle could potentially be applied to netpol
  - Caveat: I don't believe that sig-multicluster has currently a definitive opinion on netpols considering this:
    - “Note: this doc does not discuss NetworkPolicy, which cannot currently be used to describe a selector based policy that applies to a multi-cluster service.”

Could also make this mode conditional, for instance:
- enabled for NetworkPolicy
- Not enabled for CiliumNetworkPolicy
- If a C(C)NP could be enabled if it involves a host somehow too?

Might be even more confusing though :(...

### Find a way to target the local cluster

If we don't restrict netpols to the local cluster by default and can’t mix labels and entities, we should probably find a way to target a local cluster without knowing its name.

Adding some label to every local endpoint, for instance `io.cilium.k8s.policy.local-cluster: ""` and somehow prune that in the identity replication logic when exported to clustermesh api? For example:
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "allow-local-cluster"
spec:
  description: "Allow x-wing to contact rebel-base in local cluster"
  endpointSelector:
    matchLabels:
       name: x-wing
  egress:
  - toEndpoints:
    - matchLabels:
        name: rebel-base
        io.cilium.k8s.policy.local-cluster: ""
```

EDIT1: After some discussion it would be preferred to use some placeholder value to inject the cluster name directly into the cluster policy, for instance: ` io.cilium.k8s.policy.cluster: "__local_cluster__"` like proposed in the initial CFP
EDIT2: As we are probably going with restricting to local cluster this is not necessarily a problem (same as namespace currently essentially)

### Entities & clustermesh

- Combining matching labels and entities?
- Have a way to combine the cluster entity with match labels to select only the local cluster?
  - For mixing kube-apiserver + cluster id

EDIT: Apparently this is already possible today

More entities that make more sense in a clustermesh scenario?

EDIT: TBD what entity we should add, however probably not a broad “clustermesh” ones as it’s too broad/have the potential to shoot yourself in the foot
