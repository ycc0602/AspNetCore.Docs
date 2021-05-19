---
title: Scalable apps with gRPC client-side load balancing
author: jamesnk
description: Learn how to make scalable, high-performance gRPC apps with client-side load balancing in .NET.
monikerRange: '>= aspnetcore-3.0'
ms.author: jamesnk
ms.date: 05/18/2021
no-loc: [Home, Privacy, Kestrel, appsettings.json, "ASP.NET Core Identity", cookie, Cookie, Blazor, "Blazor Server", "Blazor WebAssembly", "Identity", "Let's Encrypt", Razor, SignalR]
uid: grpc/loadbalancing
---
# Scalable apps with gRPC client-side load balancing

By [James Newton-King](https://twitter.com/jamesnk)

gRPC client-side load balancing is a feature that allows gRPC clients to distribute load optimally across available servers. This article discusses how to configure a client-side load balancing to create scalable, high-performance gRPC apps in .NET.

gRPC client-side load balancing requires [Grpc.Net.Client](https://www.nuget.org/packages/Grpc.Net.Client) version XXXX or later.

## Why load balancing is important

HTTP/2 multiplexes multiple calls on a single TCP connection. If gRPC and HTTP/2 are used with a network load balancer, the connection is forwarded to a server and then all gRPC calls are sent to that one server. The other server instances the network load balancer is supposed to distribute load to are idle.

Network load balancers are a common solution for load balancing because they are fast and lightweight. For example, Kubernetes will use a network load balancer to balance connections between pod instances by default. However, network load balancers are not effective at distributing load when used with gRPC and HTTP/2.

## Proxy or client-side load balancing?

gRPC and HTTP/2 can be effectively load balanced using either an application load balancer proxy or client-side load balancing. Both of these options allow individual gRPC calls to be distributed across available servers. Deciding between proxy and client-side load balancing is an architectural choice. There are pros and cons for each.

* **Proxy** - gRPC calls are sent to the proxy, the proxy makes a load balancing decision, and the gRPC call is sent on to the final endpoint. The proxy is responsible for knowing about endpoints. Using a proxy will add an additional network hop to gRPC calls. This will add latency and consume additional resources.

* **Client-side load balancing** - The gRPC client will make a load balancing decision when a gRPC call is started. The gRPC call is sent directly to the final endpoint. In this model, the client is responsible for knowing about available endpoints. Client-side load balancing is a high-performance solution that eliminates the need for a proxy and additional network hops.

## Configure gRPC client-side load balancing

Client-side load balancing is configured when a channel is created. The two components to consider when using load balancing are the address resolver, which will resolver the addresses; and the load balancer, which will pick which address should be used for a gRPC call.

The address resolver is configured using the scheme of the address URI. For example, a `dns` scheme maps to `DnsAddressResolver`. This resolver will get the Internet Protocol (IP) addresses for the specified address.

```csharp
var channel = GrpcChannel.ForAddress(
    "dns://backend.default.svc.cluster.local",
    new GrpcChannelOptions { ChannelCredentials = ChannelCredentials.Insecure });
var client = new Greet.GreeterClient(channel);

var response = await client.SayHelloAsync(new HelloRequest { Name = "world" });
```

The preceding code:

* Configures the created channel with the address `dns://backend.default.svc.cluster.local`. The `dns` schema maps to `DnsAddressResolver`.
* Doesn't specify a load balancer. The channel will default to `PickFirstLoadBalancer`.
* Starts the gRPC call `SayHello`.
  * `DnsAddressResolver` will resolve IP addresses for the host name `backend.default.svc.cluster.local`. The result is cached and periodically refreshed. The host name value will depend upon your environment.
  * `PickFirstLoadBalancer` will attempt to connect to one of the resolved IP addresses.
  * The call will be sent to the first IP address the channel successfully connects to.

A load balancer is specified in the service config. If no service config or load balancer is specified then the channel will default to `PickFirstLoadBalancer`. Another load balancer implementation that is included is `RoundRobinLoadBalancer`. This load balancer will attempt to connect to all IP addresses returned by the address resolver. The load balancer then distributes load across all IP addresses.

```csharp
var channel = GrpcChannel.ForAddress(
    "dns://custom-hostname",
    new GrpcChannelOptions
    {
        ChannelCredentials = ChannelCredentials.Insecure,
        ServiceConfig = new ServiceConfig { LoadBalancingConfigs = { new RoundRobinConfig() } }
    });
var client = new Greet.GreeterClient(channel);

var response = await client.SayHelloAsync(new HelloRequest { Name = "world" });
```

The preceding code:

* Specifies a round robin load balancer in the service config.
* Starts the gRPC call `SayHello`.
  * `RoundRobinLoadBalancer` will attempt to connect to all resolved addresses.
  * gRPC calls are distributed evenly across the addresses using round-robin logic.

## AddressResolver and LoadBalancer

gRPC client-side load balancing has two extension points: `AddressResolver` and `LoadBalancer`. Some implementations of these types are built-in to gRPC for .NET.

* `AddressResolver` - Resolves the available endpoints for the client. An address resolver is selected using the address scheme.
  
  * `DnsAddressResolver` - Resolve addresses from a DNS host name. DNS host names are updated in the background every 5 seconds. DNS resolution is commonly used to load balance over pod instances that have a [Kubernetes headless services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services).
  * `StaticAddressResolver` - Resolve addresses from a static collection that is specified when the resolver is created. Useful if the addresses aren't dynamic.

* `LoadBalancer` - Picks from the available addresses based on the load balancing implementation. A load balancer is configured in the service config.

  * `PickFirstLoadBalancer` - Iterate through the addresses and attempting to connect to them. The first successful connection will always be used.
  * `RoundRobinLoadBalancer` - Attempts to connect to all addresses. gRPC calls are distributed across all connections using round-robin logic.

## Write and use custom resolvers and load balancers



## Additional resources

* <xref:grpc/client>
* [Retry general guidance - Best practices for cloud applications](/azure/architecture/best-practices/transient-faults)
