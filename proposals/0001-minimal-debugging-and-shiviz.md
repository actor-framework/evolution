# Minimal Support for Postmortem Debugging and ShiViz

* Proposal: [CE-0001](0001-minimal-debugging-and-shiviz-support.md)
* Author: [Dominik Charousset](https://github.com/neverlord)

## Introduction

Tools such as ShiViz are crucial for developers to understand complex errors in
distributed CAF applications.

## Motivation

Debugging CAF applications is currently not well supported. CAF does provide
logging in a Log4J-like format. However, the logfiles fail to capture
happens-before relations that are crucial in understanding program execution.
To this end, the message layer of CAF needs to log send and process events in a
way that allow tools to unambiguously reproduce happens-before relations. This
then allows us to implement a simple tool to enhance existing logfiles with
vector timestamps as well as to merge individual logs from distributed runs.
Finally, developers can visualize the enhanced and merged logfiles with tools
such as ShiViz to get comprehensible and interactive output.

## Proposed Solution

Standardizing CAF events in the DEBUG category can greatly aid developers in
understanding logs. Only grepping these events reveals the flow of messages and
lifetimes of actors. Further, tools can automatically deduce happens-before
relations from these standardized events by matching `SEND` to `RECEIVE`
events.

This enables us to scan log files for matching entries in order to establish a
happen-before relation. The relation is then used by a new tool (e.g.
"CafVector") to automatically enhance the log files with vector timestamps for
ShiViz.

In detail, I propose the following standard format for message and lifetime
related events:

- `SPAWN ; ID = <id> ; TYPE = <type> ; ARGS = <args>`
  + create a new actor
  + `id`: assigned actor_id
  + `type`: actor type name or "fun" when spawning a function-based actor
  + `args`: initialization arguments
- `SEND ; TO = <dest> ; FROM = <src> ; STAGES = <stages> ; CONTENT = <content>`
  + enqueue a message into a mailbox
  + `dest`: actor address of the receiver
  + `src`: actor address of the sender 
  + `stages`: vector of actor addresses denoting the processing pipeline
  + `content`: payload as type-erased tuple
- `REJECT`
  + enqueueing failed because the mailbox is closed (usually indicating that
    the receiver is already down)
- `ACCEPT`
  + enqueueing a message into a mailbox was successful
- `RECEIVE ; FROM = <src> ; STAGES = <stages> ; CONTENT = <content>`
  + dequeue a message from the mailbox
  + `src`: actor address of the sender
  + `stages`: vector of actor addresses denoting the processing pipeline
  + `content`: payload as type-erased tuple
- `DROP`
  + discard current message because it is unexected or receiver terminates
- `SKIP`
  + put current message back into the mailbox for later retrieval
- `FINALIZE`
  + start sending response messages etc. after handling a message successfully
- `TERMINATE ; REASON = <reason>`
  + stop an actor
  + `reason`: exit reason as observed by other actors

Every `RECEIVE` event is followed by exactly one `DROP`, `SKIP`, or `FINALIZE`
event. A `SKIP` event causes a `RECEIVE` event for the same message again
later.

It is worth mentioning that the `SPAWN` event is logged from the creator of an
actor *before* calling constructors. This allows developers (and tools) to
extract spawned-by relations for all actors in the system.

## Impact on Existing Code

New macros for logging the events uniformly, e.g.: `CAF_LOG_SEND_EVENT`,
`CAF_LOG_RECEIVE_EVENT`, and `CAF_LOG_DROP_EVENT`, and using the macros in all
actor implementations appropriately.

## Alternatives

Assigning vector timestamps to log entries could be done "fuzzily" by trying to
parse the current log output. However, there is no uniform way in which actors
log these events at the moment. This would make parsing code unnecessarily
complex and tedious. Further, the proposed solution only requires DEBUG log
level, whereas currently many enqueue implementations only log incoming
messages at TRACE level. The new standardization of CAF events also makes it
much easier for humans to grasp complex system behavior.
