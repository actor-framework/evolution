# Rework Network Layer

* Proposal: [CE-0003](0003-netowrk-layer-rework.md)
* Author: [Joseph Noir](https://github.com/josephnoir)


## Introduction

Distributed systems are a key environment of the actor frameworks. From the beginning CAF had a strong coupling to TCP. As a central entity the BASP broker is a bottle neck for communication and hinders scalability. While support for UDP was added in recent years, extending the network layer to handle new protocols requires adjustments throughout the stack.


## Motivation

The existing network layer has a few problems:

* It cannot fully use the bandwidth of standard network interfaces.
* Extending the network layer with new protocols or functionality requires changes throughout the whole stack.
* No multi-threading support.

This proposal introduces a new design for the network layer. Reworked event handlers can be configured with a transport policy for communication and a protocol policy to use on top of the protocol. For the standard actor-to-actor communication this could be TCP as a transport with BASP as a protocol on top. Serialization is no longer handled by a central entity but distributed among actors that want to send data over the network.


## Proposed Solution

A detailed explanation of the proposal.


### Multiplexer

The major new feature is multi-threading. Since we have bottlenecks elsewhere, I'd put as much of the multi-threading itself off until the basic concept works. Still, this aspect cannot be ignored as it has implications for the overall design. Foremost, thread safety has to be kept in mind when designing the interactions between proxies, event handlers, and the multiplexing thread.

Some links on multi-threaded multiplexing for later:

* [epoll](https://stackoverflow.com/questions/14584833/multithreaded-epoll), [more epoll](https://stackoverflow.com/questions/7058737/is-epoll-thread-safe), [hmmm](https://www.reddit.com/r/C_Programming/comments/7bnscf/multithreaded_epoll_server_design/)
* [kqueue](https://lists.freebsd.org/pipermail/freebsd-hackers/2004-July/007655.html), [more kqueue](https://stackoverflow.com/questions/25228938/tcp-server-workers-with-kqueue)

The `default_multiplexer` hosts a lot of socket-specific functionality at the moment. Part of this will be useful in for this design and should be cleaned up accordingly.


### Event Handler

This class existed before. It runs in the multiplexer thread and is called for network events on its socket. The new network handler has a `protocol` member that represents the protocol spoken over the transport. This could be BASP between CAF actor systems or ordering when UDP is used. The transport protocol for its communication is determined by the `transport` member, e.g., TCP, UDP, QUIC, etc.

Data to be written to the socket is stored in the `data` queue. Each element stores the buffer with the serialized payload and the mailbox element handed to the proxy. Data in the mailbox element might be important for the protocol. Thinking of BASP, the sender and receiver of the message are important to write the BASP header. This information is stored in the `mailbox_element`. This queue is fed from outside the multiplexer and requires a thread safe enqueue / dequeue -- note that only the multiplexer itself reads from the queue which allows for some optimization.

The second queue handles timeout messages. It is fed by the `thread_safe_actor_clock` which is used to set timeouts. These have priority over other messages. Timeouts are identified by an atom and an id. The event handler forwards atoms to the protocol which can use them for functionality such as retransmits.

Event handlers have a `reading` state to track if their socket is registered for reading at the multiplexer. While one of its queues has data the `event_handler` should be registered for reading.

```cpp
using buffer = std::vector<char>;
using network_queue = std::deque<buffer, caf::mailbox_element>;
using timeout_queue = std::deque<caf::atom, uint64_t>;

struct network_handler : public event_handler {
  // Should handle multiple queues for socket-multiplexing protocols,
  // see below.
  network_queue data;
  // Has priority, allows us to use the thread_safe_actor_clock for timeout
  // handling.
  timeout_queue tos;

  // Add a buffer to the data queue. Is thread safe.
  void enqueue(buffer, caf::mailbox_element);

  // Called by the protocol to create a queue for a new endpoint.
  caf::expected<network_queue> new_endpoint(...);

  protocol proto;
  transport trans;
};
```

Adding data to the queues is potentially a costly operation as it requires memory allocations. The queue itself is not the problem, but the buffers it stores might be. There are several approaches to avoid this. Reusing buffers is one. Writing a custom allocator that uses a memory pool is another one. There is also tcmalloc. This needs some thought.

*Queues:*

Protocols that are multiplexed over a single socket such as UDP or QUIC require a mapping of each packet to the endpoint it addresses. Let's call this info id. For UDP this would be host + port.

* Add a one queue per endpoint, queue knows its id.
  * Pro: Transport-independent queues.
  * Con: Overhead for queue management.
* A single queue with a filed for the transport-specific id. Each proxy knows the id for its endpoint and appends it with its payload.
  * Pro: Only one queue. No lookup since each packet knows its id.
  * Con: Queue type depends on the transport policy.
* A single queue, but the protocol keeps a mapping from the node id (available in the mailbox element) to the connection id.
  * Pro: Single queue with transport-independent type.
  * Con: Protocol has to do a hash lookup (which is often cheap).

*Challenges:*

* How does a timeout wake up the event handler? There is no socket event here that would do something like this ... It might be enough to simply register the event handler for write events when something is enqueued to any queue.
* I don't want to introduce more queues, but let's consider sending an ack after a message was received. Where is that ACK message placed? At the back of `data`?
* Do we want all event handlers to have the `<buffer, mailbox_element>` pair? This is very CAF message specific and might not be desirable for a broker-like feature we want to introduce later. Maybe this could be a template parameter?


*Alternatives:*

Data in the `data` queue is already the complete message. Before data is passed to the event handler it is already processed by the protocol which just gets a callback when a payload is send. Could work with something like an ID. This would make the interface a bit more generic and less specific to actor messages.


#### Transport

The transport policy abstracts send, receiving, and establishing connections (or communication) for a specific transport protocol. While the general functionality is smilar for many protocols, I'm not sure we can have a defined interface. A key difference between protocols that have one socket per connection (multiplexed by the kernel such as TCP) and protocols that require manual socket-multiplexing by the user (such as UDP or QUIC) is the requirement to have an endpoint defined in the read and write calls. For UDP this is the address of the remote endpoint (host + port) but could be anything. The general interface looks something like this:

```cpp
struct transport {
  // Read data from the network.
  virtual io::network::rw_state read_some(size_t& result,
                                          io::network::native_socket fd,
                                          void* buf, size_t buf_len) = 0;

  // Write data to the network.
  virtual io::network::rw_state  write_some(size_t& result,
                                            io::network::native_socket fd,
                                            const void* buf, size_t len) = 0;

  // Open a local endpoint to accept connections.
  virtual void <io::network::native_socket, uint16_t>
  open(uint16_t port, const char* addr, bool reuse = false,
       optional<io::network::protocol::network> = none) = 0;

  // Establish communication with a remote endpoint.
  virtual expected<io::network::native_socket>
  connect(const std::string&, uint16_t,
          optional<io::network::protocol::network> = none) = 0;

  // Useful for protocols that parse data from a stream.
  virtual void configure_read(io::receive_policy::config);
};
```

Ideas to address the additional parameter for socket-multiplexing protocols:

1) Add a template argument to choose the type of the identifier. UDP could set this to `ip_endpoint` and derive from transport. Not sure if `connect` would need to return such an identifier as well. I could imagine something like a connect for QUIC where the connection is established and returns the ID that is than used to address that endpoint.

```cpp
template <class Endpoint>
struct socket_multiplexing_transport {
  // Can be used by event handlers or protocols to store the identifiers.
  using endpoint_type = Endpoint;

  // Read data from the network.
  virtual io::network::rw_state
  read_some(size_t& result, io::network::native_socket fd,
            void* buf, size_t buf_len, endpoint_type& ep) = 0;

  // Write data to the network.
  virtual io::network::rw_state
  write_some(size_t& result, io::network::native_socket fd,
             const void* buf, size_t buf_len,
             const endpoint_type& ep) = 0;

  // Other functions.
};

struct ip_endpoint {
  caf::ipv4_address host;
  uint16_t port;
};

struct udp : socket_multiplexing_transport<ip_endpoint> {
  // Implementation.
};

```

2) Have an optional argument for the identifier. This might lead to problems in how to handle the identifier for protocols that don't need it. Writing a generic protocol will still not be easy and two different implementations might be required. This would give us a single parent with one interface.


#### Protocol

A protocol policy represents an application layer protocol or add augmentations to the underlying transport protocol. It defines how incoming data is processed and how a packet is augmented before it is sent. This might include adding new headers or setting timeouts. Protocols we should keep in mind during the design are BASP, ordering, reliability, and slicing.

The general idea is that protocols can be stacked and reused over different transport protocols. Having potentially different APIs for transport with kernel-multiplexed and socket-multiplexed protocols. This might be solved by having two protocol types that can handle the respective transport APIs and instantiate the same stack for both. Not sure if this is possible -- might depend on the choice we make for queue handling above.

```cpp
struct protocol {
  // Called directly before sending data. Can write headers, set timeouts, ...
  virtual void prepare(std::vector<char> payload,
                       caf::mailbox_element elem) = 0;

  // Called after data is received, can do things ...
  virtual void process()

  // Timeouts happened.
  virtual void handle_timeout(caf::atom, uint64_t);
};
```

*Challenges:*

* Socket-multiplexing protocols need some identifier or address as part of the read and write calls. For UDP this would be the endpoint information. Maybe this is a template class with an `Endpoint` type that is passed in and if the type if `None` with enable functions only expect the data?
* How is `configure_read` called? I think the read configuration depends on the `protocol`.


### Proxy

Benchmarks showed that the BASP broker spends a lot of time with message serialization. To remove this bottleneck serialization will be handled by the proxy before handing the message off to the network layer.

```cpp
struct serializing_proxy : public actor_proxy
  network_queue* queue;

  void enqueue(mailbox_element_ptr what, execution_unit* host) override {
    std::vector<char> buf;
    caf::binary_serializer bs(buf);
    what->get_msg().save(bs);
    queue->enqueue(std::move(elem), std::move(buf));
  }
};
```

*Future:*

* If serialization remains as to be the limiting factor for network communication we should examine the possibility to slim down the serialization effort or look into a parallelization similar to the ongoing/recent [deserialization rework](https://github.com/actor-framework/actor-framework/tree/topic/basp-worker).


### Kernel-multiplexed Protocols

These protocols can continue to work as before. A doorman listens on a socket to accept new connections. For each new connection it creates a scribe to handle communication. Each scribe is responsible for a single connection. Since scribes share no state with acceptors they can be created independently of each other. This way a scribe can be crated to connect to remote not as a client.

Since each socket is either listening or connected to an endpoint there is no additional information necessary to send or receive data. The `send` and `recv` calls are enough to handle this.

With the components described above, an implementation could work as follows: One `event_handler` acts as the acceptor. It only reacts to read events to accept new connections. The resulting socket is used to create a scribe (event handler + protocol + transport). I'm not sure which component should be responsible for this. Options:

* Each read event results in the creation of a new event handler that holds a new instance of the transport protocol policy and protocol policy. After the protocol policy is initialized it is has the option to react. (It could for example start a handshake.)
* The new socket is handed to the protocol policy which can choose how to react. BASP might create a new proxy with an event handler and the two policies. This could allows us to handle brokers as a policy that creates actors to handle the raw data or whatever is offered by other policies.
* ...

The client side instantiates the same scribe (event handler + transport + protocol).


### User-space-multiplexed Protocols

client / server separation


## Impact on Existing Code

What changes to the existing code base are required?


## Alternatives

What are the alternatives and why not pick one of them?
