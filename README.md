_For background on this project check out this [blog post](http://www.philipotoole.com/building-a-distributed-key-value-store-using-raft/)._

hraftd [![Build Status](https://travis-ci.org/dustin-decker/hraftd.svg?branch=master)](https://travis-ci.org/dustin-decker/hraftd) [![GoDoc](https://godoc.org/github.com/dustin-decker/hraftd?status.png)](https://godoc.org/github.com/dustin-decker/hraftd) [![Go Report Card](https://goreportcard.com/badge/github.com/dustin-decker/hraftd)](https://goreportcard.com/report/github.com/otoolep/hraftd)
======

hraftd is a reference example use of the [Hashicorp Raft implementation v1.0](https://github.com/hashicorp/raft). [Raft](https://raft.github.io/) is a _distributed consensus protocol_, meaning its purpose is to ensure that a set of nodes -- a cluster -- agree on the state of some arbitrary state machine, even when nodes are vulnerable to failure and network partitions. Distributed consensus is a fundamental concept when it comes to building fault-tolerant systems.

A simple example system like hraftd makes it easy to study the Raft consensus protocol in general, and Hashicorp's Raft implementation in particular. It can be run on Linux, OSX, and Windows.

## Reading and writing keys

The reference implementation is a very simple in-memory key-value store. You can set a key by sending a request to the HTTP bind address (which defaults to `localhost:11000`):
```bash
curl -XPOST localhost:11000/key -d '{"foo": "bar"}'
```

You can read the value for a key like so:
```bash
curl -XGET localhost:11000/key/foo
```

## Running hraftd
*Building hraftd requires Go 1.9 or later. [gvm](https://github.com/moovweb/gvm) is a great tool for installing and managing your versions of Go.*

Starting and running a hraftd cluster is easy. Download hraftd like so:
```bash
mkdir hraftd
cd hraftd/
export GOPATH=$PWD
go get github.com/otoolep/hraftd
```

Run your first hraftd node like so:
```bash
$GOPATH/bin/hraftd -id node0 ~/node0
```

You can now set a key and read its value back:
```bash
curl -XPOST localhost:11000/key -d '{"user1": "batman"}'
curl -XGET localhost:11000/key/user1
```

### Bring up a cluster
_A walkthrough of setting up a more realistic cluster is [here](https://github.com/dustin-decker/hraftd/blob/master/CLUSTERING.md)._

Let's bring up 2 more nodes, so we have a 3-node cluster. That way we can tolerate the failure of 1 node:
```bash
$GOPATH/bin/hraftd -id node1 -haddr :11001 -raddr :12001 -join :11000 ~/node1
$GOPATH/bin/hraftd -id node2 -haddr :11002 -raddr :12002 -join :11000 ~/node2
```
_This example shows each hraftd node running on the same host, so each node must listen on different ports. This would not be necessary if each node ran on a different host._

This tells each new node to join the existing node. Once joined, each node now knows about the key:
```bash
curl -XGET localhost:11000/key/user1
curl -XGET localhost:11001/key/user1
curl -XGET localhost:11002/key/user1
```

Furthermore you can add a second key:
```bash
curl -XPOST localhost:11000/key -d '{"user2": "robin"}'
```

Confirm that the new key has been set like so:
```bash
curl -XGET localhost:11000/key/user2
curl -XGET localhost:11001/key/user2
curl -XGET localhost:11002/key/user2
```

#### Stale reads
Because any node will answer a GET request, and nodes may "fall behind" updates, stale reads are possible. Again, hraftd is a simple program, for the purpose of demonstrating a distributed key-value store. If you are particularly interested in learning more about issue, you should check out [rqlite](https://github.com/rqlite/rqlite). rqlite allows the client to control [read consistency](https://github.com/rqlite/rqlite/blob/master/doc/CONSISTENCY.md), allowing the client to trade off read-responsiveness and correctness.

Read-consistency support could be ported to hraftd if necessary.

### Tolerating failure
Kill the leader process and watch one of the other nodes be elected leader. The keys are still available for query on the other nodes, and you can set keys on the new leader. Furthermore, when the first node is restarted, it will rejoin the cluster and learn about any updates that occurred while it was down.

A 3-node cluster can tolerate the failure of a single node, but a 5-node cluster can tolerate the failure of two nodes. But 5-node clusters require that the leader contact a larger number of nodes before any change e.g. setting a key's value, can be considered committed.

### Leader-forwarding
Automatically forwarding requests to set keys to the current leader is not implemented. The client must always send requests to change a key to the leader or an error will be returned.

## Production use of Raft
For a production-grade example of using Hashicorp's Raft implementation, to replicate a SQLite database, check out [rqlite](https://github.com/rqlite/rqlite).
