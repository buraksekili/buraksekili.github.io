---
editPost:
    URL: "https://github.com/buraksekili/buraksekili.github.io/blob/main/content/articles/controller-runtime-1.md"
    Text: "Edit this page on " # edit text
author: "Burak Sekili"
title: "Diving into controller-runtime | Manager"
date: "2023-11-02"
description: "Introduction to the role of controller-runtime Manager in Kubernetes Operators"
tags: [
    "Kubernetes",
    "Operator",
    "controller-runtime",
]
TocOpen: true
---

[ctrl]: https://github.com/kubernetes-sigs/controller-runtime

## Introduction

[`controller-runtime`][ctrl] package has become a fundamental tool for 
most Kubernetes controllers, simplifying the creation 
of controllers to manage resources within a Kubernetes environment efficiently. Users tend to prefer it over
[`client-go`](https://github.com/kubernetes/client-go).

The increased adoption of projects like [Kubebuilder](https://book.kubebuilder.io/) or [Operator SDK](https://sdk.operatorframework.io/) 
has facilitated the creation of Kubernetes Operator projects.
Users need to implement minimal requirements to start with Kubernetes controllers, thanks to these projects.

As a developer working on Kubernetes projects, I inevitably touch code pieces utilizing [`controller-runtime`][ctrl] 
Whenever I dive into code base, I always learn something new about the underlying mechanism of Kubernetes.

Through this blog series, I aim to share my learning regarding [`controller-runtime`][ctrl] consolidating my notes spread across various notebooks.

This article will specifically delve into the role of [`controller-runtime`][ctrl] Manager.

## What are Controllers and Operators?

[`controller-runtime`][ctrl] has emerged as the go-to package for building Kubernetes controllers. 
However, it is essential to understand what these controllers - or Kubernetes Operators - are.

In Kubernetes, controllers observe resources, such as Deployments, in a control loop to ensure the cluster resources 
conform to the desired state specified in the resource specification (e.g., YAML files) [^1].

On the other hand, according to Redhat, a Kubernetes Operator is an application-specific controller [^2]. For instance, 
the Prometheus Operator manages the lifecycle of a Prometheus instance in the cluster, including managing configurations 
and updating Kubernetes resources, such as ConfigMaps.

Roughly both are quite similar. They provide a control loop to ensure the current state meets the desired state.

## The Architecture of Controllers

Since controllers are in charge of meeting the desired state of the resources in Kubernetes, they somehow need to
be informed about the changes on the resources and perform certain operations if needed. For this, controllers follow
a special architecture to
- observe the resources,
- inform any events (updating, deleting, adding) done on the resources,
- keep a local cache to decrease the load on API Server,
- keep a work queue to pick up events,
- run workers to perform reconciliation on resources picked up from work queue.

This architecture is clearly pictured in `client-go` documentation:

![client-go-controller-interaction](/images/controller-runtime-1-client-go-controller-interaction.jpeg#center)
<p class="reference">reference: <a href="https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md"> client-go documentation  </a> </p>

Most end-users typically do not need to interact with the sections outlined in blue in the architecture. 
The [`controller-runtime`][ctrl] effectively manages these elements. The subsequent section will explain these components
in simple terms.

To simply put, controllers use 
- cache to prevent sending each getter request to API server,
- workqueue which includes the key of the object that needs to be reconciled, 
- workers to process items reconciliation.

### Informer

Informers watch Kubernetes API server to detect changes in resources that we want to. It keeps a local cache - in-memory 
cache implementing [Store](https://pkg.go.dev/k8s.io/client-go/tools/cache#Store) interface -
including the objects observed through Kubernetes API. Then controllers and operators use this cache for all getter 
requests - GET and LIST - to prevent load on Kubernetes API server. Moreover, Informers invoke controllers by sending
objects to the controllers (registering Event Handlers).

Informers leverage certain components like Reflector, Queue and Indexer, as shown in the above diagram.

#### Reflector

According to [godocs]:
> Reflector watches a specified resource and causes all changes to be reflected in the given store.

The store is actually a cache - with two options; simple one and FIFO. Reflector pushes objects to Delta Fifo queue.

By monitoring the server (Kubernetes API Server), the Reflector maintains a local cache of the resources. 
Upon any event occurring on the watched resource, implying a new operation on the Kubernetes resource, 
the Reflector updates the cache (Delta FIFO queue, as illustrated in the diagram). Subsequently, the Informer reads 
objects from this Delta FIFO queue, indexes them for future retrievals, and dispatches the object to the controller.

#### Indexer



Indexer saves objects into thread-safe Store by indexing the objects. This approach facilitates efficient querying of 
objects from the cache. 

Custom indexers, based on specific needs, can be created. For example, a custom indexer can be generated to retrieve all 
objects based on certain fields, such as Annotations.

## Manager

According to [godocs][godocs]

> manager is required to create controllers and provides shared dependencies such as clients, caches, schemes, 
etc. Controllers must be started by calling Manager.Start.

The Manager serves as a crucial component for controllers by managing their operations. To put it simply, the manager 
oversees one or more controllers that watch the resources (e.g., Pods) of interest.

Each operator requires a Manager to operate, as the Manager controls the controllers, webhooks, metric servers, 
logs, leader elections, caches, and other components.

> For all dependencies managed by the Manager, please refer to the [Manager interface](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#)

## Controller Dependencies 

As [godocs][godocs] mentioned, Manager provides shared dependencies such as clients, caches, schemes etc.
These dependencies are shared among the controllers managed by the Manager. If you have registered two controllers with 
the Manager, these controllers will share common resources.

> Reconciliation, or the reconcile loop, involves the operators or controllers executing the business logic for the watched resources. 
For example, a Deployment controller might create a specific number of Pods as specified in the Deployment spec.

The Client package exposes functionalities to communicate with the Kubernetes API [^3]. Controllers, registered with a 
specific Manager, utilize the same client to interact with the Kubernetes API. The main operations of the client include reading and writing.

Reading operations mostly utilize the cache to access the Kubernetes API, rather than accessing it directly, 
to reduce the load on the Kubernetes API. In contrast, write operations directly communicate with the Kubernetes API. 
However, this behavior can be modified so that read requests are directed to the Kubernetes API. Nevertheless, this is 
generally not recommended unless there is a compelling reason to do so.

The cache is also shared across controllers, ensuring optimal performance. Consider a scenario where there are *n* controllers 
observing multiple resources in a cluster. If a separate cache is maintained for each controller, *n* caches will attempt 
to synchronize with the API Server, increasing the load on API Server. Instead, [`controller-runtime`][ctrl] utilizes a shared cache 
called [NewSharedIndexInformer](https://pkg.go.dev/k8s.io/client-go@v0.28.3/tools/cache#NewSharedIndexInformer) for all 
controllers registered within a manager.

![client-go-controller-interaction](/images/controller-runtime-1-controller-cache.png#center)

In the diagram above, two controllers maintain separate caches where both send `ListAndWatch` requests to API Server. 
However, [`controller-runtime`][ctrl] utilizes a shared cache, reducing the need for multiple `ListAndWatch` operations.

![client-go-controller-interaction](/images/controller-runtime-1-newsharedindexinformer.png#center)
<p class="reference">reference: <a href="https://github.com/kubernetes-sigs/controller-runtime/blob/v0.16.3/pkg/cache/internal/informers.go#L56"> controller-runtime/pkg/cache/internal/informers.go  </a> </p>

## Code

Whether you use [Kubebuilder](https://book.kubebuilder.io/), 
[Operator SDK](https://sdk.operatorframework.io/), or [`controller-runtime`][ctrl] directly, operators necessitate a Manager to function.
The [`NewManager`](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.16.3#pkg-variables) from [`controller-runtime`][ctrl] 
facilitates the creation of a new manager.


```go
var (
    // Refer to godocs for details
    // ....
    // NewManager returns a new Manager for creating Controllers.
    NewManager = manager.New	
)
```

Under the hood, [`NewManager`](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.16.3#pkg-variables) calls 
[`New`](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.16.3/pkg/manager#New) from the `manager` package.


```go
func New(config *rest.Config, options Options) (Manager, error)
```

For a simple setup, we can create a new manager as follows
```go
import (
    "log"
	
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/manager"
)

func main() {
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), manager.Options{})
    if err != nil {
        log.Fatal(err)
    }
}
```


Though this code piece is sufficient to create a Manager, the crucial part involves configuring the Manager 
using [`manager.Options{}`](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.16.3/pkg/manager#Options).

### Manager Options
[`manager.Options{}`](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.16.3/pkg/manager#Options) configures various 
dependencies, such as webhooks, clients, or leader elections under the hood.

#### Scheme

As mentioned in the godocs:
> Scheme is the scheme used to resolve runtime.Objects to GroupVersionKinds / Resources.

So, scheme helps us to register your objects Go type into GVK. If you are building operators, you will realize following
code block in your operator:

```go
package main

import (
    "k8s.io/apimachinery/pkg/runtime"
    runtimeschema "k8s.io/apimachinery/pkg/runtime/schema"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    "sigs.k8s.io/controller-runtime/pkg/scheme"
)

var (
    myscheme = runtime.NewScheme()

    // SchemeBuilder is used to add go types to the GroupVersionKind scheme 
    SchemeBuilder = &scheme.Builder{
        GroupVersion: runtimeschema.GroupVersion{
            Group:   "your.group",
            Version: "v1alpha1",
        },
    }
)

func init() {
    // Adds your GVK to the scheme that you provided, which is myscheme in our case.
    utilruntime.Must(SchemeBuilder.AddToScheme(myscheme))
}
```

The scheme is responsible for registering the Go type declaration of your Kubernetes object into a GVK. 
This is significant as `RESTMapper` then translates GVK to GVR, establishing a distinct HTTP path for your Kubernetes 
resource. Consequently, this empowers the Kubernetes client to know the relevant endpoint for your resource.


[//]: # (```go {linenos=table,hl_lines=[2],linenostart=199})

#### Cache

I mentioned cache a lot, but it is one of the most crucial piece of operators and controllers, where you can see its effect
directly.
As mentioned [Controller Dependencies](#controller-dependencies) section, [`controller-runtime`][ctrl] initializes `NewSharedIndexInformer`
for our controllers under the hood. In order to configure cache, [`cache.Options{}`](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.16.3/pkg/cache#Options)
needs to be utilized. There are again a couple of possible configurations possible but be careful while configuring
your cache since it has an impact on performance and resource consumption of your operator.

I specifically want to emphasize `SyncPeriod` and `DefaultNamespaces`

`SyncPeriod` triggers reconciliation again for every object in the cache once the duration passes. 
By default, this is configured as 10 hours or so with some jitter across all controllers. Since running a reconciliation
over all objects is quite expensive, be careful while adjusting this configuration.

`DefaultNamespaces` configures caching objects in specified namespaces. For instance, to watch objects in `prod` namespace:

```go
manager.Options{
    Cache: cache.Options{
        DefaultNamespaces: map[string]cache.Config{"prod": cache.Config{}},
    },
}
```

#### Controller

The `Controller` field, in `manager.Options{}`, configures essential options for controllers registered to this Manager. 
These options are set using [`controller.Options{}`](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.16.3/pkg/config#Controller). 

Notably, the `MaxConcurrentReconciles` attribute within this configuration governs the number of concurrent reconciles allowed. 
As detailed in the [Architecture of Controllers](#the-architecture-of-controllers) section, 
controllers run workers to execute reconciliation tasks. These workers operate as `goroutines`. 
By default, a controller uses only one goroutine, but this can be adjusted using the `MaxConcurrentReconciles` attribute.

---

After configuring the Manager's options, the [`NewManager`](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.16.3/pkg/manager/manager.go#L315) 
function generates the [`controllerManager`](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/manager/internal.go#L63) structure, 
which implements the [`Runnable`](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.16.3/pkg/manager#Runnable) interface.

During the creation of the [`controllerManager`](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/manager/internal.go#L63) 
structure, [`controller-runtime`][ctrl] initializes the [`Cluster`](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.16.3/pkg/cluster#Cluster) 
to handle all necessary operations to interact with your cluster, including managing clients and caches. 

All the settings provided within [`manager.Options{}`](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.16.3/pkg/manager#Options)
are transferred to [`cluster.New()`](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.16.3/pkg/manager/manager.go#L322) 
to create the cluster. 
This process calls the private function [`newCache(restConfig *rest.Config, opts Options) newCacheFunc`](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.16.3/pkg/cache/cache.go#L300), 
initiating the [`NewInformer`](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.16.3/pkg/cache/cache.go#L361),
which uses the type [SharedIndexInformer](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.16.3/pkg/cache/internal/informers.go#L56) 
as referenced in the [Controller Dependencies](#controller-dependencies) section.

The next step involves registering controllers to the Manager. 

```go
ctrl.NewControllerManagedBy(mgr). // 'mgr' refers to the controller-runtime Manager we've set up
    For(&your.Object{}). // The 'For()' function takes the object your controller will reconcile
    Complete(r) // 'Complete' builds the controller and starts watching.
```

I will dive into the detailed explanation of the controller registration process in my future writings to avoid making this post excessively long.

## References

[^1]: https://kubernetes.io/docs/concepts/architecture/controller/
[^2]: https://www.redhat.com/en/topics/containers/what-is-a-kubernetes-operator
[^3]: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client
[^4]: https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md
[^5]: https://github.com/kubernetes-sigs/controller-runtime/blob/v0.16.3/pkg/cache/internal/informers.go#L56

[godocs]: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager