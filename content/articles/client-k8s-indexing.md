---
editPost:
  #   URL: "https://github.com/buraksekili/buraksekili.github.io/blob/main/content/articles/thread-pooling-rs.md"
  Text: "Edit this page on "
author: "Burak Sekili"
title: "Kubernetes Client-Side Indexing"
date: "2024-10-27"
description: "How Kubernetes client-side indexing works: from implementation to internals"
tags: ["Go", "Kubernetes", "Operator", "controller-runtime"]
TocOpen: true
---

## Kubernetes Client-Side Indexing

> This post is part of Kubernetes controller development.
> Check out the first part on [Diving into controller-runtime | Manager](../controller-runtime-1) if you are interested in controller-runtime.

When working with Kubernetes Operators, read requests (get and list) to the Kubernetes API server are handled by an in-memory cache that is maintained by client-go to reduce the load on your API server. This cache can be enhanced with indexes to retrieve resources more efficiently.

This post explains the underlying indexing implementation in controller-runtime (and in client-go). We'll explore how indexing works under the hood in these Kubernetes client libraries.

If you're interested in practical examples of indexing, the Kubebuilder book and controller-runtime package documentation provide great practical examples and implementations.

### Purpose of Indexing

Indexing in Kubernetes client libraries (client-go and controller-runtime) optimizes the performance of Kubernetes controllers by enabling efficient lookups of cached objects based on specific attributes or relationships. This is crucial for:

1. Efficient Resource Retrieval: Finding resources matching certain criteria without scanning the entire cache. This is particularly useful when managing resources that depend on other resources.

2. Reduced API Server Load: Minimizing the number of API calls to the Kubernetes API server through caching. Most modern Kubernetes Operators utilize controller-runtime, where this behavior comes by default.

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

While previous sections covered how to use controller-runtime for indexing resources,
this section explores the underlying implementation details.

Kubernetes contains well-structured code patterns, and indexing is one of them.
Although we used controller-runtime in our examples, it leverages client-go under the hood to handle the actual indexing operations.

Let's examine the implementation details that make indexing possible.
While I can't cover every implementation detail, we'll focus on the most interesting aspects of controller-runtime and client-go's indexing mechanisms.

### Manager and Cache Setup

The first step is instantiating a Manager from controller-runtime to handle the MyApp controller and its dependencies. We'll use `NewManager` from [controller-runtime/pkg/manager](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.16.3/pkg/manager/manager.go#L315) for manager instantiation.

> If you're new to Manager or want to understand its role in depth, check out my post on [Diving into controller-runtime | Manager](../controller-runtime-1).

Although Manager offers various configuration options, I'll focus on the cache configuration, specifically the `NewCache` option.
We typically use the default cache configuration for NewCache since it is a low-level primitive that rarely needs customization except in specific use cases.
This is already mentioned in the docs, as follows:

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

By default, NewCache instantiates an `informerCache` instance from controller-runtime that
uses `NewSharedIndexInformer` informer from client-go.
The exception is when multi-namespace is configured for cache, in which case `multiNamespaceCache` from controller-runtime is used.

### Index Field Implementation

Now, our cache is instantiated and attached to Manager. What happens when we run following code piece to create an indexer for us?

```go
mgr.GetFieldIndexer().IndexField(
    context.Background(),
    &pkg.MyApp{},
    "spec.config.configMapRef.name",
    func(obj client.Object) []string {
        myApp := obj.(*pkg.MyApp)
        cmName := myApp.Spec.Config.ConfigMapRef.Name
        return []string{cmName}
    },
)
```

> The function passed into `IndexField` will be referred as F later in the post.

The FieldIndexer here is the informerCache that we created while setting up the Manager, as mentioned above.

IndexField gets informer of the given object, in our case MyAp, indexing key (spec.config.configMapRef.name), and a function F.
In F, if you notice, we did not specify the namespace of the ConfigMap as it will be handled automatically by the controller-runtime under the hood.

