---
editPost:
  URL: "https://github.com/buraksekili/buraksekili.github.io/blob/main/content/articles/client-k8s-indexing.md"
  Text: "Edit this page on "
author: "Burak Sekili"
title: "Indexing in controller-runtime"
date: "2024-01-24"
description: "" # TODO: add description
tags: ["Go", "Kubernetes", "Operator", "controller-runtime"]
TocOpen: true
---

## Kubernetes Client-Side Indexing

When working with Kubernetes Operators, read requests (get and list) to the Kubernetes API server are handled by in-memory cache maintained by client-go to reduce the load on your API server. This cache can be enhanced with indexes to retrieve resources more efficiently.

This post explains the underlying indexing implementation in controller-runtime (and in client-go). We'll explore how indexing works under the hood in these Kubernetes client libraries.

If you're interested in practical examples of indexing, the Kubebuilder book and controller-runtime package documentation provide great practical examples and implementations.

### Purpose of Indexing

Indexing in Kubernetes client libraries (client-go and controller-runtime) optimizes the performance of Kubernetes controllers by enabling efficient lookups of cached objects based on specific attributes or relationships. This is crucial for:

1. Efficient Resource Retrieval: Finding resources matching certain criteria without scanning the entire cache. This is particularly useful when managing resources that depend on other resources.

2. Reduced API Server Load: Minimizing the number of API calls to the Kubernetes API server through local caching. Most modern Kubernetes Operators utilize controller-runtime, where this behavior comes by default.

3. Responsive Reconciliation: Enabling controllers to react to changes in related resources. This allows you to reconcile specific resources based on changes in their dependencies.

Let's explore these concepts through an example.

### Scenario

We'll examine a simplified version of an example from the Kubebuilder book.

> Note: We'll use controller-runtime in our examples. For general information about Kubernetes Operators and controller-runtime, please refer to my previous post.

Consider a Custom Resource Definition (CRD) called MyApp that deploys your application on Kubernetes and uses a ConfigMap for application-specific configurations.

The MyApp spec references a ConfigMap which includes MyApp specific configuration.
The MyApp controller reconciles the desired state based on the following MyApp custom resource:

```yaml
apiVersion: my.domain/v1alpha1
kind: MyApp
metadata:
  name: myapp-staging
spec:
  config:
    configMapRef:
      name: "myapp-staging-conf"
```

When a ConfigMap specified via `spec.config.configMapRef.name` is updated, MyApp should reflect those changes. To achieve this, MyApp needs to watch ConfigMap resources.

A basic initialization of the MyApp controller looks like this:

```go
ctrl.NewControllerManagedBy(mgr).
    For(&v1alpha1.MyApp{}).
    Owns(&corev1.ConfigMap{})
```

Here, the controller manages MyApp CR as the primary resource and ConfigMap as a secondary resource. The controller will reconcile on both MyApp CR and ConfigMap events (e.g., updates to either resource).

While this approach works, its efficiency is arguable since the controller will reconcile on every change to MyApp and ConfigMap resources. For example, changes to ConfigMaps unrelated to MyApp configuration will trigger reconciliation. Since reconciliation logic is often costly, involving external API calls or write requests to the Kubernetes API, we should avoid unnecessary reconciliations.

There are various ways to filter reconciliation requests, but the idiomatic approach uses Predicates and Event handlers. Since I covered these in previous blog post, we'll focus on scenarios where indexing is crucial.

Consider three MyApp instances:

```yaml
apiVersion: my.domain/v1alpha1
kind: MyApp
metadata:
  name: myapp-staging1
spec:
  config:
    configMapRef:
      name: "myapp-staging-conf"
---
apiVersion: my.domain/v1alpha1
kind: MyApp
metadata:
  name: myapp-staging2
spec:
  config:
    configMapRef:
      name: "myapp-staging-conf"
---
apiVersion: my.domain/v1alpha1
kind: MyApp
metadata:
  name: myapp-dev
spec:
  config:
    configMapRef:
      name: "myapp-dev-conf"
```

Here, myapp-staging1 and myapp-staging2 use a ConfigMap called myapp-staging-conf, while myapp-dev uses myapp-dev-conf.

In our reconciler, when a ConfigMap is updated, we want to update only the MyApp instances that use that particular ConfigMap. This is where indexing becomes handy, allowing us to somehow correlate MyApp instances with their corresponding ConfigMaps.

