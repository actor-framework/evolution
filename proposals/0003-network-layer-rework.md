# Rework Network Layer

* Proposal: [CE-0003](0003-netowrk-layer-rework.md)
* Author: [Joseph Noir](https://github.com/josephnoir)

## Introduction

Distributed systems are a key environment of the actor frameworks. From the beginning CAF had a strong coupling to TCP. As a central entity the BASP broker is a bottle neck for communication and hinders scalability. While support for UDP was added in recent years, extending the network layer to handle new protocols requires adjustemnts throughout the stack.

## Motivation

The existing network layer has a few problems:

* It cannot use the existing bandwidth.
* Extending the network layer with new protocols or functionality requires changes throughout the whole stack.
* No support for multi-threading.

This proposal introduces a new design for the network layer. Reworked event handlers can be configured with a transport policy for communication and a protocol policy to use on top of the protocol. For the standard actor-to-actor communication this could be TCP as a transport with BASP as a protocol on top. Serialization is no longer handled by a central entity but distributed among actors that want to send data over the network.

### Multiplexer

Not too many changes here. The most important part is getting this to run with multiple threads. Since we have bottlenecks elsewhere, I'd put bigger changes off until the rest works. The `default_multiplexer` hosts a lot of socket-specific functionality at the moment. Part of this will be useful in for this desgin and should be clean up accordingly.

Some links I found about multithreading:

* [epoll](https://stackoverflow.com/questions/14584833/multithreaded-epoll), [more epoll](https://stackoverflow.com/questions/7058737/is-epoll-thread-safe), [hmmm](https://www.reddit.com/r/C_Programming/comments/7bnscf/multithreaded_epoll_server_design/)
* [kqueue](https://lists.freebsd.org/pipermail/freebsd-hackers/2004-July/007655.html), [more kqueue](https://stackoverflow.com/questions/25228938/tcp-server-workers-with-kqueue)

### Event Handler

This class existed before. It runs in the multiplexer thread and is called when a network event happens.


```
using network_queue = std::deque<std::vector<char>,caf::mailbox_element>;
using timeout_queue = std::deque<caf::atom, uint64_t>;

struct network_handler : public event_handler {
  // Should handle mutliple queues for socket-multiplexing protocols, see below.
  network_queue data;
  // Has priority, allows us to use the thread_safe_actor_clock for timeout handling.
  timeout_queue tos;

  /// Called by the protocol to create a queue for a new endpoint.
  caf::expected<network_queue> new_endpoint(...);

  protocol proto;
  transport trans;
};
```

*Challenges:*

* How does a timeout wake up the event handler?
* I don't want to introduce more queues, but let's consider sending an ack after a message was received. Where is that ACK message placed? At the back of `data`?

#### Protocol

```
struct protocol {
  // Called directly before sending data. Can write haders, set timeouts, ...
  virtual void prepare(std::vector<char> payload, caf::mailbox_element elem) = 0;
  // Called after data is received, can do things ...
  virtual void process()
  // Timeouts happend.
  virutal void handle_timeout(caf::atom, uint64_t);
};
```


#### Transport

```
struct transport {
  // Read data from the network.
  virtual io::network::rw_state caf::error read_some(...) = 0;

  // Write data to the network.
  virtual io::network::rw_state write_some(...) = 0;

  virtual expected<io::network::native_socket>
  connect(const std::string&, uint16_t,
          optional<io::network::protocol::network> = none) = 0;

  virtual void shutdown() = 0;

  virtual void configure_read(io::receive_policy::config);
};
```

*Challenges:*

* Socket-multiplexing protocols need some identifier or address as part of the read and write calls. For UDP this would be the endpoint information. Maybe this is a template class with an `Endpoint` type that is passed in and if the type if `None` with enable functions only expect the data?
* How is `configure_read` called? I think the read configuration depends on the `protocol`.

### Proxy

Benchmarks showed that the BASP broker spends a lot of time with message seriallization. To remove this bottleneck serialization will be hanlded by the proxy before handing the message off to the network layer.

```
struct serializing_proxy : public actor_proxy {
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

* If serialization remains as to be the limiting factor for network communication we should examine the possibility to slim down the serialization effort or look into a parallelization similar to the ongoin/recent [deserialization rework](https://github.com/actor-framework/actor-framework/tree/topic/basp-worker).


### Socker-per-connection protocols

scribe / acceptor separation


### Single-socket-multiplexed protocols

client / server separation


## Proposed Solution

A detailed explanation of the proposal.

## Impact on Existing Code

What changes to the existing code base are required?

## Alternatives

What are the alternatives and why not pick one of them?
