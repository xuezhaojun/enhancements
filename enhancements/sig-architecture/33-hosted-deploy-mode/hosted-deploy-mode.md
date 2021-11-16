# Detached deploy mode

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

The proposed work would support deploying cluster-manager and klustetlet in **Detached** mode. It means when deploying cluster-manager/klusterlet to one cluster(normally without worker nodes and compute resources), all deployments are deployed in a detached way into another cluster.

## Motivation

### Motivation of detached cluster-manager 
 
We want to combine [hypershift](https://github.com/openshift/hypershift) with open-cluster-management to provide multi-cluster management and faster cluster provisioning simultaneously.
 
In hypershift, we use a powerful cluster called **management cluster** to create many hosted clusters.
 Then we want to deploy cluster-manager on these hosted clusters, but hosted clusters don't have any worker nodes.

With Detached deploy mode, we can do it successfully by running all deployments in the **management cluster**.

### Motivation of detached klusterlet
 
Combine [hypershift](https://github.com/openshift/hypershift) with open-cluster-management, we hope leaving the HostedCluster workers clean(reducing the footprint of the worker nodes), running Klusterlet outside of the managed cluster, the Detached mode, helps to achieve this.

### Goals

* Support deploy cluster-manager in Detached mode
* Support deploy klusterlet in Detached mode

### Non-Goals

* Support deploy addons in Detached mode(TBD)

## Proposal

### User Stories

#### Detached cluster-manager: Story 1

As a user, I hope to run all deployments from serval hubs on one hosted cluster to better leverage resources.

#### Detached cluster-manager: Story 2

As a user, I hope to get the same user experience when using a hub deployed in Hosted mode as deployed in Default mode.

#### Detached klusterlet: Story 1
 
As a user, I hope to manage a cluster which is not permitted to run workloads.
 
#### Detached klusterlet: Story 2
 
As a hub cluster administrator, on the premise that an untrusted cluster can be managed, I do not want the cluster to talk to the hub cluster directly.

### Architecture

#### Architecture of Detached cluster-manager

```
        +---------------------------------------------------------+                                               
        |  management-cluster                                     |                                               
        |                                                         |                                               
        |  +----------------+  +-------------+ +------------+     |                                               
        |  |deployment:     |  | deployment: | |deployment: |     |                                               
        |  |cluster-manager |  | registration| |work        | etc.|                                               
        |  |                |  |             | |            |     |                                               
        |  +------|---------+  +------|------+ +------|-----+     |                                               
        |         |                   |               |           |                                               
        +---------|-------------------|---------------|-----------+                                               
                  |                   |               |                                                           
                  |--------------|--------------------+                                                           
                                 |watch,create,update,delete...                                                   
                                 |                                                                                
        +------------------------v--------------------------------+                                               
        |  hub-cluster                                            |       registrater       +-----------------+   
        |                                                         <-------------------------> managed cluster |   
        |  +-------+ +-----------+ +---------------+              |                         +-----------------+   
        |  | CRDs  | |APIServices| |Configurations | etc.         |                                               
        |  |       | |           | |               |              |                                               
        |  +-------+ +-----------+ +---------------+              |                                               
        +---------------------------------------------------------+
```

The **hub-cluster** is where hub is installed and where the cluster agent should register.
 
The **management-cluster** is the cluster where deployments are actually running on.

Note that cluster-manager is deploy on **management-cluster** but configed with a kubeconfig secret named ”external-hub-kubeconfig” which is a cluster-admin kubeconfig of the **hub-cluster**.
 
At API level, we add a field “DeployOption” to “ClusterManagerSpec”. The detail comes with the following:

```golang
type InstallMode string

const (
	InstallModeDefault  InstallMode = "Default"
	InstallModeDetached InstallMode = "Detached"
)

type DeployOption struct {
	// Mode can be Default or Detached.
	// In Default mode, the Hub is installed as a whole and all parts of Hub are deployed in the same cluster.
	// In Detached mode, only crd and configurations are installed on one cluster(defined as hub-cluster). Controllers run in another cluster (defined as management-cluster) and connect to the hub with the kubeconfig in secret of "external-hub-kubeconfig"(a kubeconfig of hub-cluster with cluster-admin permission).
	// The purpose of Detached mode is to give it more flexibility, for example we can install a hub on a cluster with no worker nodes, meanwhile running all deployments on another more powerful cluster.
	// Do not modify the Mode field once it's applied.
	// +kubebuilder:validation:Required
	// +required
	// +kubebuilder:default=Default
	// +kubebuilder:validation:Enum=Default;Detached
	Mode InstallMode `json:"mode"`
}
```

We also need to make the `Mode` field immutable by setup a webhook server.

#### Architecture of Detached klusterlet

// TODO add Architecture of Detached klusterlet

### Graduation Criteria

Alpha:
* Detached mode is implement and works well on KinD cluster for both cluster-manager and klusterlet

### Test Plan

Alpha:
* Unit tests that makes sure the project's fundamental quality.
* Integration tests against real KinD clusters to execercise the installation.