#### Indexing Implementation

To find dependent MyApp instances, we need to find all MyApp instances using a particular ConfigMap. For example, when myapp-staging-conf is updated, we want to trigger reconciliation for both myapp-staging1 and myapp-staging2.

To achieve that, we will add an index to the MyApp custom resource containing the name of its associated ConfigMap. This enables efficient retrieval of MyApp resources through this index.

In controller-runtime, indexers are configured through the Cache interface, which provides methods to index and retrieve Kubernetes objects efficiently. The cache wraps client-go informers and provides additional capabilities, including indexing.

When creating a new manager with `ctrl.NewManager()`, it initializes a cache (`cache.Cache`) to hold informers and indexers.

Before starting the manager, add indexers via the `FieldIndexer`:

```go
mgr.GetFieldIndexer().IndexField(
    context.Background(),
    &v1alpha1.MyApp{},
    "spec.config.configMapRef.name", // Can be any string, not limited to field names
    func(obj client.Object) []string {
        myApp := obj.(*v1alpha1.MyApp)
        cmName := myApp.Spec.Config.ConfigMapRef.Name
        return []string{cmName}
    },
)
```

This indexes MyApp resources based on the `myApp.Spec.Config.ConfigMapRef.Name` field. When `mgr.Start()` is called, the cache initializes all informers and configures the indexers.

In the reconciler, we can use controller-runtime client with indexing capabilities to efficiently query objects:

```go
myAppList := &v1alpha1.MyAppList{}
listOps := &client.ListOptions{
    FieldSelector: fields.OneTermEqualSelector("spec.config.configMapRef.name", "myapp-staging-conf"),
}

r.List(ctx, myAppList, listOps)
```

This lists all MyApp resources that use the ConfigMap "myapp-staging-conf". The Kubernetes client searches the cache using the index key "spec.config.configMapRef.name".

#### Finishing the Controller initialization

Now that we can list MyApp resources based on their associated ConfigMap, we can avoid unnecessary reconciliations. For example, when myapp-staging-conf is updated, we don't need to reconcile the myapp-dev resource since it uses a different ConfigMap.

```go
ctrl.NewControllerManagedBy(mgr).
    For(&v1alpha1.MyApp{}).
    Watches(
        &corev1.ConfigMap{},
        handler.EnqueueRequestsFromMapFunc(
            func(ctx context.Context, configMap client.Object) []reconcile.Request {
                myAppList := &v1alpha1.MyAppList{}
                listOps := &client.ListOptions{
                    FieldSelector: fields.OneTermEqualSelector(
                        "spec.config.configMapRef.name", configMap.GetName(),
                    ),
                }

                if err := r.List(ctx, myAppList, listOps); err != nil {
                    return []reconcile.Request{}
                }

                requests := make([]reconcile.Request, len(myAppList.Items))
                for i := range myAppList.Items {
                    requests[i] = reconcile.Request{
                        NamespacedName: types.NamespacedName{
                            Name:      myAppList.Items[i].Name,
                            Namespace: myAppList.Items[i].Namespace,
                        },
                    }
                }

                return requests
            }),
        builder.WithPredicates(predicate.NewPredicateFuncs(
            func(object client.Object) bool {
                // No ConfigMap filtering currently.
                // Add predicate functions here to filter ConfigMaps
                // For example, consider ConfigMaps with specific
                // annotations like `myapp/config`
                // The Enqueue function runs based on this return value
                return true
            }),
        ),
    ).
    Complete(r)
```

Instead of owning all ConfigMap resources, the controller now watches ConfigMaps.
With the help of predicate functions, we can filter ConfigMap resources events before enqueuing MyApp
resources for reconciliation.
After passing predicates, we specify which MyApp resources to reconcile within `EnqueueRequestsFromMapFunc`.
Using our index, we can find MyApp instances using the updated ConfigMap, ensuring efficient reconciliation of only the relevant resources.

### How

Until now, we see how we can use controller-runtime to index resources by going over a simple scenario.

My purpose of writing this blog post is actually sharing the implementation details about indexing.
Kubernetes itself includes lots of great and structured code pieces itself. Indexing is done
actually through two packages; client-go and controller-runtime. Even though we use controller-runtime, under the hood
it also leverages client-go to make things happen.
In the rest of the blogs let's see some details related to indexing. Of course I cannot explain all
implementation details but these are the things i like while i was digging into controller-runtime and
client-go.

