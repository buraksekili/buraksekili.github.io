---
editPost:
  URL: "https://github.com/buraksekili/buraksekili.github.io/blob/main/content/articles/konnectivity-xfr-channel-size.md"
  Text: "Edit this page on "
author: "Burak Sekili"
title: "Konnectivity xfr-channel-size Configuration Guide: Understanding Konnectivity"
date: "2026-02-07"
description: "Learn how to configure and tune xfr-channel-size in Konnectivity. Understand buffer channels, fix channel full warnings, and optimize memory usage for production Kubernetes clusters."
tags: ["Kubernetes", "Networking"]
TocOpen: true
---

# Understanding xfr-channel-size Configuration in Konnectivity

I have often encountered situations where network connectivity between the API server and cluster nodes becomes complicated. Firewalls, network policies, and VPC configurations can block direct communication.
This is where Konnectivity comes in. It provides a network proxy system that tunnels traffic through these barriers.
During my work with Konnectivity, I encountered the warning message `Receive channel from agent is full` in my cluster logs. I asked the Kubernetes community for help but didn't receive a response.
I also couldn't find any documentation explaining what this warning meant or how to fix it, or even any documentation about this component :(.
So, this lead me to reading the source code to understand what `xfr-channel-size` actually does, which controls data buffering, and getting it right can make a difference in how your cluster performs under load.

In this article I will explain what `xfr-channel-size` does, why it exists, and how you can tune it for your environment based on my experience.

## What is Konnectivity?

Konnectivity is one of those Kubernetes components that works quietly in the background until something breaks.
You don't hear much about it, and good luck finding comprehensive documentation.
It's like the quiet cousin of the more famous Kubernetes networking components.
But it plays a critical role in many production clusters, especially managed Kubernetes services where the control plane is separated from worker nodes.

## What is Konnectivity?

Konnectivity is a Kubernetes component that solves the network connectivity problem between the Kubernetes API server and cluster nodes.
In some deployments, the API server cannot directly reach kubelets or pods because of network policies, firewalls, or VPC configurations (like managed Kubernetes offerings where control plane is managed by cloud provider while data plane nodes are managed by customers).
Konnectivity creates a tunnel through these barriers.
The system has two main components.
The proxy server runs alongside the Kubernetes API server in the control plane.
The agent runs as a daemon set or deployment on each worker node.
When the API server needs to communicate with a kubelet or pod, it sends the request through the proxy server.
The proxy server forwards it to the appropriate agent, and the agent delivers it to the destination. Responses follow the same path in reverse.

## Why Do We Need Buffers?

To understand why we need buffers, let me explain what would happen without them.
In Go, if you try to send data through a channel and nobody is receiving yet, the sender blocks.
It waits until someone receives the data.
In a network proxy system, this would be problematic.
Imagine the Kubernetes API server sends a request to the Konnectivity proxy server.
The proxy server needs to forward this to an agent.
But what if the agent is busy, or the network is slow, or the kubelet is overloaded?
Without a buffer, the proxy server would block while waiting for the agent.
It could not accept new requests from the API server during this time.
The entire system would stall.
Buffers solve this problem by providing temporary storage.
The proxy server can accept the request into a buffer and immediately be ready to accept more requests.
The request sits in the buffer until the agent is ready to receive it.
This makes the system asynchronous and more resilient to slow components.

## The Three Channel Types

Konnectivity uses three different types of buffer channels, each serving a specific purpose in the data path.

### Frontend Channel

The frontend channel exists on the Konnectivity server. It sits in the data path between the Kubernetes API server and the Konnectivity agent.
When the API server sends a request to a kubelet or pod, the request first reaches the Konnectivity server through the frontend. The server places the packet into the frontend channel.
This channel buffers packets that arrive from the API server before they are forwarded to the agents.
The purpose of this buffer is to handle bursts of requests from the API server. Even when agents are slow to receive packets, the server can continue accepting requests from the API server because it has this buffer space. 
The frontend channel fills up when agents are the bottleneck. This happens when agents are overloaded, when network latency to agents is high, or when agents are handling too many connections at once.

### Backend Channel

The backend channel also exists on the Konnectivity server, but it handles traffic in the opposite direction.
When agents send responses back to the Kubernetes API server, those responses go through the backend channel.
This channel buffers packets that arrive from agents before they are delivered to the API server.
The purpose is to protect agent connections from congestion on the API server side.
If the API server is busy and cannot process incoming messages quickly, the responses from agents can wait in this buffer. The agent connections do not block waiting for the API server. The backend channel fills up when the Kubernetes API server is the bottleneck. This may happen when the API server is overloaded, when it is processing expensive operations, or when it cannot keep up with the incoming response traffic.

