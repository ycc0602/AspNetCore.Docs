---
title: gRPC client-side load balancing
author: jamesnk
description: Learn how to make scalable, high-performance gRPC apps with client-side load balancing in .NET.
monikerRange: '>= aspnetcore-3.0'
ms.author: jamesnk
ms.date: 05/18/2021
no-loc: [Home, Privacy, Kestrel, appsettings.json, "ASP.NET Core Identity", cookie, Cookie, Blazor, "Blazor Server", "Blazor WebAssembly", "Identity", "Let's Encrypt", Razor, SignalR]
uid: grpc/loadbalancing
---
# gRPC client-side load balancing

By [James Newton-King](https://twitter.com/jamesnk)

gRPC client-side load balancing is a feature that allows gRPC clients to distribute load optimally across available servers. This article discusses how to configure a client-side load balancing to create scalable, high-performance gRPC apps in .NET.

gRPC client-side load balancing requires [Grpc.Net.Client](https://www.nuget.org/packages/Grpc.Net.Client) version XXXX or later.

## Configure gRPC client-side load balancing

Client-side load balancing is configured when a channel is created. The two components to consider when using load balancing:

* The address resolver, which resolvers the addresses.
* The load balancer, which picks the address that should be used for a gRPC call.

The address resolver is configured using the scheme of the address URI. For example, a `dns` scheme maps to `DnsAddressResolver`. This resolver gets the Internet Protocol (IP) addresses for the specified address.

```csharp
var channel = GrpcChannel.ForAddress(
    "dns://backend.default.svc.cluster.local",
    new GrpcChannelOptions { ChannelCredentials = ChannelCredentials.Insecure });
var client = new Greet.GreeterClient(channel);

var response = await client.SayHelloAsync(new HelloRequest { Name = "world" });
```

The preceding code:

* Configures the created channel with the address `dns://backend.default.svc.cluster.local`. The `dns` schema maps to `DnsAddressResolver`.
* Doesn't specify a load balancer. The channel defaults to `PickFirstLoadBalancer`.
* Starts the gRPC call `SayHello`.
  * `DnsAddressResolver` resolves IP addresses for the host name `backend.default.svc.cluster.local`. The result is cached and periodically refreshed. The host name value depends upon the environment.
  * `PickFirstLoadBalancer` attempts to connect to one of the resolved IP addresses.
  * The call is sent to the first IP address the channel successfully connects to.

A load balancer is specified in the service config. If no service config or load balancer is specified, the channel defaults to `PickFirstLoadBalancer`. Another load balancer implementation that is included is `RoundRobinLoadBalancer`. The `RoundRobinLoadBalancer` attempts to connect to all IP addresses returned by the address resolver. The load balancer then distributes load across all IP addresses.

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
  * `RoundRobinLoadBalancer` attempts to connect to all resolved addresses.
  * gRPC calls are distributed evenly using [round-robin](https://www.nginx.com/resources/glossary/round-robin-load-balancing/) logic.

## AddressResolver and LoadBalancer

gRPC client-side load balancing has two extension points: `AddressResolver` and `LoadBalancer`. Some implementations are built into gRPC for .NET.

* `AddressResolver` - Base type for resolving addresses for the client. An address resolver is selected using the address scheme.
  
  * `DnsAddressResolver` - Resolves addresses from a DNS host name. DNS host names are updated in the background every 5 seconds. DNS resolution is commonly used to load balance over pod instances that have a [Kubernetes headless services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services).
  * `StaticAddressResolver` - Resolves addresses from a static collection that is specified when the resolver is created. Useful if the addresses aren't dynamic.

* `LoadBalancer` - Base type for picking from the available addresses. A load balancer is configured in the service config.

  * `PickFirstLoadBalancer` - Iterates through the addresses and attempts to connect to them. The first successful connection is always used.
  * `RoundRobinLoadBalancer` - Attempts to connect to all addresses. gRPC calls are distributed across all connections using round-robin logic.

## Write and use custom resolvers and load balancers

TODO

## Why load balancing is important

HTTP/2 multiplexes multiple calls on a single TCP connection. If gRPC and HTTP/2 are used with a network load balancer (NLB), the connection is forwarded to a server and all gRPC calls are sent to that one server. The other server instances on the NLB are idle.

Network load balancers are a common solution for load balancing because they are fast and lightweight. For example, Kubernetes by default uses  a network load balancer to balance connections between pod instances. However, network load balancers are not effective at distributing load when used with gRPC and HTTP/2.

### Proxy or client-side load balancing?

gRPC and HTTP/2 can be effectively load balanced using either an application load balancer proxy or client-side load balancing. Both of these options allow individual gRPC calls to be distributed across available servers. Deciding between proxy and client-side load balancing is an architectural choice. There are pros and cons for each.

* **Proxy** - gRPC calls are sent to the proxy, the proxy makes a load balancing decision, and the gRPC call is sent on to the final endpoint. The proxy is responsible for knowing about endpoints. Using a proxy adds:

  * An additional network hop to gRPC calls.
  * Latency and consumes additional resources.
  * Proxy server must be setup and configured correctly.

* **Client-side load balancing** - The gRPC client makes a load balancing decision when a gRPC call is started. The gRPC call is sent directly to the final endpoint. When using client-side load balancing:

  * Client is responsible for knowing about available endpoints and making load balancing decisions.
  * Additional client configuration required.
  * High-performance, load balanced gRPC calls that eliminate the need for a proxy.

## Additional resources

* <xref:grpc/client>
