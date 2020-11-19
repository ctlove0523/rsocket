## Motivations动机

大型分布式系统通常由不同的团队使用多种技术和编程语言以模块化的方式实现。分布式系统中的部件需要可靠的通信并支持快速、独立的演进。模块之间高效、可扩展的通信是分布式系统中的关键问题，会影响用户体验（时延）以及构建和运行系统需要的资源。

Reactive Manifesto中记录，Reactive Streams和Reactive Extensions类库中实现的架构模式不仅支持异步消息传递，并且包含请求/响应之外的通信模式。“RSocket”协议是包含“反应式”原理的正式通信协议。

以下是定义新协议的动机：

#### Message Driven消息驱动

网络通信是异步的。RSocket协议包含了这一点，并将所有通信建模为单个网络连接上的多路消息流，并且在等待响应时从不同步阻塞。

反应式宣言：

>Reactive Systems依靠异步消息传递来在组件之间建立边界，以确保松耦合，隔离，位置透明，并提供将错误委托为消息的方法。采用显式消息传递，可以通过对系统中的消息队列进行重塑和监视并在必要时施加背压来实现负载管理，弹性和流量控制。非阻塞通信允许接收者仅在活动时消耗资源，从而减少系统开销。

此外，HTTP/2的FAQ很好地解释了通过复用持久连接的形式采用面向消息的协议的动机：

> HTTP/1.x存在一个称为“行头阻塞”的问题，实际上一次只能在一个连接上处理一个请求。

> HTTP/1.1尝试通过pipeline解决这个问题，但是问题并没有得到完全解决（一个大或者慢的响应仍然会阻塞之后的响应）。此外，部署pipeline非常困难，因为许多中介和服务器无法正确处理pipeline。

> 这迫使客户使用多种试探法（通常是猜测法）来确定什么请求在什么时候对原点进行哪个连接；由于页面加载的可用连接数通常是10的倍数（或更多），因此会严重影响性能，通常会导致被阻止的请求“泛滥”。

> 复用通过允许同时发送多个请求和响应消息来解决这些问题。甚至有可能将一条消息的一部分与另一条消息混合在一起。

> 反过来，这允许客户端仅使用一个连接来加载页面。

再议持久化连接：

> 使用HTTP/1，浏览器在每个源之间打开四个到八个连接。 由于许多站点使用多个源，因此这可能意味着单个页面加载会打开三十多个连接。

> 一个应用程序同时打开如此多的连接打破建立许多TCP连接的假设。由于每个连接都会在响应中引发大量数据，因此中间网络中的缓冲区溢出是真正的风险，从而导致拥塞事件并重传。

> 此外，使用如此多的连接会不公平地垄断网络资源，将它们从其他性能更好的应用程序（例如VoIP）“窃取”。



#### Interaction Models互动模型

不恰当的协议会增加开发系统的成本。不恰当的协议可能是不匹配的抽象从而迫使系统设计使用协议允许的模式。使用不恰当的协议，开发人员将花费更多时间来解决其缺点，以处理错误并获得可接受的性能。在多语言环境中，问题会被放大，因为不同的语言将使用不同的方法来解决此问题，并且需要团队之间进行额外的协作。 迄今为止，事实上的标准是HTTP，所有内容都是请求/响应。 在某些情况下，这可能不是实现特定功能的理想通信模型。

推送通知就是一个例子。 使用请求/响应模式会强制应用程序执行轮询，在轮询过程中客户端会不断发送请求以检查服务器中的数据。 无需花太多时间就可以找到应用程序每秒执行大量请求并被告知没有任何客户端需要的内容的例子。 这对于客户端，服务器和网络都是浪费，要花钱，增加基础架构的大小，增加操作的复杂性，从而提高可用性。 通常，它还会增加接收通知的用户体验的等待时间，因为会使用更长的时间间隔轮询以降低成本。

由于这个和其他原因，RSocket不仅限于一个交互模型。下文描述的各种受支持的交互模型为系统设计打开了强大的新可能性：