### Setup

First things first, we need Manager from controller-runtime to manage MyApp controller and its dependencies.
As per my previous post, we use `NewManager` from [controller-runtime/pkg/manager]() to instantiate a new manager.

Here, we have options to configure the Manager. I am not going to dive into this part as it is covered in my other blogs.
The important in this Manager configuration is cache configuration. In most of the times, `NewCache` option of Manager is not configured
and used with default value. NewCache is low level primitive and usually developers do not need to configure it. As per doc,

```go
// NewCache is the function that will create the cache to be used
// by the manager. If not set this will use the default new cache function.
//
// When using a custom NewCache, the Cache options will be passed to the
// NewCache function.
//
// NOTE: LOW LEVEL PRIMITIVE!
// Only use a custom NewCache if you know what you are doing.
NewCache cache.NewCacheFunc
```

So, the default NewCache function returns instantiates `informerCache` from controller-runtime that uses NewSharedIndexInformer from client-go - unless
multi namespace is configured for cache. In that case multiNamespaceCache from controller-runtime will be used.

Now, our cache is instantiated and attached to Manager. What happens when we run following code piece to create an indexer for us?

```go
mgr.GetFieldIndexer().IndexField(context.Background(), &pkg.MyApp{}, "spec.config.configMapRef.name", func(obj client.Object) []string {
        myApp := obj.(*pkg.MyApp)
        cmName := myApp.Spec.Config.ConfigMapRef.Name
        return []string{cmName}
    })
```

> The function passed into `IndexField` will be reffered as F later in the post.

The FieldIndexer here is the informerCache that we created while setting up the Manager, as mentioned above.

IndexField gets informer of the given object, in our case MyAp, indexing key (spec.config.configMapRef.name), and a function F.
In F, if you pay attention, we did not specify namespace of the ConfigMap as it will be handled automatically by the controller-runtime under the hood.

controller-runtime takes F, and use it in `indexByField` function, here: https://github.com/kubernetes-sigs/controller-runtime/blob/v0.19.1/pkg/cache/informer_cache.go#L223.
After that controller-runtime automatically generates namespaced variant (if object is namespaced) and cluster scoped variants of the extracted values.

