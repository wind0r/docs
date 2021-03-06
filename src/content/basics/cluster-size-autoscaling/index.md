+++
title = "Cluster Size and Autoscaling"
description = "This article explains options you have for defining the size of a Kubernetes cluster with Giant Swarm, and automatically scaling it"
date = "2019-11-14"
weight = 120
type = "page"
categories = ["basics"]
+++

# Cluster Size and Autoscaling

Starting with release version {{% first_aws_autoscaling_version %}} for AWS, you can leverage the benefits of the [Kubernetes autoscaler](https://github.com/kubernetes/autoscaler) to define the number of worker nodes in a cluster based on demand.
The autoscaler is provided with every cluster of version >= {{% first_aws_autoscaling_version %}} on AWS.

On Giant Swarm installations on Azure, on bare-metal, and on AWS prior to version {{% first_aws_autoscaling_version %}}, the cluster size would be defined statically.

## Setting scaling limits

With the autoscaler taking control over the number of worker nodes, you only define the bounds or limits in which the cluster size can vary. This is possible both during cluster creation as well as after creation.

To enforce an exact cluster size and **effectively disable the autoscaler**, simply set the minimum and maximum worker node count to the same value. This is also the recommended way in case you want to control the number of worker nodes through some external tooling via the Giant Swarm API.

## How the autoscaler works

When relying on the autoscaler to determine the number of worker nodes in your cluster, you'll benefit from a deeper understanding of how the autoscaler decides when to increase or decrease the number of worker nodes. We recommend to take a look a the [Kubernetes autoscaler FAQ](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md) for details. Here we only highlight a few important aspects.

Whenever pods cannot be scheduled due to insufficient resources in the cluster, the autoscaler adds a worker node.

To decide whether a cluster can be scaled down, the autoscaler periodically calculates the **utilization** of a node based on CPU and memory requests of all running pods, compared to the node's total capacity.

**Note:** This means that your pods MUST be configured with CPU and memory requests in order to inform the autoscaler about the actual node utilization.
Pods without CPU and memory requests won't count towards the utilization as calculated by the autoscaler and won't trigger any scaling.

The current utilization is compared to a configurable _utilization threshold_, which is set {{% autoscaler_utilization_threshold %}} by default.
If the utilization is below the threshold, the autoscaler decides to remove the node.

## Minimal and default cluster size

When creating a cluster without specifying the number of worker nodes, {{% default_cluster_size_worker_nodes %}} worker nodes will be created. On AWS starting with release version {{% first_aws_autoscaling_version %}}, when not specified, the maximum number of worker nodes is also set to {{% default_cluster_size_worker_nodes %}}.

Technically, while you may be able to create and run smaller clusters successfully, we don't encourage this due to reduced resilience.

## Minimal worker node instance type

Relying on autoscaling may have an effect on the instance type you should use for your cluster, due to the way the autoscaler decides when to remove a node, as described above.

With the services required to run a cluster, like DNS, kube-proxy, ingress and others, each node has a baseline utilization, even before you start your first workloads.
With small instance types, e. g. only 1 CPU core, the utilization of an otherwise empty node can already be above {{% autoscaler_utilization_threshold %}}.
Using such a small instance type with the default utilization threshold of {{% autoscaler_utilization_threshold %}} would result in a cluster that would never get scaled down, effectively running unused worker nodes.

For automatic down-scaling to work, utilization threshold and instance type have to fit together.
The default worker nodes instance type for AWS (`{{% default_aws_instance_type %}}`) and the default utilization threshold ({{% autoscaler_utilization_threshold %}}) are adjusted so that down-scaling should work as expected.

And similarly up-scaling is affected if you eg start a cluster with a minimum of one node. If one single node is too small for the baseline utilization of the cluster the autoscaler itself might not fit onto the node anymore and therefore the cluster won't ever get scaled.

We've changed the default instance type on AWS to `{{% default_aws_instance_type %}}` and up/down-scaling will work. You can still choose `m?.large`. But there will be at least problems with scaling up a cluster since the nodes are too small.

If you decide to run larger instance types, you may ask the Giant Swarm support team to adjust the utilization threshold of a particular cluster for you.

## Ingress controller replicas with autoscaling

With AWS and release {{% first_aws_autoscaling_version %}}, the amount of Ingress Controller (IC) replicas is fixed to the minimum number of worker nodes when creating the cluster. This can mean two things:

- When scaling up a cluster from an initially low minimum number of worker nodes, there might not be enough IC replicas to fullfill all your requests.

- When scaling down a cluster from an initially high minimum number of workers, there may be several IC pods per worker node, using more resources than necessary.

When in doubt, please run load tests against your autoscaling cluster, in order to make sure you have the proper amount of IC replicas running.
Also feel free to contact the Giant Swarm support team to clarify any questions on this topic.

We plan to improve this behaviour in an upcoming release by scaling the IC using the Horizontal Pod Autoscaler (HPA), to better adapt the number of ICs to the load.
This would as a consequence lead to the worker node count being adapted to the demand for IC pods.
In releases without autoscaling support, the number of Ingress Controller replicas scales linearly with the number of worker nodes.

## Further restrictions

- Scaling a cluster to zero worker nodes is currently not supported.
- The number of master nodes cannot be changed as of now.

## See also

- [Cluster Autoscaler advanced configuration](/guides/advanced-cluster-autoscaler-configuration/)
- [Recommendations and Best Practices regarding cluster size](/guides/recommendations-and-best-practices/#cluster-sizing)
- [`gsctl create cluster`](/reference/gsctl/create-cluster/): Creating a cluster
- [`gsctl create cluster`](/reference/gsctl/scale-cluster/): Scaling a cluster
- [`gsctl show cluster`](/reference/gsctl/show-cluster/): Inspecting a cluster
- [API: Create cluster](/api/#operation/addCluster)
- [API: Modify cluster](/api/#operation/modifyCluster)
- [API: Get cluster details](/api/#operation/getCluster)
- [API: Get cluster status](/api/#operation/getClusterStatus)
- [Kubernetes autoscaler FAQ](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md)