##### Fire-and-Forget

Fire-and-forget是对请求/响应模式的一种优化，在不需要响应的时候更加有用。Fire-and-Forget通过跳过响应来减少对网络的使用量，客户端和服务器因为无需记录需要等待和关联响应或者取消请求从而节省了处理时间，带来了巨大的性能提升。

对于允许有损性的场景（例如非关键事件日志记录），此交互模型十分有用。

使用方法如下：

```java
Future<Void> completionSignalOfSend = socketClient.fireAndForget(message);
```

##### Request/Response (single-response)

Standard request/response semantics are still supported, and still expected to represent the majority of requests over a RSocket connection. These request/response interactions can be considered optimized "streams of only 1 response", and are asynchronous messages multiplexed over a single connection. 

The consumer "waits" for the response message, so it looks like a typical request/response, but underneath it never synchronously blocks.

Usage can be thought of like this:

```java
Future<Payload> response = socketClient.requestResponse(requestPayload);
```

##### Request/Stream (multi-response, finite) 

Extending from request/response is request/stream, which allows multiple messages to be streamed back. Think of this as a "collection" or "list" response, but instead of getting back all the data as a single response, each element is streamed back in order.

Use cases could include things like:

- fetching a list of videos
- fetching products in a catalog
- retrieving a file line-by-line

Usage can be thought of like this:

```java
Publisher<Payload> response = socketClient.requestStream(requestPayload);
```

##### Channel

A channel is bi-directional, with a stream of messages in both directions. 

An example use case that benefits from this interaction model is:

- client requests a stream of data that initially bursts the current view of the world
- deltas/diffs are emitted from server to client as changes occur
- client update the subscription over time to add/remove criteria/topics/etc

Without a bi-directional channel, the client would have to cancel the initial request, re-request and receive all data from scratch, rather than just updating the subscription and efficiently getting just the difference. 

Usage can be thought of like this:

```java
Publisher<Payload> output = socketClient.requestChannel(Publisher<Payload> input);
```

#### Behaviors行为

Beyond the interaction models above, there are other behaviors that can benefit applications and system efficiency. 

##### single-response vs multi-response

One key difference between single-response and multi-response is how the RSocket stack delivers data to the application: A single-response might be carried across multiple frames, and be part of a larger RS connection that is streaming multiple messages multiplexed. But single-response means the application only gets its data when the entire response is received. While multi-response delivers it piecemeal. This could allow the user to design its service with multi-response in mind, and then the client can start processing the data as soon as it receives the first chunk.

##### Bi-Directional

RSocket supports bi-directional requests where both client and server can act as requester or responder. This allows a client (such as a user device) to act as a responder to requests from the server. 

For example, a server could query clients for trace debug information, state, etc. This can be used to reduce infrastructure scaling requirements by allowing server-side to query when needed instead of having millions/billions of clients constantly submitting data that may only occasionally be needed. This also opens up future interaction models currently not envisioned between client and server.

##### Cancellation