All of these operations are wrapped in an
[anonymous function](https://github.com/kubernetes-sigs/controller-runtime/blob/013f46fbca88cf495a783cdde6dda9ab5c3cd0b9/pkg/cache/informer_cache.go#L224-L257).
This anonymous function actually runs F to get []string{} result for each MyApp resources. The result of F will then be translated into special indexed value(s) as follows:

- `__all_namespaces/<MyApp.metadata.name>`
- `<MyApp.metadata.namespace>/<MyApp.metadata.name>` if resource (MyApp) is namespaced.

In our case, since MyApp is namespaced resource (not cluster scoped resource such as ClusterRole), controller-runtime adds namespace for us.
It also creates a special value with `__all_namespaces/` prefix for our index key (spec.config.configMapRef.name) in order to allow listing regardless of the object namespace in cache.

Then this anonymous function that wraps F is going to be passed into informer through adding an Indexer on the MyApp's informer.

### Indexer, IndexFunc and Indicies

We mentioned adding an Indexer. What is Indexer? What does it look like?
Indexer is actual core component used in indexing logic. It keeps track of Indexer names and functions used to calculate indexed values, such as our F.
Indexer names have special format as `field:<indexer_name>`. So, our indexer name is going to be `field:spec.config.configMapRef.name`.

Indexer and other related types are defined in client-go, as follows:

```go
type IndexFunc func(obj interface{}) ([]string, error)
type Indexers map[string]IndexFunc

type Index map[string]sets.String
type Indices map[string]Index
```

IndexFunc, by documentation, knows how to compute the set of indexed values for an object.
So, in our case, we defined a function F which is wrapped by controller-runtime (mentioned as anonymous function previously).
This is actually an IndexFunc. It is going to be used to know indexed values for given object - in our case MyApp.

For example, we have defined a custom indexer called `spec.config.configMapRef.name`.
When we first run IndexField, client-go adds this key to the indexer, as follows:

```
Indexers[field:spec.config.configMapRef.name]=<anonymous function generated in controller-runtime>
Indices = nil
```

Since we do not have any actual resource created at the time we call IndexField function, Indices is not yet populated; thus, nil.
On the other hand, since we registered a custom indexer, Indexer field is populated where the key of Indexer
is the one we speicfied in IndexField function and value is the aforementioned anonymous function.

For Indices and Index, now let's assume that we already have a MyApp instance called `myapptest`.

Indices will contain all indexes for particular indexer.
For our custom indexer `field:spec.config.configMapRef.name`, Indices will look like

```
"field:spec.config.configMapRef.name" -> set("default/myapptest", "__all_namespaces/myapptest")
```

So that it contains all indexes calculated by the indexer called `field:spec.config.configMapRef.name`.

Since we assumed MyApp resource called myapptest has been created, we have to Index for `field:spec.config.configMapRef.name`; one with namespace included ("default"), the other is without namespaces.
In the ListOptions/GetOptions while using controller-runtime library, if you do not specify a namespace, "\_\_all_namespaces/myapptest" will match your request.

---

Back to our actual work, we mentioned that `Then this anonymous function that wraps F is going to be passed into informer through adding an Indexer on the MyApp's informer.`-
controller-runtime does this here: https://github.com/kubernetes-sigs/controller-runtime/blob/v0.19.1/pkg/cache/informer_cache.go#L259.

MyApp's Informer (type of sharedIndexInformer) takes this request and pass this request to its field called [indexer](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/shared_informer.go#L553) that implements [Indexer](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/index.go#L35) interface.

The actual implementation of indexer field is [cache](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/store.go#L158) that has 1 core
field called `cacheStorage` which is an actual storage of the client-go cache. It implements [ThreadSafeStore](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/thread_safe_store.go#L41) interface.
Specifically, it uses [threadSafeMap](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/thread_safe_store.go#L224) implementation.

`threadSafeMap` implementation uses `items` (map[string]interface{}) for objects in the cache, and
`index` field which is a type of `storeIndex`.

```go
type threadSafeMap struct {
	lock  sync.RWMutex
	items map[string]interface{}
	index *storeIndex
}

// storeIndex implements the indexing functionality for Store interface
type storeIndex struct {
	indexers Indexers
	indices Indices
}
```

So, Indexers and Indicies mentioned previously are actually used here in client-go cache.

### Indexer in action

Now we registered an indexer. Now, let's create an instance of MyApp CR and let's see what happens.

When we create an instance of MyApp, [processDelta](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/controller.go#L541) function iterates through Deltas,
a list of one or more Delta where Delta means the type stored by a DeltaFIFO.
Delta represents what change happened, and the object's state after that change (according to client-go docs: https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/delta_fifo.go#L184)

Based on the delta type, processDelta either removes or updates cache in client-go.

Since the cache in client-go uses threadSafeMap, it checks our object's key which corresponds to
`<namespace>/<name>` of the MyApp resource on the cache.

Since we are creating the resource for the first time, this key does not exist on cache (specifially in threadSafeMap).

That's why, client-go adds the Object (MyApp instance) to our cache.

At the moment, we have two indexer; one is default namespace one - which works with <namespace>/<name> metadata of objects - another one is the Indexer we created
called field:spec.config.configMapRef.name.

client-go uses MetaNamespaceKeyFunc to make keys for cache, which generates <namespace>/<name> keys for namespaced objects. After generating the key for MyApp instance,
client-go updates underlying cache storage (threadSafeMap) by inserting the object to the key.

```
key: default/myapp-staging1
value: *pkg.MyApp
```

The cool part here is that client-go actually runs Update method under the hood, as follows:

```go
// First we call Add, where key: default/myapp-staging1 and value is the actual
// MyApp resource instance.
func (c *threadSafeMap) Add(key string, obj interface{}) {
	c.Update(key, obj)
}

// Then Update method is called by App method. It checks for given key on
// the threadSafeMap (which is an actual storage used in client-go cache) items.
// And runs updateIndices.
func (c *threadSafeMap) Update(key string, obj interface{}) {
	c.lock.Lock()
	defer c.lock.Unlock()
	oldObject := c.items[key]
	c.items[key] = obj
	c.index.updateIndices(oldObject, obj, key)
}

// This docs is directly taken from client-go code.
//
// updateIndices modifies the objects location in the managed indexes:
// - for create you must provide only the newObj
// - for update you must provide both the oldObj and the newObj
// - for delete you must provide only the oldObj
// updateIndices must be called from a function that already has a lock on the cache
func (i *storeIndex) updateIndices(oldObj interface{}, newObj interface{}, key string) {
	for name := range i.indexers {
		i.updateSingleIndex(name, oldObj, newObj, key)
	}
}
```

So based on oldObj and newObj values, updateSingleIndex knows which key or keys need to be added or
removed from index of the given indexer.

In our case, since this is first time we are populating the index, we have only newObj without oldObj meaning that
we are going to add key to index that simply add indexed value, for instance \_\_all_namespaces/myapp-staging-conf, to the
Set of the Index, as follows:

```go
objectKey := "default/myapp-staging1"
indexerName := "field:spec.config.configMapRef.name"
index := threadSafeMap.index.Indices[indexerName]

myAppListUsingSpecificConfigMap := index["__all_namespaces/myapp-staging-conf"]
if myAppListUsingSpecificConfigMap == nil {
    set = sets.String{}
    index["__all_namespaces/myapp-staging-conf"] = set
}
set.Insert(key)
```

### How to use this registered indexer?

When we call `r.List()`, controller-runtime checks if the object is uncached (or unstructured).
In our case, the objects are cached, controller-runtime client tries to List all MyApp
instances from cache, as follows:

```go
return c.cache.List(ctx, obj, opts...)
```

As we initialized informerCache (remember Manager setup), informerCache's List method (defined [here](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.19.1/pkg/cache/informer_cache.go#L91-L108))
is called.

Now, in informerCache's List method, we first get MyApp object's informer.
If the informer has started, we try to run List method on MyApp informer's cache.

In the list options passed to `r.List`, we specified our indexer key.

Remember following code piece:

```go
myAppList := &pkg.MyAppList{}
listOps := &client.ListOptions{
    FieldSelector: fields.OneTermEqualSelector(
        "spec.config.configMapRef.name", "myapp-staging-conf",
    ),
}

r.List(ctx, myAppList, listOps)
```

Cache looks for objects such that if you specified FieldSelector, it requires you to specify selector as `Exact Match`.
Then, controller-runtime goes through each Indexers (in ourcase we have two, one `field:spec.config.configMapRef.name` and `namespace` one.)

Since we haven't specified Namespace in ListOpts, controller-runtime is going to look for `__all_namespaces/myapp-staging-conf` indexed value
in the index called `field:spec.config.configMapRef.name`.

```
- indexName: "field:spec.config.configMapRef.name"
indexName is derived from ListOptions

- indexedValue: "spec.config.configMapRef.name"
indexedValue is derived from ListOptions
```

After getting these informations, controller-runtime actually forwards our request to client-go's
Indexer interface's [ByIndex](https://github.com/kubernetes/client-go/blob/v0.31.2/tools/cache/index.go#L47-L49) method which is documented as follows:

```go
// ByIndex returns the stored objects whose set of indexed values
// for the named index includes the given indexed value
ByIndex(indexName, indexedValue string) ([]interface{}, error)
```

Remember how we call this client-go method from controller-runtime

```go
objs, err = indexer.ByIndex("field:spec.config.configMapRef.name", "__all_namespaces/myapp-staging-conf")
```

Based on its documentation, ByIndex looks for stored objects for key "\_\_all_namespaces/myapp-staging-conf" in the index of "field:spec.config.configMapRef.name" indexer

This may sound complicated but the flow can be simplified in the following diagram:

```go
// the key is indexer's name, such as "field:spec.config.configMapRef.name"
// the value is IndexFunc, such as anonymous function mentioned previously.
type Indexer map[string]IndexFunc

// the key is indexer's name, such as "field:spec.config.configMapRef.name"
// the value is Index
type Indices map[string]Index

// the key is indexed value, such as __all_namespaces/myapp-staging-conf
// the value is set of string that corresponds to
// <namespace>/<name> metadata of the objects.
//  For ex, default/myapp-staging1
type Index map[string]sets.String
```

Now, we have set including <namespace>/<name> metadata of the MyApp resources. 
Then, threadSafeMap converts this set to list and List operation is done by using the
indexing functionality that we added.
