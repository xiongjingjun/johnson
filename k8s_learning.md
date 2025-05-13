# 一. Overview

## K8S Components
**Core Plane Components**：Manage the overall state of the cluster
1) **kube-apiserver**： The core component server that exposes the Kubernetes HTTP API
2) **etcd**: Consistent and highly-available key value store for all API server data
3) **kube-scheduler**: Looks for Pods not yet bound to a node, and assigns each Pod to a suitable node.
4) **kube-controller-manager**: Runs controllers to implement Kubernetes API behavior. each controller is a separate process. 
Four types of controllers: node controller, job controller, EndpointSlice controller & ServiceAccount controller
5) **cloud-controller-manager**(Optional): Integrates with underlying cloud provider(s).

**Node Components**: Run on every node, maintaining running pods and providing the Kubernetes runtime environment.
1) **kubelet**: Ensures that Pods are running, including their containers.
2) **kube-proxy**(Optional): Maintains network rules on nodes to implement Services. [a default implementation of service proxying]
3) **Container runtime**: A fundamental component that empowers Kubernetes to run containers effectively. It is responsible for managing the execution and lifecycle of containers within the Kubernetes environment.

**Addons**: Addons use Kubernetes resources (DaemonSet, Deployment, etc) to implement cluster features. 
extend the functionality of Kubernetes, including below examples.
1) **DNS**: For cluster-wide DNS resolution. Containers started by Kubernetes automatically include this DNS server in their DNS searches.
2) **Web UI (Dashboard)**: For cluster management via a web interface
3) **Container Resource Monitoring**: For collecting and storing container metrics
4) **Cluster-level Logging**: For saving container logs to a central log store

## Objects in K8S
**Each object** includes **two nested fileds**：
a. **spec**：when creating an object, providing a description of the characteristics you want the resource to have its desired state, as well as some basic information about the object (such as a name).
b. **status**：describes the current state of the object, supplied and updated by the Kubernetes system and its components.

1. The **kubectl** command-line tool supports several different ways to **create and manage Kubernetes objects**
2. **Each object** in your cluster has a **Name** that is unique for that type of resource. Every Kubernetes object also has a **UID** that is unique across your whole cluster.
3. **Labels** are key/value pairs that are attached to objects. 
Via a **label selector**, the client/user can identify a set of objects. The label selector is the core grouping primitive in Kubernetes.
4. **Namespaces** are intended for use in environments with many users spread across multiple teams, or projects. Provide a mechanism for isolating groups of resources within a single cluster.
Namespaces cannot be nested inside one another and each Kubernetes resource can only be in one namespace.
A Kubernetes namespace provides the scope for Pods, Services, and Deployments in the cluster.

**Initial namespaces**
a. **default**: Kubernetes includes this namespace so that you can start using your new cluster without first creating a namespace.
b. **kube-node-lease**: This namespace holds Lease objects associated with each node. Node leases allow the kubelet to send heartbeats so that the control plane can detect node failure.
c. **kube-public**: This namespace is readable by all clients (including those not authenticated). This namespace is mostly reserved for cluster usage, in case that some resources should be visible and readable publicly throughout the whole cluster. The public aspect of this namespace is only a convention, not a requirement.
d. **kube-system**：The namespace for objects created by the Kubernetes system.

5. Annotations
6. Field Selectors
7. Finalizer
You can **visualize** and **manage** Kubernetes objects with more tools than **kubectl** and the **dashboard**.
applications are informal and described with metadata
8. Recommended Labels
You can visualize and manage Kubernetes objects with more tools than kubectl and the dashboard. A common set of labels allows tools to work interoperably, describing objects in a common manner that all tools can understand
The **Deployment** is used to oversee the pods running the application itself.
The **Service** is used to expose the application.
9. Kubernetes API 
The Kubernetes API lets you query and manipulate the state of objects in Kubernetes. The core of Kubernetes' control plane is the API server and the HTTP API that it exposes.
Most operations can be performed through the kubectl command-line interface or other command-line tools, such as kubeadm, which in turn use the API. However, you can also access the API directly using REST calls. 
Persistence: Kubernetes stores the serialized state of objects by writing them into **etcd**

# 二. Cluster Architecture
A Kubernetes cluster consists of a control plane +  a set of worker machines, called nodes, that run containerized applications.

The control plane manages the worker nodes and the Pods in the cluster.

You need a working container runtime on each Node in your cluster, so that the kubelet can launch Pods and their containers.
The Container Runtime Interface (CRI) is the main protocol for the communication between the kubelet and Container Runtime.