controller-runtime takes F and uses it in `indexByField` function (reference: [here](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.19.1/pkg/cache/informer_cache.go#L223))
and generates both namespaced and cluster-scoped variants of the extracted values.

All of these operations are wrapped in an
[anonymous function](https://github.com/kubernetes-sigs/controller-runtime/blob/013f46fbca88cf495a783cdde6dda9ab5c3cd0b9/pkg/cache/informer_cache.go#L224-L257).
This anonymous function actually runs F to get extracted values from each MyApp resources. The result of F will then be translated into special indexed value(s) as follows:

- `__all_namespaces/<MyApp.metadata.name>`
- `<MyApp.metadata.namespace>/<MyApp.metadata.name>` if resource (MyApp) is namespaced.

In our case, since MyApp is namespaced resource (not cluster scoped resource such as ClusterRole), controller-runtime adds namespace for us.
It also creates a special value with `__all_namespaces/` prefix for our index key (spec.config.configMapRef.name) in order to allow listing regardless of the object namespace in cache.

Then this anonymous function that wraps F is going to be passed into informer through adding an Indexer to the MyApp's informer.

### Understanding Indexers and Indices

We mentioned adding an Indexer to an informer but what is Indexer? What does it look like?

Indexer is actual core component used in indexing logic. It keeps track of Indexer names and functions used to calculate indexed values, such as our F.
Indexer names have a special format as `field:<indexer_name>`. So, our indexer name is going to be `field:spec.config.configMapRef.name`.

Indexer and other related types are defined in client-go, as follows:

```go
// IndexFunc returns indexed values for objects,
// such as ConfigMap names for MyApp resources like anonymous
// function (that wraps F) mentioned above.
type IndexFunc func(obj interface{}) ([]string, error)
// Indexers maintain indexer names as keys and
// their corresponding calculation functions as value.
type Indexers map[string]IndexFunc

// Indices stores the actual index data for each registered Indexer name.
type Indices map[string]Index
type Index map[string]sets.String
```

For example, when we create a custom indexer named `spec.config.configMapRef.name` by calling IndexField, client-go adds it to the indexer with a special format:

```
Indexers[field:spec.config.configMapRef.name]=<anonymous function generated in controller-runtime>
Indices=nil
```

Since we do not have any actual resource created at the time we call IndexField function, Indices is not yet populated; thus, nil.

For Indices and Index, assume that we already have a MyApp instance called `myapptest`.

Indices will contain all indexes for the particular indexer.
For our custom indexer `field:spec.config.configMapRef.name`, Indices is going to look like

```
"field:spec.config.configMapRef.name" -> set("default/myapptest", "__all_namespaces/myapptest")
```

It contains all indexes calculated by the indexer called `field:spec.config.configMapRef.name`.

### Cache Implementation Details

Back to our actual work, recall that the anonymous function wrapping F needs to be passed to the informer by adding an Indexer to the MyApp's informer. controller-runtime
handles this [here](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.19.1/pkg/cache/informer_cache.go#L259).

The MyApp's Informer (of type sharedIndexInformer) takes this request and passes it to its indexer [indexer](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/shared_informer.go#L553) field, which implements the [Indexer](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/index.go#L35) interface.

The indexer field is implemented by [cache](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/store.go#L158) that has a core
field called `cacheStorage` serving as as the actual storage of the client-go cache. It implements [ThreadSafeStore](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/thread_safe_store.go#L41) interface.
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

The Indexers and Indices we discussed earlier are implemented here in the client-go cache.

### Indexer in action

With the indexer registered, let's create an instance of MyApp CR and examine what happens.

When creating a MyApp instance, [processDelta](https://github.com/kubernetes/client-go/blob/3dc7fd5f4c1d8afaf5924c461eae2ab27db0045a/tools/cache/controller.go#L541) function handles cache updates; either removes or updates keys from the cache.

Since the cache in client-go uses threadSafeMap, it checks our object in its storage using the object's key. The key is generated by [MetaNamespaceKeyFunc](https://github.com/kubernetes/client-go/blob/v0.31.2/tools/cache/store.go#L108), with `<namespace>/<name>` format.

Since we are creating the resource for the first time, this key does not exist in the cache (specifially in threadSafeMap). That's why, client-go adds the object (MyApp instance) to its cache.

client-go updates underlying cache storage (threadSafeMap) by inserting the object into the key.

```
key: default/myapp-staging1
value: *pkg.MyApp
```

At the moment, we have two indexers: the default "namespace" indexer that works with `<namespace>/<name>` metadata of objects, and our custom indexer called "field:spec.config.configMapRef.name".
client-go iterates through these indexers
and updates them one by one.

The cool part here is that client-go actually runs Update method under the hood, as follows:

```go
// First we call Add, where key: default/myapp-staging1 and value is the actual
// MyApp resource instance.
func (c *threadSafeMap) Add(key string, obj interface{}) {
	c.Update(key, obj)
}

// Then Update method is called by Add method. It checks for the given key on
// the threadSafeMap (which is an actual storage used in client-go cache) items.
// And runs updateIndices.
func (c *threadSafeMap) Update(key string, obj interface{}) {
	c.lock.Lock()
	defer c.lock.Unlock()
	oldObject := c.items[key]
	c.items[key] = obj
	c.index.updateIndices(oldObject, obj, key)
}

// This doc is directly taken from client-go code.
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

Based on the oldObj and newObj values, updateSingleIndex determines which keys need to be added to or removed from the index of the given indexer.

The index update process follows this pattern:

```go
objectKey := "default/myapp-staging1"
indexerName := "field:spec.config.configMapRef.name"
index := threadSafeMap.index.Indices[indexerName]

// set corresponds to Set that contains all MyApp
// resources that use myapp-staging-conf ConfigMap.
set := index["__all_namespaces/myapp-staging-conf"]
if myAppListUsingSpecificConfigMap == nil {
    set = sets.String{}
    index["__all_namespaces/myapp-staging-conf"] = set
}
set.Insert(objectKey)
```

### Using the Indexer

When we call `r.List()`, controller-runtime checks if the object is uncached (or unstructured).
In our case, the objects are cached, controller-runtime client tries to List all MyApp
instances from the cache, as follows:

```go
return c.cache.List(ctx, obj, opts...)
```

As we initialized informerCache (recall Manager setup), controller-runtime client calls informerCache's List method (defined [here](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.19.1/pkg/cache/informer_cache.go#L91-L108)).
Now, in informerCache's List method, we first get MyApp object's informer.
If the informer has started, we try to run List method on MyApp informer's cache, which delegates
operation to client-go.

Recall the following code piece:

```go
myAppList := &pkg.MyAppList{}
listOps := &client.ListOptions{
    FieldSelector: fields.OneTermEqualSelector(
        "spec.config.configMapRef.name", "myapp-staging-conf",
    ),
}

r.List(ctx, myAppList, listOps)
```

In the list options passed to `r.List`, we specified our indexer key. When using a FieldSelector, the cache requires an exact match for the selector.
Then, controller-runtime goes through each Indexers (in ourcase we have two, one `field:spec.config.configMapRef.name` and `namespace` one.)

Since we haven't specified a Namespace in ListOpts, controller-runtime will look for the `__all_namespaces/myapp-staging-conf` indexed value
in the index called `field:spec.config.configMapRef.name`.

```
- indexName: "field:spec.config.configMapRef.name"
indexName is derived from ListOptions

- indexedValue: "spec.config.configMapRef.name"
indexedValue is derived from ListOptions
```

After getting this informations, controller-runtime actually forwards our request to client-go's
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

This may sound complicated but to simplify this flow, let's re iterate through the following types:

```go
// the key is indexer's name, such as "field:spec.config.configMapRef.name"
// the value is IndexFunc, such as the anonymous function mentioned previously.
type Indexer map[string]IndexFunc

// the key is indexer's name, such as "field:spec.config.configMapRef.name"
// the value is Index
type Indices map[string]Index

// the key is indexed value, such as __all_namespaces/myapp-staging-conf
// the value is set of strings that corresponds to
// <namespace>/<name> metadata of the objects.
//  For ex, default/myapp-staging1
type Index map[string]sets.String
```

Now, we have set including <namespace>/<name> metadata of the MyApp resources.
Then, threadSafeMap converts this Set to a list, and List operation is done by using the
indexing functionality that we added.

## Conclusion

Throughout this post, we've walked through Kubernetes client-side indexing,
exploring both its practical use and internal workings. While this post shares one approach to using indexing for improving controller performance,
there are many other patterns and best practices in the Kubernetes ecosystem that might better suit your specific needs.

If you're interested in learning more about controller development, you might find my [earlier post about Controller Runtime Manager](../controller-runtime-1) helpful, where I shared my experience with the core components of controllers.
The [Kubebuilder documentation](https://book.kubebuilder.io) is also an excellent resource that provides more comprehensive examples and different indexing use cases.

I hope sharing these experiences with client-side indexing has been helpful for your journey :) Every controller and operator has unique requirements, and I'd be interested to hear about your approaches to similar challenges. Feel free to share your thoughts or ask questions about implementing indexing in your controllers.

If you notice any mistakes or have feedback, feel free to reach out to me on [Twitter](https://x.com/buraksekili), [LinkedIn](https://www.linkedin.com/in/sekili/), or GitHub.

## References

- https://book.kubebuilder.io/
- https://pkg.go.dev/sigs.k8s.io/controller-runtime
