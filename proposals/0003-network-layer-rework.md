# Rework Network Layer

* Proposal: [CE-0003](0003-netowrk-layer-rework.md)
* Author: [Joseph Noir](https://github.com/josephnoir)

## Introduction

Distributed systems are a key environment of the actor frameworks. From the beginning CAF had a strong coupling to TCP. As a central entity the BASP broker is a bottle neck for communication and hinders scalability. While support for UDP was added in recent years, extending the network layer to handle new protocols requires adjustments throughout the stack.

## Motivation

The existing network layer has a few problems:

* It cannot use the existing bandwidth.
* Extending the network layer with new protocols or functionality requires changes throughout the whole stack.
* No support for multi-threading.

This proposal introduces a new design for the network layer. Reworked event handlers can be configured with a transport policy for communication and a protocol policy to use on top of the protocol. For the standard actor-to-actor communication this could be TCP as a transport with BASP as a protocol on top. Serialization is no longer handled by a central entity but distributed among actors that want to send data over the network.

## Proposed Solution

A detailed explanation of the proposal.

### Multiplexer

The major new feature is multithreading. Since we have bottlenecks elsewhere, I'd put as much of the multi-threading itself off until the basic concept works. This cannot be ignored completely as it has implications for the overall design. Foremost, threadsafety has to be kept in mind when designing the interactions between proxies, event handlers, and the multiplexing thread. The `default_multiplexer` hosts a lot of socket-specific functionality at the moment. Part of this will be useful in for this design and should be clean up accordingly.

Some links I found about multithreading:

* [epoll](https://stackoverflow.com/questions/14584833/multithreaded-epoll), [more epoll](https://stackoverflow.com/questions/7058737/is-epoll-thread-safe), [hmmm](https://www.reddit.com/r/C_Programming/comments/7bnscf/multithreaded_epoll_server_design/)
* [kqueue](https://lists.freebsd.org/pipermail/freebsd-hackers/2004-July/007655.html), [more kqueue](https://stackoverflow.com/questions/25228938/tcp-server-workers-with-kqueue)

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
  // Should handle mutliple queues for socket-multiplexing protocols, see below.
  network_queue data;
  // Has priority, allows us to use the thread_safe_actor_clock for timeout handling.
  timeout_queue tos;

  // Add a buffer to the data queue. Is threadsafe.
  void enqueue(buffer, caf::mailbox_element);

  // Called by the protocol to create a queue for a new endpoint.
  caf::expected<network_queue> new_endpoint(...);

  protocol proto;
  transport trans;
};
```

Adding data to the queues is potentially a costly operation as it requires memory allocations. The queue itself is not the problem, but the buffers it stores might be. There are several approaches to avoid this. Reusing buffers is one. Writing a custom allocator that uses a memory pool is another one. There is also tmalloc. This needs some thought.

Protocols that are multiplexed over a single socket such as UDP or QUIC require more complex queue handling. An idea is to introduce one queue per endpoint. This requires a mechanism to dynamically manage queues in the event handler, including a mechanism to know which queues have data.

*Challenges:*

* How does a timeout wake up the event handler? There is no socket event here that would do something like this ... It might be enough to simply register the event handler for write events when something is enqueued to any queue.
* I don't want to introduce more queues, but let's consider sending an ack after a message was received. Where is that ACK message placed? At the back of `data`?
* Do we want all event handlers to have the `<buffer,mailbox_element>` pair? This is very CAF message specific and might not be desirable for a broker-like feature we want to introduce later. Maybe this could be a template parameter?


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

  // Some other functions.
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

A protocol policy represents an application layer protocol or add augmentations to the underlying transport protocol.

```cpp
struct protocol {
  // Called directly before sending data. Can write haders, set timeouts, ...
  virtual void prepare(std::vector<char> payload, caf::mailbox_element elem) = 0;

  // Called after data is received, can do things ...
  virtual void process()

  // Timeouts happend.
  virutal void handle_timeout(caf::atom, uint64_t);
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


### Socker-per-connection protocols

scribe / acceptor separation


### Single-socket-multiplexed protocols

client / server separation


## Impact on Existing Code

What changes to the existing code base are required?

## Alternatives

What are the alternatives and why not pick one of them?