## Communication between Nodes and the Control Plane
1. **Node to Control Plane**:  The API server is configured to listen for remote connections on a secure HTTPS port with one or more forms of client authentication enabled. 
a. **Nodes** should be provisioned with the public root **certificate** for the cluster such that they can connect securely to the API server along with valid client credentials. A good approach is that the client credentials provided to the kubelet are in the form of a client certificate
b. **Pods** that wish to connect to the API server can do so securely by leveraging a **service account** so that Kubernetes will automatically inject the public root certificate and a valid bearer token into the pod when it is instantiated. The kubernetes service (in default namespace) is configured with a virtual IP address that is redirected (via kube-proxy) to the HTTPS endpoint on the API server. 
2. **Control plane to node**
a. via api server to kublet [can then use the --kubelet-certificate-authority flag to provide the API server with a root certificate bundle to use to verify the kubelet's serving certificate]
b. via api server to nodes/pods/services [using plain HTTP connections and this way not currently safe]

## Leases(契约)
Distributed systems often have a need for leases, which provide a **mechanism** to lock shared resources and coordinate activity between members of a set. In Kubernetes, the lease concept is represented by Lease objects in the coordination.k8s.io API Group, which are used for system-critical capabilities such as **node heartbeats** and component-level **leader election**。
1. Node Heartbeats
uses the Lease API to communicate kubelet node heartbeats to the Kubernetes API server. For every Node, there is a Lease object with a matching name in the **kube-node-lease** namespace.
2. Leader election
Kubernetes also uses Leases to ensure **only one instance** of a component is **running** at any given time. This is used by control plane components like **kube-controller-manager** and **kube-scheduler** in HA configurations, where only one instance of the component should be actively running while the other instances are on stand-by.
3. API server identity
Each kube-apiserver uses the Lease API to publish its identity to the rest of the system. While not particularly useful on its own, this provides a mechanism for clients to discover how many instances of kube-apiserver are operating the Kubernetes control plane.

## Kubernetes Self-Healing
Self-Healing capabilities:
a. **Container-level restarts**: If a container inside a Pod fails, Kubernetes restarts it based on the **restartPolicy**. This is done by kubelet.
b. **Replica replacement**: If a Pod in a Deployment or StatefulSet fails, Kubernetes creates a replacement Pod to maintain the specified number of replicas. This is done by its according **controller**.
c. **Persistent storage recovery**
d. **Load balancing for Services**

## Container Runtime Interface
Need a working container runtime on **each Node** in the cluster, so that the **kubelet can launch Pods and their containers**.

The Kubernetes CRI defines the main gRPC **protocol** for the communication between the node components kubelet and container runtime.

## Garbage Collection
Many objects in Kubernetes link to each other through owner references. Kubernetes uses **owner references** to give the control plane, and other API clients, the opportunity to clean up related resources before deleting an object.

two ways:
a. Cascading deletion 
b. Foreground cascading deletion

# 三. Containers

### Images
Container images are usually given a name. If you don't specify a registry hostname, Kubernetes assumes that you mean the Docker public registry. You can change this behaviour by setting default image registry in container runtime configuration.

By default, kubelet pulls images serially. In other words, kubelet sends only one image pull request to the image service at a time.
If you would like to enable parallel image pulls, you can set the field serializeImagePulls to false in the kubelet configuration.
The kubelet never pulls multiple images in parallel on behalf of one Pod. 

### Runtime Class
RuntimeClass is a feature for selecting the container runtime configuration. The container runtime configuration is used to run a Pod's containers.

The RuntimeClass resource currently only has 2 significant fields: RuntimeClass name (**metadata.name**) and handler (**handler**).

2 Other fields:
**runtimeclass.scheduling** field（set constraints to ensure that Pods running with this RuntimeClass are scheduled to nodes that support it. If scheduling is not set, this RuntimeClass is assumed to be supported by all nodes.）
**runtimeclass.scheduling.nodeSelector** field (The RuntimeClass's nodeSelector is merged with the pod's nodeSelector in admission, effectively taking the intersection of the set of nodes selected by each. If there is a conflict, the pod will be rejected.)

### Container Lifecycle Hooks
There are two hooks that are exposed to Containers:
1. PostStart: executed immediately after a container is created. 

2. PreStop: called immediately before a container is terminated due to an API request or management event such as a liveness/startup probe failure, preemption, resource contention and others.

# 四. Workload
A **workload** is an application running on Kubernetes. A workload can be a single component or several that work together.

you can use workload resources that manage a set of pods on your behalf. These resources **configure controllers** that make sure the right number of the right kind of pod are running, to match the state you specified

**Built-in workload resources**:
a. **Deployment & ReplicaSet**
b. **StatefulSet**: runs a group of Pods, and maintains a sticky **identity** for each of those Pods.  
c. **DaemonSet**: defines Pods that provide facilities that are local to nodes. Every time you add a node to your cluster that matches the specification in a DaemonSet, the control plane schedules a Pod for that DaemonSet onto the new node. 
d. **Job & CronJob**
A Job creates one or more Pods and will continue to retry execution of the Pods until a specified number of them successfully terminate. 
For a fixed completion count Job, you should set .spec.completions to the number of completions needed.
For a work queue Job, you should set .spec.parallelism to the number of parallelism.
e. **ReplicationController**: Legacy API for managing workloads that can scale horizontally. Superseded by the Deployment and ReplicaSet APIs.

**ReplicationController VS ReplicaSet**
ReplicaSet is the next-generation ReplicationController that supports the new set-based label selector. It's mainly used by Deployment as a mechanism to orchestrate pod creation, deletion and updates. Note that we recommend using Deployments instead of directly using Replica Sets

**Deployment VS ReplicaSet**:
Deployment is an object which can own ReplicaSets and update them and their Pods via declarative, server-side rolling updates. While ReplicaSets can be used independently, today they're mainly used by Deployments as a mechanism to orchestrate Pod creation, deletion and updates. When you use Deployments you don't have to worry about managing the ReplicaSets that they create. Deployments own and manage their ReplicaSets. 

### Pod
A Pod is similar to **a set of containers** with shared **namespaces** and shared **filesystem volumes**.
<bar>
Pods **used in two ways**:
a. run a single container
b. run multiple containers that need to work together  
<bar>
**Workload resources for managing pods**:
Usually you don't need to create Pods directly. Instead, **create** them **using workload resources** such as **Deployment** or **Job**. If your Pods need to track state, consider the **StatefulSet** resource.
<bar>
**Pods and controllers**:
You can use **workload resources**(Deployment, StatefulSet, DaemonSet) to **create and manage multiple Pods** for you. A **controller** for the resource handles replication and rollout and automatic healing in case of Pod failure.
<bar>
**Pod Template**
Controllers for workload resources create Pods **from a pod template** and manage those Pods on your behalf.
<bar>
**Static Pods**
Static Pods are managed directly by the kubelet daemon on a specific node, while most Pods are managed by the control plane.
<br>
**Pod Lifecycle**
1. assigning a Pod to a specific node is called **binding**, and the process of selecting which node to use is called **scheduling**. Once a Pod has been scheduled and is bound to a node, Kubernetes tries to **run** that Pod on the node.
2. Pod Phase: Pending, Running, Succeeded, Failed, Unknown
3. Container Status: Waiting, Running, Terminated 
4. Kubernetes manages **container failures within Pods** using a **restartPolicy**（Always, OnFailure, and Never）defined in the Pod spec. The restartPolicy for a Pod **applies to app containers** in the Pod and to regular **init containers**. 
Having below 4 status in sequence:
Initial crash, Repeated crashes, CrashLoopBackOff, Backoff reset
5. A Pod has a **PodStatus**, which has an array of PodConditions. Kubelet manages the following **PodConditions：**
PodScheduled, PodReadyToStartContainers, ContainersReady, Initialized, Ready
6. **pod.readinessGates**: Your application can inject extra feedback or signals into PodStatus: Pod readiness. Set readinessGates in the Pod's spec to specify **a list of additional conditions** that the kubelet evaluates for Pod readiness.
7. Pod network readiness: After a Pod gets scheduled on a node, it needs to be admitted by the kubelet and to have any required storage volumes mounted. Once these phases are complete, the kubelet works with a container runtime (using Container Runtime Interface (CRI)) to set up a runtime sandbox and configure networking for the Pod. 
8. Container probes
A probe is a diagnostic performed periodically by the kubelet on a container. By either executes code within the container, or makes a network request. 
4 Ways to check:
a. **exec**: Executes a specified command inside the container. 
b. **grpc**: Performs a remote procedure call using gRPC.
c. **httpGet**: Performs an HTTP GET request against the Pod's IP address on a specified port and path.
d. **tcpSocket**: Performs a TCP check against the Pod's IP address on a specified port.
9. Probe outcome: Success, Failure, Unknown
10. Types of probe： kubelet can optionally perform and react to three kinds of probes on running containers（livenessProbe， readinessProbe，startupProbe）
<br>

**Init Containers**
1. Init containers run before app containers.
If a Pod's init container fails, the kubelet repeatedly restarts that init container until it succeeds.
2. Regular init containers **do not** support the lifecycle, livenessProbe, readinessProbe, or startupProbe fields.
3. If an init container is created with its restartPolicy set to **Always**, it will start and remain running during the **entire life** of the Pod. 
<br>

**Sidecar Containers**
Sidecar containers are the secondary containers that run along with the main application container within the same Pod. These containers are used to **enhance** or to **extend** the functionality of the primary app container by providing additional services, or functionality such as logging, monitoring, security, or data synchronization.

**Ephemeral Containers**
1. Runs temporarily in an **existing** Pod to accomplish user-initiated actions such as troubleshooting. You use ephemeral containers to **inspect services** rather than to build applications.
2. Ephemeral containers differ from other containers in that they **lack guarantees for resources or execution**, and they will never be automatically restarted.

**Pod Quality of Service Classes**
1. Kubernetes assigns every Pod a QoS class based on the resource requests and limits of its component Containers. 
2. The possible QoS classes are **Guaranteed**, **Burstable**, and **BestEffort**
3. Memory requests and limits of containers in pod are used to set specific interfaces **memory.min** and **memory.high** provided by the **memory controller**

**Downward API**
1. Two ways to expose Pod and container fields to a running container: 
a. environment variables: 
spec.serviceAccountName,spec.nodeName,status.hostIP,status.hostIPs,status.podIP,status.podIPs,
b. files in a downwardAPI volume:
【fieldRef】metadata.labels,metadata.annotations,metadata.name,metadata.namespace,metadata.uid
【resourceFieldRef】resource: limits.cpu，resource: requests.cpu，resource: limits.memory，resource: requests.memory, etc.

# 五. Services, Load Balancing, and Networking
**K8S network model**
1. **contains several pieces**:
a. **Each pod** in a cluster gets its **own unique cluster-wide IP address**. A pod has its own private network namespace which is shared by all of the containers within the pod.
b. The **pod network** (also called a cluster network) handles communication between pods. Agents on a node (such as system daemons, or kubelet) can communicate with all pods on that node.
c. The **Service API** lets you provide a stable IP address or hostname for a service implemented by one or more backend pods.
d. The **Gateway API** (or its predecessor, Ingress) allows you to make Services accessible to clients that are outside the cluster.
e. **NetworkPolicy** is a built-in Kubernetes API that allows you to control traffic between pods, or between pods and the outside world.

2. **Key functionality** done by components:
a. Pod network namespace setup by Container Runtime Interface
b. Pod network itself is managed by a pod network implementation. On Linux, use Container Networking Interface (CNI). These implementations are often called CNI plugins.
c. Kubernetes provides a default implementation of service proxying, called kube-proxy
d. NetworkPolicy is generally also implemented by the pod network implementation

**Service**
1. A Service is a method for **exposing a network application** that is running as one or more Pods in your cluster
2. A Service controller for that Service continuously scans for Pods that match its **selector**, and then makes any necessary updates to the set of EndpointSlices for the Service.
3. Services **without** Selectors, by adding **EndpointSlices** object **manually** （setting the ‘kubernetes.io/service-name’ label）
4. **Service Types**
a. **ClusterIP**(default type): Exposes the Service on a cluster-internal IP
b. **NodePort**: Exposes the Service on each Node's IP at a static port (the NodePort)
c. **LoadBalancer**: Exposes the Service externally using an external load balancer.
d. **ExternalName**
5. headless Service， means set value of '.spec.clusterIP' to "None"
6. Find a service via two ways: **environment variables** and **DNS**

**Ingress**
Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by **rules** defined on the Ingress resource.

**Ingress Controllers**
1. In order for an Ingress to work in your cluster, there must be an ingress controller running. You need to select at least one ingress controller and make sure it is set up in your cluster. 
2. Unlike other types of controllers which run as part of the kube-controller-manager binary, Ingress controllers are not started automatically with a cluster.
3. You may deploy any number of ingress controllers using ingress class within a cluster.

**Gateway API**
1. Gateway API is a family of API kinds that provide dynamic infrastructure provisioning and advanced traffic routing.
2. Design and architecture principles：Role-oriented, Portable, Expressive, Extensible
3. Three API kinds:
a. **GatewayClass**: Defines a set of gateways with common configuration and managed by a controller that implements the class.
b. **Gateway**：Defines an instance of traffic handling infrastructure, such as cloud load balancer.
c. **HTTPRoute**: Defines HTTP-specific rules for mapping traffic from a Gateway listener to a representation of backend network endpoints. These endpoints are often represented as a Service.
4. Request flow as: client (sends a http request) --> Gateway --> HttpRoute (according to Routing rule) --> Service --> Pods 
5. Gateway API is the successor to the Ingress API. 

**EndpointSlices**
1. An **EndpointSlice** contains references to **a set of network endpoints**. The control plane automatically creates EndpointSlices for any Kubernetes Service that has a selector specified. These EndpointSlices include references to all the Pods that match the Service selector. EndpointSlices group network endpoints together by unique combinations of IP family, protocol, port number, and Service name.
2. By default, the control plane creates and manages **EndpointSlices** to have no more than **100 endpoints each**. You can configure this with the --max-endpoints-per-slice kube-controller-manager flag, up to a maximum of 1000.

# 六. Storage 

1. **Volume Category**

Ephemeral volume types have a lifetime linked to a specific Pod, but persistent volumes exist beyond the lifetime of any individual pod

To use a volume, specify the volumes to provide for the Pod in **.spec.volumes** and declare where to mount those volumes into containers in .spec.containers[*].volumeMounts

When a pod is launched, a process in the container sees a filesystem view composed from the initial contents of the container image, plus volumes (if defined) mounted inside the container.

A **ConfigMap** provides a way to inject **configuration data** into pods. 

A **downwardAPI** volume makes downward API data available to applications. Within the volume, you can find the **exposed data** as read-only files in plain text format.

**emptyDir**
a. For a Pod that defines an emptyDir volume, the volume is created **when the Pod is assigned to a node**. As the name says, the emptyDir volume is **initially empty**. All containers in the Pod can read and write the same files in the emptyDir volume, though that volume can be mounted at the same or different paths in each container. When a Pod is removed from a node for any reason, the data in the emptyDir is **deleted** permanently.
b. The **emptyDir.medium field** controls where emptyDir volumes are stored. 
c. A **size limit** can be specified for the default medium

**hostPath**
a. A hostPath volume mounts a file or directory from the host node's filesystem into your Pod.
b. hostPath volume types： Empty string (default)、DirectoryOrCreate、Directory、FileOrCreate、File、Socket、CharDevice、BlockDevice

**image**：An image volume source represents an OCI object (a container image or artifact) which is available on the kubelet's host machine

**local**
a. A local volume represents a mounted local storage device such as a disk, partition or directory.
b. Local volumes can only be used as a statically created PersistentVolume. Dynamic provisioning is not supported.
c. Compared to hostPath volumes, local volumes are used in a durable and portable manner without manually scheduling pods to nodes. local volumes are subject to the availability of the underlying node and are not suitable for all applications. 

**nfs**
An nfs volume allows an existing NFS share to be mounted into a Pod. 
When a Pod is removed, the contents of an nfs volume are preserved and the volume is **merely unmounted**.

**persistentVolumeClaim** 
A persistentVolumeClaim volume is used to **mount a PersistentVolume into a Pod**. 

**secret**
A secret volume is used to pass sensitive information, such as passwords, to Pods. secret volumes are backed by tmpfs (a RAM-backed filesystem) so they are never written to non-volatile storage.

2. **PV VS PVC**
A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. It is a **resource** in the cluster just like a node is a cluster resource. This API object captures the details of the implementation of the storage, be that NFS, iSCSI, or a cloud-provider-specific storage system.

A PersistentVolumeClaim (PVC) is a request for storage by a user. It is similar to a Pod. Pods consume node resources and **PVCs consume PV resources**. Pods can request specific levels of resources (CPU and Memory). Claims can request specific size and access modes. 

PVs are resources in the cluster. PVCs are requests for those resources and also act as claim checks to the resource. 

3. two ways of PVs provisioned: static & Dynamic 

4. Binding
If a PV was dynamically provisioned for a new PVC, the loop will always bind that PV to the PVC. Once bound, PersistentVolumeClaim binds are exclusive, regardless of how they were bound. A PVC to PV binding is a **one-to-one mapping**, using a **ClaimRef** which is a bi-directional binding between the PersistentVolume and the PersistentVolumeClaim.

5. Reserving a PersistentVolume:  Specify the relevant PersistentVolumeClaim in the **claimRef field of the PV** so that other PVCs can not bind to it.

6. Expanding PVCs: You can only expand a PVC if its storage class's **allowVolumeExpansion** field is set to true.

7. Reclaim Policy
a. Retain -- manual reclaimation
b. Recycle -- basic scrub (rm -rf /thevolume/*)
c. Delete -- delete the volume

8. Node Affinity: A PV can specify node affinity to define constraints that limit what nodes this volume can be accessed from.

9. Cross namespace data sources: To use cross namespace volume data sources, you must enable the AnyVolumeDataSource and CrossNamespaceVolumeDataSource feature gates for the kube-apiserver and kube-controller-manager. Also, you must enable the CrossNamespaceVolumeDataSource feature gate for the csi-provisioner.

10. Projected Volumes: A projected volume maps several existing volume sources into the same directory. Currently, these types of volumn sources could be projected: secret, downwardAPI, configMap, serviceAccountToken, clusterTrustBundle

11. Types of ephemeral volumes supported: emptyDir, configMap, downwardAPI, secret, image, CSI ephemeral volumes, generic ephemeral volumes

12. **Volume Attributes Classes**: Each VolumeAttributesClass contains the driverName and parameters, which are used when a PersistentVolume (PV) belonging to the class needs to be dynamically provisioned or modified.

13. **Dynamic Volume Provisioning**
The implementation of dynamic volume provisioning is based on the API object **StorageClass** from the API group **storage.k8s.io**.

14. **VolumeSnapshotContent & VolumeSnapshot**
A VolumeSnapshotContent is a snapshot taken from a volume in the cluster that has been provisioned by an administrator. It is a **resource** in the cluster just like a PersistentVolume is a cluster resource.
A VolumeSnapshot is a **request** for snapshot of a volume by a user. It is similar to a PersistentVolumeClaim.

15. **CSI Volume Cloning**
The CSI Volume Cloning feature adds support for specifying existing PVCs in the **dataSource** field to indicate a user would like to clone a Volume. The source PVC must be bound and available (not in use)

# 七. Configuration
1. **Configuration Best Practices**
a. Naked Pods will not be rescheduled in the event of a node failure.
b. Create a Service before its corresponding backend workloads (Deployments or ReplicaSets). Any Service that a Pod wants to access must be created before the Pod itself, or else the environment variables will not be populated. DNS does not have this restriction.
c. Don't specify a hostPort for a Pod unless it is absolutely necessary. Kubernetes will use 0.0.0.0 as the default hostIP and TCP as the default protocol.
If you only need access to the port for debugging purposes, you can use the apiserver proxy or kubectl port-forward.
d. Use headless Services (which have a ClusterIP of None) for service discovery when you don't need kube-proxy load balancing.

2. ConfigMaps & Secrets

3. Resource Management for Pods and Containers
a. When you specify the resource request for containers in a Pod, the **kube-scheduler** uses this information to decide which node to place the Pod on. 
When you specify a resource limit for a container, the **kubelet** enforces those limits so that the running container is not allowed to use more of that resource than the limit you set. 
**Limits** are a different story. Both cpu and memory limits are applied by the **kubelet (and container runtime)**, and are ultimately enforced by the **kernel**.
b. Resource requests and limits of container
spec.containers[].resources.limits/requests.cpu/memory
c. Resource requests and limits of pod
spec.resources.limits/requests.cpu/memory
d. Configure **resource quotas **to limit the total amount of resources that a **namespace** can consume. Setting resource quotas helps to prevent one team from using so much of any resource that this over-use affects other teams.

4. Organizing Cluster Access Using kubeconfig Files
a. With **kubeconfig** files, you can organize your clusters, users, and namespaces. You can also define **contexts** to quickly and easily switch between clusters and namespaces.
b. A context element in a kubeconfig file is used to group access parameters under a convenient name. Each context has three parameters: **cluster, namespace, and user**. 
c. if KUBECONFIG environment variable not set, kubectl uses the **default** kubeconfig file, **$HOME/.kube/config**

# 八. Security - Service Account
1. **Definition**: A service account is a type of **non-human account** that, in Kubernetes, provides a distinct **identity** in a Kubernetes cluster. **Application Pods, system components, and entities inside and outside the cluster** can use a specific ServiceAccount's credentials to identify as that ServiceAccount
2. **Properties**: Namespaced, Lightweight, Portable
3. By default, user accounts don't exist in the Kubernetes API server; instead, the API server treats user identities as opaque data
4. Default Service Account: When you create a cluster, Kubernetes automatically creates a ServiceAccount object named **default** for **every namespace** in your cluster. 
5. Use Cases:
a. Pods need to communicate with the Kubernetes API server. E.g. read-only access to secrets. crossing namespace access. 
b. Pods need to communicate with an external service.
iii) Authenticating to a private image registry using an imagePullSecret.
c. An external service needs to communicate with the Kubernetes API server. E.g. authenticating to the cluster as part of a CI/CD pipeline.
d. Use third-party security software in your cluster that relies on the ServiceAccount identity of different Pods to group those Pods into different contexts.
6. Assign a ServiceAccount to a Pod: Set the **spec.serviceAccountName** field in the Pod specification. Kubernetes then automatically provides the credentials for that ServiceAccount to the Pod. 

# 九. Policies: Kubernetes policies are configurations that manage other configurations or runtime behaviors via **various forms** below
a. Apply policies using **API objects**: **NetworkPolicies**(restrict ingress and egress traffic for a workload), **LimitRanges**(resource allocation constraints across different object kinds, e.g. pod or pvc), **ResourceQuotas**( limit resource consumption for a namespace)
b. Apply policies using **admission controllers**
c. Apply policies using **ValidatingAdmissionPolicy**: A ValidatingAdmissionPolicy operates on an API request and can be used to block, audit, and warn users about non-compliant configurations.
d. Apply policies using dynamic admission control
e. Apply policies using Kubelet configurations: Process ID limits and reservations(limit and reserve allocatable PIDs), Node Resource Managers(manage compute, memory, and device resources for latency-critical and high-throughput workloads)

1. **LimitRange**：A LimitRange is a policy to **constrain** the resource allocations (limits and requests) that you can specify for each applicable **object kind** (such as Pod/Container or PersistentVolumeClaim) in a **namespace**.

2. **ResourceQuota** (in a namespace)
a. compute resource quota: limits/requests.cpu/memory
   storage resource quota: requests.storage, persistentvolumeclaims, etc.
   object count quota: count/**
b.** Quota Scopes**: Each quota can have an associated set of scopes. A quota will only measure usage for a resource if it matches the intersection of enumerated scopes. 
Scopes: Terminating, Terminating, BestEffort（only on pods）, NotBestEffort, PriorityClass, CrossNamespacePodAffinity, VolumeAttributesClass
**scopeSelector** with sub fields: **operator, scopeName, values** 
c. Cross-namespace Pod Affinity Quota：to limit which namespaces are allowed to have pods with affinity terms that cross namespaces. Specifically, it controls which pods are allowed to set **namespaces** or **namespaceSelector** fields in pod affinity terms
d. Resource Quota Per VolumeAttributesClass: PersistentVolumeClaims can be created with a specific volume attributes class, and might be modified after creation. 
When the quota is scoped for the volume attributes class using the scopeSelector field, the quota object is restricted to track only the following resources: persistentvolumeclaims or requests.storage
e. Quota & Cluster Capacity: ResourceQuotas are **independent** of the cluster capacity.
f. Limit PriorityClass consumption by default: It may be desired that pods at a particular priority, should be allowed in a namespace, if and only if, a matching quota object exists.

3. PID Limits And Reservations: **Node** or **Pod** PID limits

4. **Node Resource Managers**

# 十. Scheduling, Preemption and Eviction
1. kube-scheduler: the default scheduler for Kubernetes. 
kube-scheduler selects a node for the pod in a 2-step operation: **Filtering** and then **Scoring**
2. Assigning Pods to Nodes
a. Node labels
b. Node isolation/restriction: use node labels & NodeRestriction admission plugin to prevent kubelet from modifying 
c. nodeSelector: add the nodeSelector field to your Pod specification and specify the node labels you want the target node to have.
d. Affinity and anti-affinity: nodeSelector is the simplest way to constrain Pods to nodes with specific labels. Affinity and anti-affinity **expands** the types of constraints you can define.
- Node/Pod affinity 
- Node/Pod affinity weight: specify a **weight** between 1 and 100 for each instance of the preferredDuringSchedulingIgnoredDuringExecution affinity type.
- Node/Pod affinity per scheduling profile: associate a profile with a node affinity, which is useful if a profile only applies to a specific set of nodes.
- nodeName: nodeName is a field in the Pod spec. It is a more **direct** form of node selection than affinity or nodeSelector
- Pod topology spread constraints
3. Pod Overhead: In Kubernetes, Pod Overhead is a way to account for the resources consumed by the Pod infrastructure on top of the container requests & limits. It is set at admission time according to the overhead associated with the Pod's **RuntimeClass**, which defines the **overhead** field.
4. Configuring Pod **schedulingGates**: The schedulingGates field contains a list of strings, and each string literal is perceived as a criteria that Pod should be **satisfied** before considered schedulable. 
5. **Dynamic Resource Allocation**: Dynamic resource allocation is an **API for requesting and sharing resources** between pods and containers inside a pod.
a. Cluster-level API support for dynamic resource allocation under below conditions:
- use K8s V1.33
- enabled explicitly
- install a resource driver for specific resources that are meant to be managed using this API
b. resource.k8s.io/v1beta1 and resource.k8s.io/v1beta2 API groups provide these types: ResourceClaim, ResourceClaimTemplate, DeviceClass, ResourceSlice, DeviceTaintRule
6. **Scheduler Performance Tuning**
a. kube-scheduler is the Kubernetes default scheduler. It is responsible for placement of Pods on Nodes in a cluster.
b. Nodes in a cluster that meet the scheduling requirements of a Pod are called feasible Nodes for the Pod. The scheduler finds feasible Nodes for a Pod and then runs a set of functions to score the feasible Nodes, picking a Node with the highest score among the feasible ones to run the Pod. The scheduler then notifies the API server about this decision in a process called **Binding**.
7. **Pod Priority and Preemption**
a. **how to use?**
- Add one or more PriorityClasses.
- Create Pods with priorityClassName set to one of the added PriorityClasses.
b. **PriorityClass**: is a non-namespaced object that defines a mapping from a priority class name to the integer value of the priority. The higher the value, the higher the priority.
c. **Pod priority**: After you have one or more PriorityClasses, you can create Pods that specify one of those PriorityClass names in their specifications. The priority admission controller uses the **priorityClassName** field and populates the integer value of the priority. If the priority class is not found, the Pod is rejected.
8. **Node-pressure Eviction**: is the process by which the **kubelet** proactively terminates pods to reclaim resources on nodes. 
The kubelet monitors resources like memory, disk space, and filesystem inodes on your cluster's nodes. When one or more of these resources reach specific consumption levels, the kubelet can proactively fail one or more pods on the node to reclaim resources and prevent starvation.
9. **API-initiated Eviction**：is the process by which you use the **Eviction API** to create an Eviction object that triggers graceful pod termination.
You can request eviction by calling the Eviction API directly, or programmatically using a client of the API server, like the kubectl drain command. This creates an Eviction object, which causes the API server to terminate the Pod.

# 十一. Cluster Administration
##### 1. Cluster Networking
a. Kubernetes is all about sharing machines among applications. Typically, sharing machines requires ensuring that two applications do not try to use the same ports. 
b. Kubernetes clusters require to allocate non-overlapping IP addresses for Pods, Services and Nodes, from a range of available addresses configured in the following components:
- The network plugin is configured to assign IP addresses to Pods.
- The kube-apiserver is configured to assign IP addresses to Services.
- The kubelet or the cloud-controller-manager is configured to assign IP addresses to Nodes.
##### 2. Metrics For Kubernetes System Components
Kubernetes components emit metrics in **Prometheus** format.
In most cases metrics are available on /metrics endpoint of the HTTP server.
##### 3. Proxies in Kubernetes
several different proxies when using:
a. The **kubectl proxy**
- runs on a user's desktop or in a pod
- proxies from a localhost address to the Kubernetes apiserver
- client to proxy uses HTTP
- proxy to apiserver uses HTTPS
- locates apiserver
- adds authentication headers
b. The **apiserver proxy**
- is a bastion built into the apiserver
- connects a user outside of the cluster to cluster IPs which otherwise might not be reachable
- runs in the apiserver processes
- client to proxy uses HTTPS (or http if apiserver so configured)
- proxy to target may use HTTP or HTTPS as chosen by proxy using available information
- can be used to reach a Node, Pod, or Service
- does load balancing when used to reach a Service
c. The **kube proxy**
- runs on each node
- proxies UDP, TCP and SCTP
- does not understand HTTP
- provides load balancing
- is only used to reach services
d. **A Proxy/Load-balancer in front of apiserver(s)**
existence and implementation varies from cluster to cluster (e.g. nginx)
- sits between all clients and one or more apiservers
- acts as load balancer if there are several apiservers.
e. **Cloud Load Balancers on external services**
- are provided by some cloud providers (e.g. AWS ELB, Google Cloud Load Balancer)
- are created automatically when the Kubernetes service has type LoadBalancer
- usually supports UDP/TCP only
- SCTP support is up to the load balancer implementation of the cloud provider
- implementation varies by cloud provider.

# 十二. Others

### Accessing Kubernetes API from a Pod
1. Accessing the API from within a Pod
When accessing the API from **within a Pod**, locating and authenticating to the API server are slightly different to the external client case. The easiest way to use the Kubernetes API from a Pod is to use one of the **official client libraries**. 

2. Directly accessing the REST API 
While running in a Pod, your container can create an HTTPS URL for the Kubernetes API server by fetching the KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT_HTTPS environment variables. The API server's in-cluster address is also published to a Service named kubernetes in the default namespace so that pods may reference kubernetes.default.svc as a DNS name for the local API server.

3. Using kubectl proxy
If you would like to query the API without an official client library, you can run kubectl proxy as the command of a new sidecar container in the Pod. This way, kubectl proxy will authenticate to the API and expose it on the localhost interface of the Pod, so that other containers in the Pod can use it directly.

**Namespace:**
Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called namespaces. They let you partition resources into logically named groups.

use the **kubeadm** tool to create and manage Kubernetes clusters.

When you call kubeadm init, the **kubelet configuration** is marshalled to disk at **/var/lib/kubelet**


### Best Practice
1. Large Clusters:
**1.1** No more than 110 pods per node
No more than 5,000 nodes
No more than 150,000 total pods
No more than 300,000 total containers
**1.2** multiple control plane instances for high availbility
**1.3** etcd
a. start and configure additional etcd instance
b. configure the API server to use it for storing events
**1.4** Resource limit
2. Running in Mutiple Zones
node label selecting accross different zones

### K8S Commands Reference
##### Node Status
> kubectl describe node {nodeName}
> kubectl get namespaces
> kubectl get nodes
> kubectl get pods
> 
> kubectl get services
> kubectl get deployments

> kubectl get quota {quotaName} --namespace={namespace}

> kubectl describe node {nodeName}
> kubectl describe pod {podName}
> kubectl describe job {jobName}

> kubectl logs {podName}

> kubectl cluster-info

> kubectl config view

> kubectl version

> kubectl get pvc 
> kubectl get pv

> kubectl get serviceaccounts
> kubectl get endpoints

##### Run a pod
> kubectl run -it --rm --restart=Never busybox --image=gcr.io/google-containers/busybox sh 

##### Print the logs for a container in a pod or specified resource. If the pod has only one container, the container name is optional.
> kubectl logs [-f] [-p] (POD | TYPE/NAME) [-c CONTAINER] [options] 

##### Execute a command in a container
> kubectl exec (POD | TYPE/NAME) [-c CONTAINER] [flags] -- COMMAND [args...] [options]

##### Create and run a particular image in a pod
> kubectl run PODNAME --image=image [--env="key=value"] [--port=port] [--dry-run=server|client] [--overrides=inline-json]
[--command] -- [COMMAND] [args...] [options]

##### Update fields of a resource using strategic merge patch, a JSON merge patch, or a JSON patch
> kubectl patch (-f FILENAME | TYPE NAME) [-p PATCH|--patch-file FILE] [options]

##### Expose a resource as a new Kubernetes service. 
Possible resources include (case insensitive):
pod (po), service (svc), replicationcontroller (rc), deployment (deploy), replicaset (rs)

> kubectl expose (-f FILENAME | TYPE NAME) [--port=port] [--protocol=TCP|UDP|SCTP] [--target-port=number-or-name]
[--name=name] [--external-ip=external-ip-of-service] [--type=type] [options]

##### Debug cluster resources using interactive debugging containers.  'debug' provides automation for common debugging tasks for cluster objects identified by resource and name. Pods will be used by default if no resource is specified.
> kubectl debug (POD | TYPE[[.VERSION].GROUP]/NAME) [ -- COMMAND [args...] ] [options]

##### Modify kubeconfig files：
> kubectl config SUBCOMMAND [options]

##### Display Resource (CPU/Memory) usage：the top command allows you to see the resource consumption for nodes or pods.
> kubectl top node/pod

##### Update fields of a resource using strategic merge patch, a JSON merge patch, or a JSON patch. JSON and YAML formats are accepted.
> kubectl patch (-f FILENAME | TYPE NAME) [-p PATCH|--patch-file FILE] [options]

##### Forward one or more local resouce type/name's ports to a pod. The resouce could may contain an applicaiton. 
> kubectl port-forward TYPE/NAME [options] [LOCAL_PORT:]REMOTE_PORT [...[LOCAL_PORT_N:]REMOTE_PORT_N]

##### Print the address of the control plane and cluster services
> kubectl cluster-info

##### Manage the rollout of a resource【deployments/daemonsets/statefusets】
> kubectl rollout history/pause/restart/resume/status/undo [options]

##### Update the labels on a resource
> kubectl label [--overwrite] (-f FILENAME | TYPE NAME) KEY_1=VAL_1 ... KEY_N=VAL_N [--resource-version=version]
[options]

##### Apply a configuration to a resource by file name or stdin.
[It is suggested to maintain a set of configuration files in source control (see configuration as code), then use kubectl apply to push your configuration changes to the cluster]
> kubectl apply (-f FILENAME | -k DIRECTORY) [options]
