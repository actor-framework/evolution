# Minimal Support for Postmortem Debugging and ShiViz Support

* Proposal: [SE-0001](0001-minimal-debugging-and-shiviz-support.md)
* Author: [Dominik Charousset](https://github.com/neverlord)
* Status: Accepted

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

All actors, e.g., event-based and blocking, log the following three events
using the DEBUG category in the same way: 1) SEND: enqueueing a message to a
mailbox, 2) RECEIVE: start processing a message, and 3) DROP: discarding a
message in case the receiver is already down. These entries should have the
form `<event> ; FROM = <sender> ; TO = <receiver> ; STAGES = <stages> ; CONTENT
= <content>`.

This enables us to scan log files for matching entries in order to establish a
happen-before relation. The relation is then be used by a new tool (e.g.
"CafVector") to automatically enhance the log files with vector timestamps for
ShiViz.

## Impact on Existing Code

* Three new macros for logging the events uniformly: `CAF_LOG_SEND_EVENT`,
  `CAF_LOG_RECEIVE_EVENT`, and `CAF_LOG_DROP_EVENT`.
* Using `CAF_LOG_SEND_EVENT` in all implementations of the virtual member
  function `enqueue`.
* Using `CAF_LOG_DROP_EVENT` in `enqueue` in case message delivery fails due to
  a closed mailbox (i.e. the receiving actor is already down). Further, the
  mailbox needs to call the macro on each discarded message after closing.
* Using `CAF_LOG_RECEIVE_EVENT` in all actor types before calling into
  user-defined message handlers.

## Alternatives

Assigning vector timestamps to log entries could be done "fuzzily" by trying to
parse the current log output. However, there is no uniform way in which actors
log these events at the moment. This would make parsing code unnecessarily
complex and tedious. Further, the proposed solution only requires DEBUG log
level, whereas currently many enqueue implementations only log incoming
messages at TRACE level.