All streams (including request/response) support cancellation to allow efficient cleanup of server (responder) resources. This means that when a client cancels, or walks away, servers are given the chance to terminate work early. This is essential with interaction models such as streams and subscriptions, but is even useful with request/response to allow efficient adoption of approaches such as "backup requests" to tame tail latency (more information [here](http://highscalability.com/blog/2012/3/12/google-taming-the-long-latency-tail-when-more-machines-equal.html), [here](http://highscalability.com/blog/2012/6/18/google-on-latency-tolerant-systems-making-a-predictable-whol.html), [here](http://www.bailis.org/blog/doing-redundant-work-to-speed-up-distributed-queries/), and [here](http://static.googleusercontent.com/external_content/untrusted_dlcp/research.google.com/en/us/people/jeff/Stanford-DL-Nov-2010.pdf))


#### Resumability可恢复性

With long-lived streams, particularly those serving subscriptions from mobile clients, network disconnects can significant impact cost and performance if all subscriptions must be re-established. This is particularly egregious when the network is immediately reconnected, or when switched between Wifi and cell networks. 

RSocket supports session resumption, allowing a simple handshake to resume a client/server session over a new transport connection.


#### Application Flow Control应用流控

RSocket supports two forms of application-level flow control to help protect both client and server resources from being overwhelmed.

This protocol is designed for use both in datacenter, server-to-server, use cases, as well as server-to-device use cases over the internet, such as to mobile devices or browsers. 

##### "Reactive Streams" `request(n)` Async Pull

This first form of flow control is suited to both server-to-server and server-to-device use cases. It is inspired by the Reactive Streams [Subscription.request(n)](https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.0/README.md#3-subscription-code) behavior. [RxJava](https://github.com/ReactiveX/RxJava/), [Reactor](https://github.com/reactor/reactor), and [Akka Streams](http://doc.akka.io/docs/akka/2.4/scala/stream/index.html) are examples of implementations using this form of "async pull-push" flow control.

RSocket allows for the `request(n)` signal to be composed over network boundaries from requester to responder (typically client to server). This controls the flow of emission from responder to requestor using Reactive Streams semantics at the application level and enables use of bounded buffers so rate of flow adjusts to application consumption and not rely solely on transport and network buffering.

This same data type and approach has been adopted into Java 9 in the `java.util.concurrent.Flow` [suite of types](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.Subscription.html).

##### Leasing

The second form of flow control is primarily focused on server-to-server use cases in a data center. When enabled, a responder (typically a server) can issue leases to the requester based upon its knowledge of its capacity in order to control requests rates. On the requester side, this enables application level load balancing for sending messages only to responders (servers) that have signalled capacity. This signal from server to client allows for more intelligent routing and load balancing algorithms in data centers with clusters of machines. 


#### Polyglot Support多语言支持

Many of the motivations above can be achieved by leveraging existing protocols, libraries, and techniques. However, this often ends up being tightly coupled with specific implementations that must be agreed upon across languages, platforms and tech stacks. Formalizing the interaction models and flow control behaviors into a protocol provides a contract between implementations in different languages. This in turn improves polyglot interactions in a broader set of behaviors than the ubiquitous HTTP/1.1 request/response, while also enabling Reactive Streams application level flow control across languages (rather than just in Java for example where Reactive Streams was originally defined).

#### Transport Layer Flexibility灵活的传输层

Just like HTTP request/response is not the only way applications can or should communicate, TCP is not the only transport layer available, and not the best for all use cases. Thus, RSocket allows for swapping of the underlying transport layer based on environment, device capabilities and performance needs. RSocket (the application protocol) targets WebSockets, TCP, and [Aeron](https://github.com/real-logic/Aeron), and is expected to be usable over any transport layer with TCP-like characteristics, such as [Quic](https://www.chromium.org/quic).

Perhaps more importantly though, it makes TCP, WebSockets and Aeron usable without significant effort. For example, use of WebSockets is often appealing, but all it exposes is framing semantics, so using it requires the definition of an application protocol. This is generally overwhelming and requires a lot of effort. TCP doesn't even provide framing. Thus, most applications end up using HTTP/1.1 and sticking to request/response and missing out on the benefits of interaction models beyond synchronous request/response.

Thus, RSocket defines application layer semantics over these network transports to allow choosing them when they are appropriate. Later in this document is a brief comparison with other protocols that were explored while trying to leverage WebSockets and Aeron before determining that a new application protocol was wanted.

#### Efficiency & Performance效率和性能

A protocol that uses network resources inefficiently (repeated handshakes and connection setup and tear down overhead, bloated message format, etc.) can greatly increase the perceived latency of a system. Also, without flow control semantics, a single poorly written module can overrun the rest of the system when dependent services slow down, potentially causing retry storms that put further pressure on the system. [Hystrix](https://github.com/Netflix/Hystrix/wiki#problem) is an example solution trying to address the problems of synchronous request/response. It comes [at a cost](https://github.com/Netflix/Hystrix/wiki/FAQ#what-is-the-processing-overhead-of-using-hystrix) though in overhead and complexity.

Additionally, a poorly chosen communication protocol wastes server resources (CPU, memory, network bandwidth). While that may be acceptable for smaller deployments, large systems with hundreds or thousands of nodes multiply the somewhat small inefficiencies into noticeable excess. Running with a huge footprint leaves less room for expansion as server resources are relatively cheap but not infinite. Managing giant clusters is much more expensive and less nimble even with good tools. And often forgotten, the larger a cluster is, the more operationally complex it is, which becomes an availability concern. 

RSocket seeks to:

- Reduce perceived latency and increase system efficiency by supporting non-blocking, duplex, async application communication with flow control over multiple transports from any language.

- Reduce hardware footprint (and thus cost and operational complexity) by:
   - increasing CPU and memory efficiency through use of binary encoding
   - avoid redundant work by allowing persistent connections

- Reduce perceived user latency by:
   - avoiding handshakes and the associated round-trip network overhead
   - reducing computation time by using binary encoding
   - allocating less memory and reducing garbage collection cost


## Comparisons比较

Following is a brief review of some protocols reviewed before deciding to create RSocket. It is not trying to be exhaustive or detailed. It also does not seek to criticize the various protocols, as they all are good at what they are built for. This section is meant solely to express that existing protocols did not sufficiently meet the requirements that motivated the creation of RSocket.

For context: 

- RSocket is an OSI Layer 5/6, or TCP/IP Application Layer protocol. 
- It is intended for use over duplex, binary transport protocols that are TCP-like in behavior (described further [here](https://github.com/RSocket/reactivesocket/blob/master/Protocol.md#transport-protocol)).

#### TCP & QUIC

No framing or application semantics. Must provide an application protocol.

#### WebSockets

No application semantics, just framing. Must provide an application protocol.

#### HTTP/1.1 & HTTP/2

HTTP provides barely sufficient raw capabilities for application protocols to be built with, but an application protocol still needs to be defined on top of it. It is insufficient in defining application semantics. ([GRPC from Google](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) is an example of a protocol being built on top of HTTP/2 to add these type of semantics).

These limited application semantics generally requires an application protocol to define things such as:
  - Use of GET, POST or PUT for request
  - Use of Normal, Chunked or SSE for response
  - MimeType of payload
  - error messaging along with standard status codes
  - how client should behave with status codes
  - Use of SSE as persistent channel from server to client to allow server to make requests to client

There is no defined mechanism for flow control from responder (typically server) to requestor (typically client). HTTP/2 does flow control at the byte level, not the application level. The mechanisms for communicating requestor (typically server) availability (such as failing a request) are inefficient and painful. It does not support interaction models such as fire-and-forget, and streaming models are inefficient (chunked encoding or SSE, which is ASCII based).

Despite its ubiquity, REST alone is insufficient and inappropriate for defining application semantics. 

What about HTTP/2 though? Doesn't it resolve the HTTP/1 issues and address the motivations of RSocket?

Unfortunately, no. HTTP/2 is MUCH better for browsers and request/response document transfer, but does not expose the desired behaviors and interaction models for applications as described earlier in this document.

Here are some quotes from the HTTP/2 [spec](https://http2.github.io/http2-spec/) and [FAQ](https://http2.github.io/faq/) that are useful to provide context on what HTTP/2 was targeting:

> “HTTP's existing semantics remain unchanged.”

> “… from the application perspective, the features of the protocol are largely unchanged …”

> "This effort was chartered to work on a revision of the wire protocol – i.e., how HTTP headers, methods, etc. are put “onto the wire”, not change HTTP’s semantics."

Additionally, "push promises" are focused on filling browser caches for standard web browsing behavior:

> “Pushed responses are always associated with an explicit request from the client.”

This means we still need SSE or WebSockets (and SSE is a text protocol so requires Base64 encoding to UTF-8) for push.

HTTP/2 was meant as a better HTTP/1.1, primarily for document retrieval in browsers for websites. We can do better than HTTP/2 for applications. 
