---
layout: post
title:  "Atomic Snapshots"
tag: [time]
date:   2019-05-05 09:00:00 +0200
---
In this installment I will discuss an interesting problem in distributed computing, which is how to implement *atomic snapshots*. I consider multiple solutions, and show how clocks, and in particular interval clocks, can be used in a particularly efficient solution.

I'll describe the problem in a simple setting: a partitioned key-value store (KVS). The KVS runs on many, think hundreds or thousands, of server computers (*servers*). The key space is partitioned across the servers, so that each server stores multiple keys, but each key is assigned to a single server. There are many clients, and they communicate with the servers to access and update the values. The assignment of keys to servers is fixed and known to everyone.

Computers (both clients and servers) are connected in a network, and can communicate only by sending and receiving messages.

![KVS organization](/assets/kvs_organization.png)

There are some obvious operations that clients can perform:
* write(key, value) -> () - associate value with key
* read(key) -> value - returns the value last associated with the key (NULL if no value has been previously associated)

The implementations of these operations are pretty straight forward; a client sends a request message to the server responsible for the key, and the server performs the operation locally and sends a response back to the client.

We now want to add a *multi_read* operation. The idea is that we want to read the values associated with any number of keys, atomically, as they were at some point in time during the operation. The operation is:

* multi_read(k1, k2, ...) -> (v1, v2, ...)

All operations are required to be *atomic*, meaning that it appears as if (i.e. all observations are consistent with that) each operation is performed instantaneously at some point in time between its invocation and response.
Note that it is possible that a multi_read operation reads all keys in the KVS, for example if a client wants to perform an aggregation across all keys. We refer to a multi_read that reads the values associated with all keys as an *atomic snapshot*.

## Solution 1: Locking

One possible way to implement the multi_read operation is to use locking. A multi_read is then performed in three steps:

1. The client sends requests to each involved server to block writes to the keys in question. A server receiving such a request records locally that it will not accept writes to those keys until the client releases the lock.
2. When the client has received information that all servers have locked their keys, the client sends read requests for each key to the servers. A server receiving such a read request responsd with the current values.
3. After the client has received values for each key, the client sends messages to the servers telling them to release the blocking for the keys.

In this way, the values received by the client are guaranteed to be part of an atomic snapshot, i.e. as they were at some point in time during the operation.

### Downside

The downside of solution 1 is that during the time that a multi_read is executing, writes cannot be performed on those keys. Considering that an atomic snapshot can access very many keys, a multi_read can run for quite a long time, and hence block writes for a long time. We would much prefer if writes could continue while a multi_read is running, and we therefore look to other solutions.

## Solution 2: Multiversioning with a timestamp server

Instead of a server storing a single value associated with a key, it can store multiple versions of the value, and each version has a timestamp that orders that version against other versions.

In this solution, the system has a central *timestamp server* that hands out monotonically increasing timestamps. The timestamp server has an integer variable whose value starts at zero, and then is incremented for each timestamp that is returned.

A client that wants to write a value does:

1. The client sends a request to the server whose key is written to, with the key and the value.
2. After receiving the write request from the client, the server communicates with the timestamp server to obtain a timestamp.
3. The server stores the value and its timestamp as a new version for the key, and sends a response to the client.

A client that wants to multi_read does:

1. Contact the timestamp server and obtain a timestamp.
2. Send requests to each server that holds keys involved in the multi_read operation; the request contains the timestamp obtained from the timestamp server.
3. The server is going to return the last version whose associated timestamp is smaller than the timestamp in the request. If there are writes in progress however, for which the server is waiting to obtain a timestamp, then the server has to wait until it has obtained that timestamp, so that it can determine if the write is before or after the read.

In this way, multi_read operations always return atomic snapshots, and writes are not blocked by long running multi_reads.

### Downsides

The server must store multiple versions of values for each key, but this is the price that must be paid for allowing writes to proceede concurrently with multi_reads.

