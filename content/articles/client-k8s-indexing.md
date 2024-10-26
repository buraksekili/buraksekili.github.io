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

## Kubernetes Client side indexing

If you have worked with Kubernetes Operators, you probably know that read requests (get and list) to the Kubernetes
API server are handled by in-memory caches maintained by client-go to reduce the load on your API server.
You may also know that indexes can enhance these caches to retrieve resources more efficiently.

This post explains the underlying indexing implementation in controller-runtime (and ultimately in client-go). We'll look at how indexing works under the hood in these Kubernetes client libraries.

If you are interested in practical examples of indexing, the Kubebuilder book and controller-runtime package documentation already provide practical examples.

### Purpose of Indexing

One of the purposes of indexing in Kubernetes client libraries (client-go and controller-runtime) is to optimize the performance of
Kubernetes controllers enable efficient lookups of cached objects based on specific attributes or relationships. This is crucial for:

1.  Efficient Resource Retrieval: Find resources matching certain criteria without scanning the entire cache. This becomes quite handy if you manage resources depending on other resources.

2.  Reduced API Server Load: Minimizing the number of API calls to the Kubernetes API server through the local cache.
    I assume most of Kubernetes Operators nowadays utilize controller-runtime where this behaviour comes by default.

3.  Responsive Reconciliation: Enabling controllers to react to changes in related resources. Here, I mean
    that sometimes you only want to reconcile specific resources based on changes in specific resources.

Let's go over these points through an example.

### Scenario

Let's go over the simplified version of an example used in kubebilder book.

> We are going to use controller-runtime in our examples.
> If you need general information about Kubernetes Operators and controller-runtime, you can
> have a look at this post.

Assume that you have a CRD called MyApp to deploy your application on Kubernetes, which uses ConfigMap for MyApp specific configurations.

MyApp spec refers to ConfigMap for the base configuration. Based on the configuration file, MyApp controller
reconciles to meet the desired state specified in the MyApp controller.

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

The desired behaviour in your mind is that whenever a ConfigMap specified via
`spec.config.configMapRef.name` is updated, your MyApp needs to reflect those changes.
In order to achieve this, MyApp needs to watch for ConfigMap resources.

The rough initialization of MyApp controller will look like the following:

```go
ctrl.NewControllerManagedBy(mgr).
    For(&v1alpha1.MyApp{}).
    Owns(&corev1.ConfigMap{})
```

where the controller manages MyApp CR as a primary resource and ConfigMap as a secondary resource. This
means that our controller will reconcile on MyApp CR and ConfigMap events, e.g., updating MyApp/ConfigMap.

The problem with the current approach is that even though we have a working solution,
the efficiency of the implementation is arguable as the controller will reconcile each change on MyApp and ConfigMap resources.
For example, any changes to any ConfigMap resources such as the ones not related to MyApp configuration will trigger reconciliation in this setup.
Usually, reconcile logic we implement is costly. They may send external API calls, send Write requests
to Kubernetes API etc. Therefore, as a developer, you may want to avoid running redundant reconciliations.

There are various way to achieve this, but idiomatic way to filter reconciliation requests goes through
Predicates and Event handlers. I will not go through the details of this as they explained in my other blog. However, we'll see some example use cases where indexing is going to be crucial to achieve this.

In our scenario assume that we have 3 MyApp instances created as follows:

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

Here, our myapp-staging1 and myapp-staging2 MyApp instances use a ConfigMap called myapp-staging-conf while
myapp-dev MyApp instance uses ConfigMap called myapp-dev-conf.

In our reconciler, whenever a ConfigMap is updated, we want to update MyApp instances that use this
particular ConfigMap.

That's where indexing becomes handy. To find out dependent MyApp instances, we want to correlate
MyApp instances with the ConfigMap they use.

#### Indexing

Given a ConfigMap, we want to find all MyApp instances using this particular ConfigMap.

Let's assume we updated myapp-staging-conf ConfigMap which needs to trigger reconciliation on myapp-staging1 and myapp-staging2 MyApp instances.

To do that, we are going to add an index to MyApp custom resource which contains the name of the
ConfigMap it uses. With this approach, we can then retrieve all MyApp resources through this particular
index.

In `controller-runtime`, indexers are set up through the Cache interface, which provides methods to
index and retrieve Kubernetes objects efficiently.
The cache wraps around the client-go informers and provides additional capabilities, including indexing.

When you create a new manager using `ctrl.NewManager()`, it initializes a cache (`cache.Cache`) which will hold informers and indexers.

Before starting the manager, you can add indexers via the `FieldIndexer`:

```go
if err := mgr.GetFieldIndexer().IndexField(
    context.Background(),
    &v1alpha1.MyApp{},
    "spec.config.configMapRef.name", // It can be anystring not only a field name such as `spec.config` etc.
    func(obj client.Object) []string {
        myApp := obj.(*v1alpha1.MyApp)
        cmName := myApp.Spec.Config.ConfigMapRef.Name
        return []string{cmName}
    }); err != nil {
    return err
}
```

This indexes MyApp resources based on myApp.Spec.Config.ConfigMapRef.Name field. When mgr.Start() is
called, the cache starts all informers and sets up the indexers as configured.
In the reconciler, we can use controller-runtime client with indexing capabilities to efficiently query objects:

```go
myAppList := &v1alpha1.MyAppList{}
listOps := &client.ListOptions{
    FieldSelector: fields.OneTermEqualSelector("spec.config.configMapRef.name", "myapp-staging-conf"),
}

r.List(ctx, myAppList, listOps)
```

This will list all MyApp resources on the cluster that use a ConfigMap called myapp-staging-conf.
Kubernetes client looks for MyApp resources in the cache by using an index key ("spec.config.configMapRef.name").

#### Finishing controller

Now, we are able to list MyApp resources based on ConfigMap they use. Therefore, we do not need to
run reconciliation on every MyApp instance if they do not use updated ConfigMap. What I mean by that
is if you update myapp-staging-conf ConfigMap, MyApp controller does not need to run reconciliation on
myapp-dev MyApp resource as this resource not using myapp-staging-conf.

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
                // At the moment, we do not filter any ConfigMap based on metadata etc.
                // You can filter some here by using predicate functions.
                // For example, you may want to consider ConfigMaps with specific
                // annotations such as `myapp/config` for reconciliation.
                // Such annotation checks can happen here.
                // Based on return value here (true or false), Enqueue function above
                // will run.
                return true
            }),
        ),
    ).
    Complete(r)
```

Now, instead of owning all ConfigMap resources in the controller setup, we watches ConfigMaps based on
prediciate function. After predicates are passed, we specify which MyApp resources will be reconciled
within EnqueueRequestsFromMapFunc. As you can see, we use the index we created to find out all
MyApp instances using the updated ConfigMap. Now, instead of running reconciliation over all MyApp
instances, we only run over the ones that use the particular ConfigMap.
