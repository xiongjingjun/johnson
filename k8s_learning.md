# K8S Components

**Core Plane Components**：Manage the overall state of the cluster
1） **kube-apiserver**： The core component server that exposes the Kubernetes HTTP API
2） **etcd**: Consistent and highly-available key value store for all API server data
3) **kube-scheduler**: Looks for Pods not yet bound to a node, and assigns each Pod to a suitable node.
4) **kube-controller-manager**: Runs controllers to implement Kubernetes API behavior. each controller is a separate process. 
Four types of controllers: node controller, job controller, EndpointSlice controller & ServiceAccount controller
5) **cloud-controller-manager**(Optional): Integrates with underlying cloud provider(s).

**Node Components**: Run on every node, maintaining running pods and providing the Kubernetes runtime environment.
1) kubelet: Ensures that Pods are running, including their containers.
2) **kube-proxy**(Optional): Maintains network rules on nodes to implement Services.
3) **Container runtime**: A fundamental component that empowers Kubernetes to run containers effectively. It is responsible for managing the execution and lifecycle of containers within the Kubernetes environment.

**Addons**: Addons use Kubernetes resources (DaemonSet, Deployment, etc) to implement cluster features. 
extend the functionality of Kubernetes, including below examples.
1) **NDS**: For cluster-wide DNS resolution. Containers started by Kubernetes automatically include this DNS server in their DNS searches.
2) **Web UI (Dashboard)**: For cluster management via a web interface
3) **Container Resource Monitoring**: For collecting and storing container metrics
4) **Cluster-level Logging**: For saving container logs to a central log store

# 2. Objects in K8S
1. The **kubectl** command-line tool supports several different ways to **create and manage Kubernetes objects**

2. **Each object** in your cluster has a **Name** that is unique for that type of resource. Every Kubernetes object also has a **UID** that is unique across your whole cluster.

3. **Labels** are key/value pairs that are attached to objects. 
Via a **label selector**, the client/user can identify a set of objects. The label selector is the core grouping primitive in Kubernetes.

4. **Namespace**s are intended for use in environments with many users spread across multiple teams, or projects. Provide a mechanism for isolating groups of resources within a single cluster.
Namespaces cannot be nested inside one another and each Kubernetes resource can only be in one namespace.

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

# Cluster Architecture
A Kubernetes cluster consists of a control plane +  a set of worker machines, called nodes, that run containerized applications.

The control plane manages the worker nodes and the Pods in the cluster.


You need a working container runtime on each Node in your cluster, so that the kubelet can launch Pods and their containers.
The Container Runtime Interface (CRI) is the main protocol for the communication between the kubelet and Container Runtime.

# Containers

### Images
Container images are usually given a name. If you don't specify a registry hostname, Kubernetes assumes that you mean the Docker public registry. You can change this behaviour by setting default image registry in container runtime configuration.

By default, kubelet pulls images serially. In other words, kubelet sends only one image pull request to the image service at a time.
If you would like to enable parallel image pulls, you can set the field serializeImagePulls to false in the kubelet configuration.
The kubelet never pulls multiple images in parallel on behalf of one Pod. 

RuntimeClass is a feature for selecting the container runtime configuration. The container runtime configuration is used to run a Pod's containers.

A workload is an application running on Kubernetes. A workload can be a single component or several that work together.

Built-in workload resources:
a. Deployment & ReplicaSet
b. StatefulSet 
c. DaemonSet: defines Pods that provide facilities that are local to nodes. Every time you add a node to your cluster that matches the specification in a DaemonSet, the control plane schedules a Pod for that DaemonSet onto the new node. 
d. Job & CronJob


use the **kubeadm** tool to create and manage Kubernetes clusters.


1. Node Status:
> kubectl describe node {nodeName}
> kubectl get namespaces
> kubectl get nodes
> kubectl get pods
> 
> kubectl get services
> kubectl get deployments


> kubectl describe node {nodeName}
> kubectl describe pod {podName}

> kubectl logs {podName}

> kubectl cluster-info


