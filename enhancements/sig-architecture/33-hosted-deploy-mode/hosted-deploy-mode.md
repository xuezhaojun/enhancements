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

As a user, I hope to get the same user experience when using a hub deployed in Detached mode as deployed in Default mode.

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

#### Architecture of Detached klusterlet

```
    +------------------------------------------------------------------+                                   
    |  management-cluster                                              |                                   
    |                                                                  |                                   
    | +--------------+  +--------------------+  +---------------+      |                                   
    | | deployment:  |  | deployment:        |  | deployment:   |  etc.|                                   
    | | klusterlet   |  | registration-agent |  | work-agent    |      |                                   
    | +------|-------+  +--------------------+  +-------|-------+      |                                   
    |        |                    |                     |              |                                   
    +--------|--------------------|---------------------|--------------+                                   
             |                    |                     |                                                  
             |                    |                     |                                                  
             ---------------------|----------------------                                                  
                                  | watch,create,update,delete...                                          
                                  |                                                                        
    +-----------------------------v------------------------------------+                                   
    |  managed-cluster                                                 |                                   
    |                                                                  |                                   
    |  +----------------------+   +----------------------+             |               +--------------+    
    |  | CRDs:                |   | Configurations:      |             ----------------> hub cluster  |    
    |  | clusterClaim         |   | role,rolebinding,sa  |   etc.      |               +--------------+    
    |  | appliedmanifestwork  |   |                      |             |                                   
    |  +----------------------+   +----------------------+             |                                   
    +------------------------------------------------------------------+   
```

- The **managed-cluster** is the cluster managed by the hub cluster, manifestwork will be deployed onto this cluster.
- The **management-cluster** is the cluster where Klusterlet related deployments are actually running on.

In Detached mode, the "klusterlet.spec.namespace" will be **ignored**, all agent components will be installed to the namespace named `<klusterlet's name>-open-cluster-management-agent`. The other thing to note is that users are responsible for creating an `external-managed-kubeconfig` in the agent installation namespace, Klusterlet-operator will use this kubeconfig to [get a new minimum permission kubeconfig](#minimize-the-permission-of-the-external-kubeconfig) for the managed cluster, then render it to registration-agent and work-agent.

#### Minimize the permission of the external kubeconfig

In Detached mode, users are required to provide a `external-hub/managed-kubeconfig`, klusterlet-operator will use this kubeconfig to create roles, service accounts, rolebindings with the minimum permission that the registration/work needs in the hub/managed cluster, and retrieve a token from the secret that the created service account relates in the hub/managed cluster to construct the external hub/managed cluster kubeconfig for registration and work.


### Api Changes

At API level, we add a field "DeployOption" to the "ClusterManagerSpec" and "KlusterletSpec". The detail comes with the following:

```golang
// DeployOption describes the deploy options for cluster-manager or klusterlet
type DeployOption struct {
	// Mode can be Default or Detached.
	// For cluster-manager:
	//   - In Default mode, the Hub is installed as a whole and all parts of Hub are deployed in the same cluster.
	//   - In Detached mode, only crd and configurations are installed on one cluster(defined as hub-cluster). Controllers run in another cluster (defined as management-cluster) and connect to the hub with the kubeconfig in secret of "external-hub-kubeconfig"(a kubeconfig of hub-cluster with cluster-admin permission).
	// For klusterlet:
	//   - In Default mode, all klusterlet related resources are deployed on the managed cluster.
	//   - In Detached mode, only crd and configurations are installed on the spoke/managed cluster. Controllers run in another cluster (defined as management-cluster) and connect to the mangaged cluster with the kubeconfig in secret of "external-managed-kubeconfig"(a kubeconfig of managed-cluster with cluster-admin permission).
	// The purpose of Detached mode is to give it more flexibility, for example we can install a hub on a cluster with no worker nodes, meanwhile running all deployments on another more powerful cluster.
	// And we can also register a managed cluster to the hub that has some firewall rules preventing access from the managed cluster.
	//
	// Note: Do not modify the Mode field once it's applied.
	//
	// +required
	// +kubebuilder:validation:Required
	// +kubebuilder:default=Default
	// +kubebuilder:validation:Enum=Default;Detached
	Mode InstallMode `json:"mode"`
}

// InstallMode represents the mode of deploy cluster-manager or klusterlet
type InstallMode string

const (
	// InstallModeDefault is the default deploy mode.
	// The cluster-manager will be deployed in the hub-cluster, the klusterlet will be deployed in the managed-cluster.
	InstallModeDefault InstallMode = "Default"

	// InstallModeDetached means deploying components outside.
	// The cluster-manager will be deployed outside of the hub-cluster, the klusterlet will be deployed outside of the managed-cluster.
	InstallModeDetached InstallMode = "Detached"
)
```

We also need to make the `Mode` field immutable by setup a webhook server.

### Graduation Criteria

Alpha:
* Detached mode is implement and works well on KinD cluster for both cluster-manager and klusterlet

### Test Plan

Alpha:
* Unit tests that makes sure the project's fundamental quality.
* Integration tests against real KinD clusters to execercise the installation.

