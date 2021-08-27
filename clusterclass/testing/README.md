# Setup

* Build controller images
* Run quickstart e2e test (kill the test before the cluster is deployed)
* Setup telepresence (incl local controllers patch)

# Test cases

Branch: `poc-clusterclass-manual-testing`

Base:

```bash
k apply -f ./base/clusterclass.yaml
k apply -f ./base/cluster.yaml
```

* Basic testcase:
  * create basic setup (cluster.yaml + clusterclass.yaml)
  * change topology fields (cluster-2.yaml)
    * control plane: add labels/annotations (to test rotation)
      * [ClusterClass: Topology metadata is not propagated to machines #5168](https://github.com/kubernetes-sigs/cluster-api/issues/5168)
        * [🐛 ClusterClass : fix propagate metadata to machines, KCP fix propagate annotations #5173](https://github.com/kubernetes-sigs/cluster-api/pull/5173)
    * control plane: scale up / scale down
  * change clusterclass fields: (clusterclass-2.yaml)
    * add workers
    * control plane & worker: add labels/annotations 
  * change topology fields (cluster-3.yaml)
    * add workers
    * workers: scale up / scale down
    * workers: change labels/annotations (to test rotation)
      * [TODO: BUG: MachineDeployment rollout is broken, because we delete templates to soon](https://vmware.slack.com/archives/C02940RMBD3/p1629989630007600)
  * change clusterclass fields:
    * change a label/annotation which is overwritten in the topology and shouldn't trigger a rotation
  * change topology fields
    * workers: delete (cluster-4.yaml)
  * delete cluster

TODO:
* Check if MD templates are deleted after MD deletion (probably not)
* Same with errors?
* ClusterClass updates:
    * ref changes
* ClusterClass rebase

# Tasks

* webhook currently forbids all changes to ClusterClass refs, not only incompatible ones
* rolloutAfter is there but isn't implemented yet

# Issues

# Findings    

* topology.controlPlane.metadata is optional but is currently not a pointer (and has no +optional tag)

* control plane machines are named like this: "my-cluster-m4c56-52fjw". I wonder if we want to put something with control-plane in the control plane name

* controlPlane machineInfrastructure must be set twice (ClusterClass + KubeadmControlPlaneTemplate)
    * Previously discussed [here](https://vmware.slack.com/archives/C02940RMBD3/p1628787368113600?thread_ts=1628786625.112900&cid=C02940RMBD3)

* Resource leak: we should probably make sure we never create duplicate resources (at least for InfrastructureCluster and ControlPLane)
    * Might be good to add an additional check (i.e. check if they already exist?) during reconcile (in addition to improving our cleanup behaviour on errors (which is already planned))

* We could block ClusterClass deletion via webhook as long as it's used in Clusters
