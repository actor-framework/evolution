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


### Socket Manager

This class existed in form of the `event_handler` before. Since it takes ownership of the socket it handles it is renamed to have the name better fit its purpose. Socket managers run in the multiplexer thread and are scheduled when an event on their socket happens. 

Each socket manager has a `data` queue that stores data to send via its socket. Each queue element stores a buffer with the serialized payload and the related mailbox element. Storing the serialized payload allows us to perform the serialization in a separate thread. Since the mailbox element itself contains information such as the destination it might be important for building the complete packet or addressing the right endpoint.

The second queue handles timeout messages which allows us to use the existing `thread_safe_actor_clock` for timeouts. This queue has a higher priority than the `data` queue. Timeouts are identified by an atom and an integer id.

The socket manager (or its derived classes) have two policies that represent how communication with the endpoint works (communication policy) and which transport is used (transport policy).

```cpp
using buffer = std::vector<char>;
using data_queue_ptr
  = intrusive_ptr<std::some_queue<buffer, mailbox_element>>;
using timeout_queue_ptr
  = intrusive_ptr<std::some_queue<atom, uint64_t>>;

struct socket_manager {

  error handle_read_event();

  error handle_write_event();

  intrusive_ptr<data_queue> queue();

  intrusive_ptr<timeout_queue> timeouts();

  void register_read();

  void register_write();

  socket handler();

  multiplexer& mpx();

  void stop_reading();

  communication_policy cp;

  transport_policy tp;
};
```

### Communication Policy

This policy builds packets / buffers that can be send over the network from the serialized payload and the mailbox element in the data queue of the socket manager. Examples are an application layer protocol or augmentations to the underlying transport protocol. When instantiating a socket manager the policy can be chosen to fit the use case. For messaging between CAF nodes this is BASP, but communication policies could implement protocols such as HTTP or simply add ordering, etc. Policies might be stackable to allow composition of different building blocks, such as reliable + ordered UDP.

```cpp
struct communication_policy {
  // Called directly before sending data. Can write headers, set timeouts, ...
  // payload:  Serialized payload.
  // elem:     Mailbox element, has node information, etc.
  // tp:       Transport policy -- not sure if necessary, but if the policy
  //           wants to slice data it would be easier to append packets to an
  //           internal buffer than returning a vector or list ...
  // m:        Socket manager might be needed to set timeouts?
  virtual void prepare(std::vector<char> payload,
                       caf::mailbox_element elem,
                       transport_policy& tp,
                       socket_manager& m) = 0;

  // Called after data is received. Same is true as above, needs the transport
  // to send messages in response to received packets (e.g., ACK messages).
  virtual void process(std::vector<char> payload,
                       transport_policy& tp,
                       socket_manager& m) = 0;

  // Timeouts forwarded by the socket manager.
  // atm:      Policies might be stacked and this atom identifies the
  //           responsible policy
  // id:       Differentiate timeouts per policy, could be an id for
  //           retransmits.
  // m:        Option to set more timeouts.
  virtual void handle_timeout(caf::atom atm,
                              uint64_t id,
                              transport_policy& tp,
                              socket_manager& m);

  // Wrapped stack of policies.
  internal_policy pol;
};
```


### Transport Policy

The transport policy abstracts over a transport protocol, i.e, sending and receiving data as well as creating sockets and establishing communication with remote endpoints.

To avoid transport-specific interfaces the policy is called with the payload and mailbox element and can then use the communication policy to create packets to send to the network before doing so. This gives the policy the opportunity to resolve the node ID to additional information that might be required to send data such as the IP + port tuple required for (non-connected) UDP.

```cpp
struct transport_policy {
  // Write event delegated from a socket manager.
  // payload:   Serialized data to send.
  // elem:      Meta information for the payload.
  // cp:        Policy to build packets from the payload.
  // sm:        Manager that delegated the event.
  error write_event(buffer& payload,
                    mailbox_element& elem,
                    communication_policy& cp,
                    socket_manager& sm);

  // Read event from the multiplexer
  // 
  error read_event(communication_policy& cp,
                   socket_manager& sm);

  error accept();

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

  // Potentially (!) an internal queue with data to send out. This might
  // be necessary to give the communication policy a place to put its prepare
  // packets. With slicing the policy might create multiple packets and needs
  // to put them somewhere.
  packet_buffer sending;
};
```


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


### Overview

#### Kernel-multiplexed Protocols

doorman + scribe, scribe is full socket manager

##### Accepting

##### Sending

This starts when a proxy serializes the payload from a mailbox element and appends both to the data queue of its socket manager. If the manger is not registered for writing this happens via a callback when data is enqueued.

For sending, the transport policy gets called with the payload, the mailbox element, the communication policy and the socket manager. First it forwards

##### Receiving


#### User-space-multiplexed Protocols

doorman + scribe, 



## Impact on Existing Code

What changes to the existing code base are required?


## Alternatives

What are the alternatives and why not pick one of them?