### Agent Channel

The agent channel exists on the Konnectivity agent, not on the server. It sits in the last mile of the data path between the proxy server and the kubelet.
This channel buffers data before sending it to the actual backend service.
The agent receives data from the proxy server over a gRPC connection.
It needs to write this data to the actual TCP connection to the kubelet or pod.
The agent channel sits between these two operations.
It buffers the data so the agent can continue receiving from the proxy server even when the kubelet or pod is slow to accept data.
The agent channel typically needs to be larger than the server channels because the path to kubelets has the most variability.
Endpoints can be slow, connections can have high latency, and the agent handles actual data payloads which can be much larger than packet headers.

## Understanding Warning Messages

Konnectivity logs warning messages when these channels reach capacity. Understanding these warnings helps you diagnose performance issues in your cluster.

### Receive channel from frontend is full

When you see this message in the Konnectivity server logs, it means the frontend channel is full.
The Kubernetes API server is sending requests faster than the server can deliver them to the agents.
The agents are the bottleneck in the system.
They might be slow because of network latency, overload, or too many concurrent connections.
You can monitor this situation with `konnectivity_network_proxy_server_full_receive_channels` metric.
When you see this warning frequently, consider increasing the server `xfr-channel-size` or investigating why your agents are slow.

### Receive channel from agent is full

When you see this message, it means the backend channel is full. The Konnectivity server is receiving responses from agents faster than it can deliver them to the Kubernetes API server.
The API server is the bottleneck.
It might be busy processing expensive operations or simply overloaded.
You can monitor this with the same metric.
When you see this warning frequently, the issue is on the API server side. Increasing the server `xfr-channel-size` might help, but you should also investigate why the API server cannot process messages quickly enough.

### Data channel on agent is full

When you see this message in the agent logs, it means the agent channel is full.
The Konnectivity server is sending data faster than the agent can write to the kubelet or pod.
The data is queuing up on the agent side waiting to be written to the backend.
This typically happens when the kubelet or pod is slow to accept data because of load or network conditions.
When you see this warning frequently, consider increasing the agent `xfr-channel-size` or investigating the performance of your kubelets and pods.

> It is important to understand that these messages are **WARNINGS**, not errors.The system is still working. Packets are not lost. They are just delayed in the buffers. The warnings tell you that the system has some backpressure and things are moving slower than ideal, but no requests are failing.

## Default values

The Konnectivity server defaults to 10 packets for xfr-channel-size. The Konnectivity agent defaults to 150 packets.
These different values reflect the different roles of the components.

> Again, i could not find the motivations behind these magical numbers - so, the explanation below is purely depending on my understanding.

The server handles many more concurrent connections, so it uses smaller buffers to control memory usage.
The agent handles fewer connections but each one requires more buffer space because of the variability in the last mile to kubelets and pods.
The agent also handles actual data payloads rather than just packet headers, so it needs more buffer space.

### When to increase channel size

You should increase the channel size when you frequently see the channel full warning messages.
If you observe high values on the `konnectivity_network_proxy_server_full_receive_channels` metric, this indicates sustained backpressure.
You should also consider increasing the size when you experience latency spikes during traffic bursts.
A larger buffer can absorb the burst without causing backpressure. If you proxy to slow or overloaded endpoints, like kubelets on overloaded nodes, a larger agent channel can help smooth out the variability.

> You do not need to match the channel size between the agent and the server. They serve different purposes and operate at different scales.
> The agent channel size affects how much data can wait to be written to each backend connection.
> The server channel sizes affect how many packets can wait for every active connection through the proxy. Tune them independently based on the warning messages you see and the memory constraints in your environment.

## References

The research for this article is based on the apiserver-network-proxy codebase (unfortunately for me :)).
You can find the server frontend channel implementation in [pkg/server/server.go](https://github.com/kubernetes-sigs/apiserver-network-proxy/blob/d6ea7d8286376fbe19b19b281f008504bccf5bb3/pkg/server/server.go#L769).
The server backend channel warning is at the same [package](https://github.com/kubernetes-sigs/apiserver-network-proxy/blob/d6ea7d8286376fbe19b19b281f008504bccf5bb3/pkg/server/server.go#L842).

If you are also interested in konnectivity, you may find the following Kubecon talks interesting
- [Proxy Service: A New Network Traffic Abstraction in Kubernetes - Walter Fender & Yongkun Gui, Google](https://www.youtube.com/watch?v=y0DBopR17-s)
- [Remote Control Planes With Konnectivity; What, Why And How? - Jussi Nummelin & Rastislav Szabo](https://www.youtube.com/watch?v=0yltsB3Cbr4)
