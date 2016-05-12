# Seq

This document gives a gentle overview of the possible design solutions to the common problem of generating sequential / monotonically increasing IDs in a distributed system.  
Specifically, it focuses on maximizing performances and guaranteeing a fair distribution of the workload between nodes, as the size of the cluster increases.

## Possible designs

### Consensus protocols

Perhaps the most obvious and straightforward way of solving this problem is to implement a distributed locking mechanism upon a consensus protocol such as [Raft](https://raft.github.io/).

In fact, several tools such as [Consul](https://www.consul.io/) or [ZooKeeper](https://zookeeper.apache.org/) already exist out there, and provide all the necessary abstractions for emulating atomic integers across the network; out of the box.

Using these capabilities, it is quite straightforward to expose a `get-and-incr` atomic endpoint for clients to query.

**pros**:

- Strong consistency & sequentiality guarantees  
  Using a quorum, the system can A) guarantee the sequentiality of the IDs returned over time, and B) assure that there is no "holes" in the sequence.
- Good fault-tolerance guarantees  
  The system can and will stay available as long as 2N+1 nodes are still available.

**cons**:

- Poor performance  
  Since every operation requires communication between nodes, most of the time is spent in costly network IO.
- Uneven workload distribution  
  Due to the nature of the Leader/Follower model; a single node, the leader, is charged of handling all of the incoming traffic (e.g. serialization/deserialization of RPC requests).

**Further reading**:

[thesecretlivesofdata](http://thesecretlivesofdata.com/raft/) offers a great visual introduction to the inner workings of the Raft protocol.

### Consensus protocols + client-side caching

A simple enhancement to the *consensus protocols* approach is to batch the fetching of IDs: instead of returning a single ID every time a client queries the service, the system will allocate a *range* of available IDs and return this range to the client.

This has the obvious advantage of greatly reducing the number of network calls necessary to obtain N IDs (depending on the size of the ranges used); thus fixing the performance issues of the basic approach.  
However, this requires the client to maintain a local cache for storing its allocated range, which comes with various drawbacks:  
- Adds extra application-logic to the client side
- Can result in "holes" or "gaps" in the sequence if the client loses its cache for any reason (e.g. crash)  
  (Obviously the client could make sure to persist its cache to avoid this issue; but then you're juste adding even more client-side complexity.)

Overall, the performance boost is certainly worth the extra cost in complexity if the potential discontinuity of the sequence is not considered an issue at the application level.
