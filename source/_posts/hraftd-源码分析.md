---
title: hraftd 源码分析
date: 2018-09-13 15:08:15
tags: raft hraftd raft源码 go
---

[https://github.com/otoolep/hraftd](https://github.com/otoolep/hraftd/tree/353050de98a8a0366a6af80da347c883bb61fc34)

本文通过分析`hraftd`的源码，了解如何使用 raft库 `hashicorp/raft`。

为后面分析 `hashicorp/raft` 做准备。

## 简介

raft 是一个分布式一致性协议，它的目的是保持集群中节点状态的一致性，即使部分节点失败或者网络分区。当构建容错系统时，分布式一致性是一个基础的概念。

`hraftd` 是使用 `hashicorp/raft` 的一个例子，可以帮助我们学习raft。

但是它有很多问题并没有解决
- 当客户端访问一个非leader节点时，直接返回了失败。
	- 可以返回重定向
	- 可以节点内部代理
- 现在是3个节点提供读，所以可能读到脏数据
- 即使一个节点提供读，但是网络分区的情况下，仍可能读到脏数据

具体的使用可以参考 github的 README。

## 源码分析

hraftd 的结构如下

![xx](./hraftd_arch.png)

store 由以下几部分组成
- `raft.FSM` 负责存储业务状态，即http请求的kv都是存储在这里的
- `raft.LogStore` 负责存储raft协议内部的日志。raft通过日志来同步状态机
- `raft.StableStore` 负责存储raft协议内部的一些持久化数据
- `raft.SnapshotStore` 负责存储FSM的快照，以压缩LogStore
- `raft.Tranport` 负责raft nodes之间的网络通讯

下面是 `hraftd` 使用 `hashicorp/raft` 代码摘录。
```go
// Instantiate the Raft systems.
ra, err := raft.NewRaft(config, (*fsm)(s), logStore, stableStore, snapshots, transport)
if err != nil {
	return fmt.Errorf("new raft: %s", err)
}
```

### `raft.FSM`

```go
// FSM provides an interface that can be implemented by
// clients to make use of the replicated log.
type FSM interface {
	// Apply log is invoked once a log entry is committed.
	// It returns a value which will be made available in the
	// ApplyFuture returned by Raft.Apply method if that
	// method was called on the same Raft node as the FSM.
	Apply(*Log) interface{}

	// Snapshot is used to support log compaction. This call should
	// return an FSMSnapshot which can be used to save a point-in-time
	// snapshot of the FSM. Apply and Snapshot are not called in multiple
	// threads, but Apply will be called concurrently with Persist. This means
	// the FSM should be implemented in a fashion that allows for concurrent
	// updates while a snapshot is happening.
	Snapshot() (FSMSnapshot, error)

	// Restore is used to restore an FSM from a snapshot. It is not called
	// concurrently with any other command. The FSM must discard all previous
	// state.
	Restore(io.ReadCloser) error
}

// FSMSnapshot is returned by an FSM in response to a Snapshot
// It must be safe to invoke FSMSnapshot methods with concurrent
// calls to Apply.
type FSMSnapshot interface {
	// Persist should dump all necessary state to the WriteCloser 'sink',
	// and call sink.Close() when finished or call sink.Cancel() on error.
	Persist(sink SnapshotSink) error

	// Release is invoked when we are finished with the snapshot.
	Release()
}

// Log entries are replicated to all members of the Raft cluster
// and form the heart of the replicated state machine.
type Log struct {
	// Index holds the index of the log entry.
	Index uint64

	// Term holds the election term of the log entry.
	Term uint64

	// Type holds the type of the log entry.
	Type LogType

	// Data holds the log entry's type-specific data.
	Data []byte
}
```

raft 负责同步日志，存储日志，压缩日志。但是实际状态的存储，需要raft使用者自己实现。`FSM`接口 是 `raft -> fsm instance` （数据从 raft 流向 fsm instance）。

### `raft.LogStore`

```go
// LogStore is used to provide an interface for storing
// and retrieving logs in a durable fashion.
type LogStore interface {
	// FirstIndex returns the first index written. 0 for no entries.
	FirstIndex() (uint64, error)

	// LastIndex returns the last index written. 0 for no entries.
	LastIndex() (uint64, error)

	// GetLog gets a log entry at a given index.
	GetLog(index uint64, log *Log) error

	// StoreLog stores a log entry.
	StoreLog(log *Log) error

	// StoreLogs stores multiple log entries.
	StoreLogs(logs []*Log) error

	// DeleteRange deletes a range of log entries. The range is inclusive.
	DeleteRange(min, max uint64) error
}
```

LogStore 是由raft使用者实现，用于 raft log 持久化。

### `raft.StableStore`

```go
// StableStore is used to provide stable storage
// of key configurations to ensure safety.
type StableStore interface {
	Set(key []byte, val []byte) error

	// Get returns the value for key, or an empty byte slice if key was not found.
	Get(key []byte) ([]byte, error)

	SetUint64(key []byte, val uint64) error

	// GetUint64 returns the uint64 value for key, or 0 if key was not found.
	GetUint64(key []byte) (uint64, error)
}

var (
	keyCurrentTerm  = []byte("CurrentTerm")
	keyLastVoteTerm = []byte("LastVoteTerm")
	keyLastVoteCand = []byte("LastVoteCand")
)
```

raft 协议实现上需要一些持久化的状态。

### `raft.SnapshotStore`

```go
// SnapshotStore interface is used to allow for flexible implementations
// of snapshot storage and retrieval. For example, a client could implement
// a shared state store such as S3, allowing new nodes to restore snapshots
// without streaming from the leader.
type SnapshotStore interface {
	// Create is used to begin a snapshot at a given index and term, and with
	// the given committed configuration. The version parameter controls
	// which snapshot version to create.
	Create(version SnapshotVersion, index, term uint64, configuration Configuration,
		configurationIndex uint64, trans Transport) (SnapshotSink, error)

	// List is used to list the available snapshots in the store.
	// It should return then in descending order, with the highest index first.
	List() ([]*SnapshotMeta, error)

	// Open takes a snapshot ID and provides a ReadCloser. Once close is
	// called it is assumed the snapshot is no longer needed.
	Open(id string) (*SnapshotMeta, io.ReadCloser, error)
}
```

日志的快照存储

### `raft.Tranport`

```go
// Transport provides an interface for network transports
// to allow Raft to communicate with other nodes.
type Transport interface {
	// Consumer returns a channel that can be used to
	// consume and respond to RPC requests.
	Consumer() <-chan RPC

	// LocalAddr is used to return our local address to distinguish from our peers.
	LocalAddr() ServerAddress

	// AppendEntriesPipeline returns an interface that can be used to pipeline
	// AppendEntries requests.
	AppendEntriesPipeline(id ServerID, target ServerAddress) (AppendPipeline, error)

	// AppendEntries sends the appropriate RPC to the target node.
	AppendEntries(id ServerID, target ServerAddress, args *AppendEntriesRequest, resp *AppendEntriesResponse) error

	// RequestVote sends the appropriate RPC to the target node.
	RequestVote(id ServerID, target ServerAddress, args *RequestVoteRequest, resp *RequestVoteResponse) error

	// InstallSnapshot is used to push a snapshot down to a follower. The data is read from
	// the ReadCloser and streamed to the client.
	InstallSnapshot(id ServerID, target ServerAddress, args *InstallSnapshotRequest, resp *InstallSnapshotResponse, data io.Reader) error

	// EncodePeer is used to serialize a peer's address.
	EncodePeer(id ServerID, addr ServerAddress) []byte

	// DecodePeer is used to deserialize a peer's address.
	DecodePeer([]byte) ServerAddress

	// SetHeartbeatHandler is used to setup a heartbeat handler
	// as a fast-pass. This is to avoid head-of-line blocking from
	// disk IO. If a Transport does not support this, it can simply
	// ignore the call, and push the heartbeat onto the Consumer channel.
	SetHeartbeatHandler(cb func(rpc RPC))
}
```

# Note

源码引用的`hashicorp/raft`
[https://github.com/hashicorp/raft](https://github.com/hashicorp/raft/tree/82694fb663be3ffa7769961ee9a65e4c39ebbf2c)
