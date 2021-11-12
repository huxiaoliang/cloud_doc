# Support Managed Kubernetes Provider

## Glossary
The lexicon used in this document is described in more detail [here](https://github.com/kubernetes-sigs/cluster-api/blob/master/docs/book/src/reference/glossary.md). Any discrepancies should be rectified in the main Cluster API glossary.

- **K8SaaS** - Kubernetes as a service.
- **MKP** - Managed Kubernetes Provider: Provides a managed Kubernetes cluster where you can easily deploy, manage, and scale your container-based applications using K8SaaS.

## Summary

In current Cluster API, users can create Kubernetes cluster by a series of [providers](https://github.com/kubernetes-sigs/cluster-api/blob/master/docs/book/src/reference/providers.md#infrastructure), the providers interact with cloud infrastructure resource such as machine, load balance, security groups directly and is generally referred to as infrastructure provider. The infrastructure provider usually works by create machines on the cloud or bare metal and turning them into Kubernetes master nodes (control plane) or worker nodes, finally expose them as workload/target Kubernetes cluster to end user for using.

The most of public clouds vendor not only provide infrastructure service but also provide K8SaaS as known as MKP . For example, [AWS provider](https://github.com/kubernetes-sigs/cluster-api-provider-aws) depends on Amazon EC2 service build Kubernetes cluster, actually Amazon Elastic Kubernetes Service (Amazon EKS) can achieve this as well. This proposal outlines adding a new approach to Cluster API by leveraging MKP like EKS, IKS, GKE and AKS, etc to create and manage Kubernetes cluster directly. The operations to provision infrastructure specific resource and turning resource to Kubernetes node are provided by MKP itself, no cloud infrastructure resource interactions and bootstrap process (kubeadm by default) get involved in Cluster API anymore.

MKP is an enhancement and optional against current infrastructure provider, that means user still can use current Cluster API infrastructure provider but have another choice to manage workload/target Kubernetes cluster, since not every cloud vendor will have MKP functionality.

## Motivation

The public clouds cloud vendor have invested a significant amount of time optimizing and reliable of operation to manage Kubernetes cluster. The current Cluster API provider provisioning and turning solution has a lot of steps and interactions with infrastructure layer which may lead error-prone. Allowing users of Cluster API to leverage the optimization approach exposed by MKP could prove beneficial.

**Potential benefits include:**
- Faster cluster provisioning
- Improved provisioning success rates
- Automatic cluster operations for create/delete/upgrade/scaling if supported by cloud vendor
- Deeply integrated with cloud provider on storage, network, load balance, security and HA
- Improved user experience
   - less YAML files to maintain

### Goals

-  Enhance existing Cluster API providers for leveraging MKP to manage Kubernetes cluster.
  - Add [EKS](https://aws.amazon.com/eks/) support to [AWS provider](https://github.com/kubernetes-sigs/cluster-api-provider-aws) 
  - Add [IKS](https://www.ibm.com/cloud/container-service/) support to [IBM Cloud provider](https://github.com/kubernetes-sigs/cluster-api-provider-ibmcloud)
  - Add [AKS](https://azure.microsoft.com/en-in/services/kubernetes-service/) support to [Azure provider](https://github.com/kubernetes-sigs/cluster-api-provider-azure)
  - Add [GKE](https://cloud.google.com/kubernetes-engine/) support to [GCP provider](https://github.com/kubernetes-sigs/cluster-api-provider-gcp)
  - Add [DOKS](https://www.digitalocean.com/products/kubernetes/) support to [DigitalOcean provider](https://github.com/kubernetes-sigs/cluster-api-provider-digitalocean)
  - Add [ACK](https://www.alibabacloud.com/product/kubernetes) support to [Alibaba Cloud](https://github.com/oam-oss/cluster-api-provider-alicloud)

### Non-goals/Future Work

- To integrate with the Kubernetes cluster autoscaler.

## Proposal

This proposal introduces MKP for the purpose of delegating the management of Kubernetes cluster to existing infrastructure [provider](https://github.com/kubernetes-sigs/cluster-api/blob/master/docs/book/src/reference/providers.md#infrastructure), after that, the providers in `Goals` section will have 2 ways manage Kubernetes cluster according to different user case.

It should have 3 approaches to extend existing [providers](https://github.com/kubernetes-sigs/cluster-api/blob/master/docs/book/src/reference/providers.md#infrastructure) to have MKP functionality: 
 1.  Reuse `infrastructureRef`from `Cluster` but have new MKP implementation:  for example, [awscluster_controller.go](https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/master/controllers/awscluster_controller.go) VS `ekscluster_controller.go`
 2.  Implement MKP based controller Control Plane to fill the gap from `Non-Goals/Future Work` section in [kubeadm-based-control-plane.md](https://github.com/kubernetes-sigs/cluster-api/blob/master/docs/proposals/20191017-kubeadm-based-control-plane.md), but cover both control plane and worker load plane.
 3.  (**Preferred**)Create individual entity `managedK8SRef` to `Cluster` against `infrastructureRef`with new controller: for example, [awscluster_controller.go](https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/master/controllers/awscluster_controller.go) VS `ekscluster_controller.go`


**Note**: 
1. The EKS/AKS/IKS, etc, only export the workload nodes to user, the control plane is completely managed by itself and invisible to user, reuse `ControlPlaneRef` will get confused to user. In addition, build control plane and workload nodes is atomic operation from MKP point of veiw(one api call to public cloud), it is hard to split to control plane and workload nodes stage.
2. Add new individual entity `managedK8SRef` plug-in to `Cluster` will make the code logic clear and align with Kunernetes declarative API design style, `infrastructureRef`, `ControlPlaneRef` and `managedK8SRef` are different user case from it's name, the preferred option will has minim change from Cluster API core and easy for existing provider to implement and maintain.

### User Stories

- As a cluster operator, I would like to create a Kubernetes cluster by MKP.
- As a cluster operator, I would like to upgrade Kubernetes cluster by MKP.
- As a cluster operator, I would like to scaling up Kubernetes cluster by MKP.
- As a cluster operator, I would like to scaling down Kubernetes cluster by MKP.
- As a cluster operator, I would like to delete Kubernetes cluster by MKP.

## Implementation Details/Notes/Constraints

#### Data Model Change

```go
type ClusterSpec struct
```
- **To add**
    - **managedK8SRef**
        - Type: `*corev1.ObjectReference`
        - Description: managedK8SRef is a reference to a provider-specific resource that holds the details for provisioning Kubernates cluster by MKP.

Take [AWS provider](https://github.com/kubernetes-sigs/cluster-api-provider-aws)  as an example,  the `EKSCluster` is provider-specific MKP resource for AWS, other cloud MKP will have `IKSCluster`,  `AKSCluster`, etc accordingly.   

```go
// EKSClusterSpec defines the desired state of EKSCluster
type EKSClusterSpec struct {
	// The Region the cluster lives in.
	Region string `json:"region,omitempty"`
	// The version of the cluster.
	Version string `json:"version,omitempty"`
	// The node type of the cluster.
	NodeType string `json:"nodeType,omitempty"`
	// The number of nodes of cluster.
	Nodes int `json:"nodes,omitempty"`
	// The minimum nodes in autoscaling group
	NodesMin int `json:"nodesMin"`
	// The maximum nodes in autoscaling group
	NodesMax int `json:"nodesMax"`	
	// ControlPlaneEndpoint represents the endpoint used to communicate with the control plane.
	ControlPlaneEndpoint clusterv1.APIEndpoint `json:"controlPlaneEndpoint"`
	// AdditionalTags is an optional set of tags to add to AWS resources managed by the AWS provider, in addition to the
	// ones added by default.
	// +optional
	AdditionalTags Tags `json:"additionalTags,omitempty"`
}

// EKSClusterStatus defines the observed state of EKSCluster
type EKSClusterStatus struct {
	Ready          bool                     `json:"ready"`
}

// EKSCluster is the Schema for the eksclusters API
type EKSCluster struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`
	Spec   AWSClusterSpec   `json:"spec,omitempty"`
	Status AWSClusterStatus `json:"status,omitempty"`
}
```
#### Spec Model Change
```
kind: Cluster
apiVersion: cluster.x-k8s.io/v1alpha3
metadata:
  name: my-cluster
  namespace: default
spec:
  managedK8SRef:
    kind: EKSCluster
    apiVersion: managedk8s.cluster.x-k8s.io/v1alpha1
    name: my-ekscluster
    namespace: default
---
apiVersion: managedk8s.cluster.x-k8s.io/v1alpha1
kind: EKSCluster
metadata:
  name: my-ekscluster
  namespace: default
spec:
  region: antarctica-1
  version: 1.14
  nodeType: t3.medium
  nodes: 1
  nodesMin: 1
  nodesMax: 2
```

#### Controller collaboration

-  The generic `Cluster` controller `Reconcile` from Cluster API core will sync up with the latest`EKSCluster` status and generate `kubeconfig`
- The provider's cluster MKP controller `Reconcile` will handle cluster life cycle management: create/scaling up/scaling down/delete.

####  Cluster update/upgrade


####  Cluster  scaling up/down
