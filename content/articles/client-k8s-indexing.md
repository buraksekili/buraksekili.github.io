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
if err := mgr.GetFieldIndexer().IndexField(
    context.Background(),
    &v1alpha1.MyApp{},
    "spec.config.configMapRef.name", // Can be any string, not limited to field names
    func(obj client.Object) []string {
        myApp := obj.(*v1alpha1.MyApp)
        cmName := myApp.Spec.Config.ConfigMapRef.Name
        return []string{cmName}
    }); err != nil {
    return err
}
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
