---
editPost:
  URL: "https://github.com/buraksekili/buraksekili.github.io/blob/main/content/articles/eks.md"
  Text: "Edit this page on " # edit text
author: "Burak Sekili"
title: "Fundamental EKS requirements"
date: "2024-08-13"
description: "Brief summary of EKS requirments on AWS resources"
tags: ["Kubernetes", "AWS", "EKS"]
TocOpen: true
---

> This guide is a summary of AWS EKS Best practices documentation to help me to skim through
some concepts that i usually refer to. For details, please have a look to official EKS docs
mentioned on [Resources](#resources) section.

# EKS (Amazon Elastic Kubernetes Service)

It manages Kubernetes control-plane on behalf of you, ensures that each cluster has its own control plane.
So, as an end-user, you only need to handle your workloads in worker nodes. All control plane related stuff will be handled by AWS.

Control plane provisions at least two Kubernetes API Servers and three etcd instances across three
AZs in the AWS.

EKS will monitor your control-plane and if it fails or falls down, EKS handles this situations so that
you will have control planes at high availability.

## Networking

The control plane is deployed in a VPC managed by AWS. So, this VPC is not visible to customers.
And in order to deploy EKS cluster (workloads, nodes etc...), you need to deploy them into a
VPC again which is different than the one managed by AWS.

So, this means that you have two VPCs - one is managed by AWS for control plane and the other is
managed by customer for data plane (workloads).

In that case, you need to configure your VPC configuration carefully. Your VPC needs to connect
AWS's VPC where control plane is created in order to talk with API Server. Otherwise - if
you VPC cannot communicate with AWS's VPC -, your nodes cannot register Kubernetes control plane
which prevents scheduler from scheduling pods to worker nodes.

Your API Server's endpoint can be public - which is default - or private.
As a default option, EKS bootstraps an endpooint for API server.
Optionally, you can enable private access to the Kubernetes API server. This means that the communication
between the nodes and API server happens within the VPC.

> For more details, please advise to EKS docs: https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html

EKS both supports IPv4 and IPv6 but by default, it uses IPv4 and recommends having at least two subnets
created in different AZs while creating the cluster.

> `cluster subnets`: subnets specified during cluster creation.

### VPC

- Your VPC must have sufficient number of IP addresses so that resources on the cluster can be assigned
  to an IP address.
- Your VPC must support DNS hostname and resolution support to allow nodes to register to the cluster.

### Subnet

- Two subnets in different AZs - in at least two AZs.
- Subnets must have at least six IP addresses - ideally at least 16 IP addresses for EKS.
- Subnets can be public or private but the recommend way is private subnets.
- If you need an ingress from internet to your Pods, make sure that you have at least 1 public subnet
  to deploy loadbalancers. In this case, loadbalancers can be deployed into public or private subnets based
  on your use case but the nodes should be deployed in private subnets if possible.
- If you want to deploy LB into a subnet, subnet needs specific tags. If the LB needs to be private, add `kubernetes.io/role/internal-elb: 1`, otherwise `kubernetes.io/role/elb: 1`.

### Service of LoadBalancer type

To load balance the network traffic at L4, deploy kubernetes `Service` with type of `LoadBalancer`.
In AWS, EKS provisions ALB (AWS Application Load Balancer) for you.

Make sure that your VPC and Subnet requirements are met. ALB chooses one subnet from each AZ.
If you tag `kubernetes.io/cluster/my-cluster=shared` public subnet(s), it enables creating `Service` of
`LoadBalancer`

### Security Groups

SGs are used to control communication between the control plane and worker nodes.
When you provision a cluster, EKS creates a default SG and associates this SG to all nodes.
The default rules allow all traffic communication between nodes and all outbound traffic to any destination.

### Node communication 
The recommended way is having a both private and public subnets with minimum of two public subnets and two
private subnets where private subnets are in two AZs.

You can provision load balancers in public subnets which forwards traffic to the workloads (Pods) created
on private subnets.

### Cluster endpoint

As mentioned above, EKS provisions the cluster with public-only cluster endpoint. The recommended way is
to have public-private mode where Kubernetes API calls within your cluster (from control plane to worker nodes or vice versa)
uses private VPC.

### CNI

When you provision EKS cluster, Amazon Virtual Private Cloud (VPC) CNI is enabled by default


## Load Balancing

> Choose ALB if your workloads runs HTTP/HTTPS. If your workloads run TCP, use NLB.

For EKS, there are two targets for the target group of your load balancer; `instance` and `ip`.

> Target group forwards requests to the registered targets such as EC2 instance as per [docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)

### `Instance` target

This target will be the node IP of the worker node.

So, the incoming traffic will be routed to worker node through `NodePort`, 
and then the `Service` of `ClusterIP` selects a pod to forward the traffic.

This process has extra network hops and also sometimes ClusterIP service might select a Pod which
runs on different node (different AZ) which will increase the latency.

### `IP` target

Now, the traffic will be directly forwarded to the Pods. So, it significantly reduces the latency as 
the process does not include hops between NodePort and Service.

## Resources

- https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html
- https://aws.github.io/aws-eks-best-practices/