A bigger problem is that the central timestamp server must be contacted once for each executed operation, no matter which server stores the value for that key. Considering that there are thousands of servers storing values, the timestamp server is likely to become a performance bottleneck. Furthermore, the timestamp server is a single point of failure, and if it becomes unavailable the entire KVS will stop being able to execute operations. We will see how these problems can be avoided.

## Solution 3: Multiversioning with interval clocks

Instead of having a central timestamp server, this solution uses synchronized clocks to obtain timestamps.

However, as we shall see, we cannot simply use "normal" synchronized clocks (i.e. clocks that are synchronized using the NTP protocol), and read those clocks to obtain timestamps instead of communicating with the timestamp server.

### Non-solution 3a: Using normal clocks synchronized with NTP

Assume that each computer has a clock that is synchronized using the Network Time Protocol (NTP). Then modify the algorithm used in solution 2 by, wherever a timestamp was obtained by communicating with the timestamp server, instead get the timestamp by reading the local synchronized clock.

It is easy to see that this solution will not work correctly. The reason is that the time returned by a normal clock has no notion of uncertainty, but as all clocks are subject to drift, the time returned by the clock could in fact be arbitrarily much before or after the actual time. A write operation could obtain a timestamp that is too large, and a later multi_read could obtain a timestamp that is too small, and therefore the multi_read will not include the written value even though that write was completed before the multi_read began.

### Solution 3b: Using interval clocks

What we need instead is *interval clocks*. An interval clock is different from a normal clock in that an interval clock returns an interval of time instead of a single time value. If an interval clock is read at time *t* then the interval returned is guaranteed to include *t*. An interval clock has two operations:

* upper() - returns the upper bound of the current interval, which is a time that is guaranteed to be greater than the time when the high() operation was invoked.
* lower() - returns the lower bound of the current interval, which is a time that is guaranteed to be smaller than the time when the low() operation completed.

Using these two interval clock operations, we make changes to the algorithm in solution 2 as follows. To perform a write, the following actions are performed:

1. The client sends a request with the key and value to the server.
2. When receiving the write request, the server executes upper() to obtain a timestamp that is guaranteed to be greater or equal to the current time. The timestamp must also be greater than all timestamps used before by that server, which can be guaranteed locally. The server then writes the value and its timestamp as a new version to the key, and sends a response to the client that includes the timestamp.
3. When the client receives the response, the client has to wait until executing lower() returns a time that is greater than the timestamp used for the write before the write is considered complete, which guarantees that the timestamp is a time in the past.

The following steps are taken to perform a multi_read:

1. The client executes upper() to obtain a timestamp that is greater then or equal to the current time. The client then sends requests containing the keys and the timestamp to each server that has keys that should be read.
2. A server that receives a multi_read request will respond with the value whose timestamp is smaller than or equal to the timestamp in the request. Before deciding what that value is, the server has to wait until upper() returns a value that is greater than the timestamp, which guarantees that no later write can obtain a greater timestamp.
3. When the client has received responses with values for each key, the client has to wait until executing lower() returns a time that is greater than the timestamp used for the multi_read. This guarantees that a later write cannot get a smaller timestamp.

We see that having interval clocks is essential to making this solution work. Solution 3b allows writes to proceed concurrently with long running multi_reads, and also avoids a server that must be contacted for each operation.

### Downside

One potential downside is that if an interval clock has a large uncertainty (it returns a wide interval) then the time it takes to execute an operation will be affected by that. In following blog entries I will describe the implementation of an interval clock synchronization algorithm, and show that the clock's uncertainty is typically on the order of the uncertainty in the message delay between servers, and the communication delay will therefore typically be greater then the interval clock uncertainty. This means that after the communication between client and server is performed, which needs to happen anyway, there is no need for further waiting.

## Summary

Three solutions to this problem has described, that incrementally removes restrictions in the previous solution. The restrictions that have been removed are:

1. Writes are blocked by long running multi_reads.
2. A central timestamp server needs to be contacted to execute each operation.

Having the ability to perform an atomic snapshot of the KVS can be useful to get a consistent view of the KVS as it was at some particular point in time. This could be used for aggregations or doing a census.

In later entries in this blog I will explain how an algorithm that synchronizes interval clocks works.
