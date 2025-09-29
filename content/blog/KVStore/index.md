+++
date = '2025-09-28T13:14:54-04:00'
draft = false
title = 'KVStore'
+++

Unveiling Paxos in Practice: How We Built a Distributed KV-Store

<!--more-->

# The Proposal

Distributed systems are the backbone of many modern applications, but managing consistency and fault tolerance in them is a challenge. In this post, I'll share our journey in developing a distributed Key-Value Store for a college course, using the complex but robust Paxos consensus algorithm and the Go language.

# What is a Distributed Consensus Algorithm?

To provide some context, let's first define a Distributed System.

Distributed systems are characterized by a set of independent computers that present themselves to the user as a single, coherent system. These systems are designed to achieve scalability, availability, and fault tolerance—essential characteristics in modern applications such as distributed databases, file systems, and cloud computing platforms.

With this in mind, one of the greatest difficulties in these distributed environments is coordination between the participating nodes of the application, especially when it's necessary to guarantee consistency between replicated states in scenarios where failures (unavailable nodes, network problems) can occur.

Faced with this problem, distributed consensus algorithms were developed, such as Paxos or Raft, which serve to coordinate nodes and ensure agreement on values or operations, even under adverse conditions.

## Paxos Algorithm

The Paxos algorithm, proposed by Leslie Lamport, is a robust solution to the consensus problem in asynchronous distributed systems with failures. It solves the consensus problem by ensuring that a set of nodes reaches agreement on a single value, even if up to half of the network nodes fail. To achieve this, Paxos operates in three main roles: Proposer (proposes values), Acceptor (accepts or rejects proposals), and Learner (learns the agreed value). It's a complex algorithm, but its reliability makes it ideal for systems that require greater consistency.

To understand how Paxos guarantees this consensus, let's detail its two main phases of operation:

- **Phase 1, preparation**:
    - **1a. Prepare**: A Proposer creates a message called "Prepare". This message is identified by a unique number **N**, which must be greater than any number previously used in a message. This message is sent to a Quorum of Acceptors.
    - **1b. Promise**: Upon receiving a message, an Acceptor checks if **N** is greater than any proposal number it has already promised to accept. If so, it responds with a Promise, committing not to accept proposals with a number lower than **N** and informing the highest-numbered proposal, **N'**, that it has already accepted, along with its value, **V'**. If **N** is not greater, it simply ignores or responds with denial.

- **Phase 2, acceptance**:
    - **2a. Accept**: If the Proposer receives Promises from a Quorum of Acceptors, it chooses a value **V** for its proposal. If any Acceptor informed having accepted a previous proposal, the Proposer must choose the value **V'** from the proposal with the highest number **N'**. Otherwise, it can choose a new value. Then, the Proposer sends an Accept(**N**, **V**) message to the Quorum of Acceptors.
    - **2b. Accepted**: Upon receiving an Accept(**N**, **V**) message, an Acceptor checks if it has already promised not to accept proposals with a number lower than **N**. If there is no such promise, it accepts the proposal, records the value **V** as accepted, and sends an Accepted(**N**, **V**) message to the Learners. If there's already a promise for a higher **N**, it simply ignores.

In Paxos, consensus is reached when the majority of Acceptors agree on the same proposal identifier number. Since each identifier is unique to a Proposer and a single value is associated with that identifier, ensuring that the majority accepts the same proposal ID implies they will agree on the same value.

However, this entire process is used to choose only one value. In a real scenario, we would have a continuous flow of agreed values acting as commands for a distributed state machine. However, if each command were the result of a Paxos instance, there would be significant network overhead, since for each command 2 messages would be sent to all network nodes (1a and 2a) and one node would have to receive 2 messages from each network node (1b and 2b).

To mitigate this, a crucial optimization is the election of a Leader, which simplifies Phase 1 for subsequent decisions, reducing network traffic.

# What is a KV-Store?

In summary, a Key-Value store is a simpler and more efficient type of database that stores data as key-value pairs. It has better read and write performance since it doesn't have a rigid schema or relationships between stored values. It's widely used in distributed systems and for caching.

# A KV-Store with Paxos

Right, but how do we combine the two things?

As mentioned earlier, in a real system using Paxos, we would have a continuous flow of agreed commands applied to a distributed state. Thus, we can understand the KV-store state as the result of applying in order all commands decided by Paxos rounds.

For this project, we defined the following commands:

- **SET** key value
- **DELETE** key

Example:

If we have the following 2 decided commands:

- **SET** key1 value1
- **SET** key2 value2

The state would be:
{ key1: value1, key2: value2 }

If we did **DELETE** key1, we would have: { key2: value2 }

# Implementation

For the implementation of this project, we used Golang as the main language, since it has very good support for concurrency and distributed communication, facilitating the development of necessary functionalities. For communication between nodes, we used gRPC.

## gRPC and protobuf

gRPC communication requires a protobuf definition of the functions, parameters, and responses implemented by the nodes.

Every node in our system implements the following functions:

```proto
service Paxos{
  rpc Prepare(PrepareRequest) returns (PrepareResponse);
  rpc Accept(AcceptRequest) returns (AcceptResponse);
  rpc ProposeLeader(ProposeLeaderRequest) returns (ProposeLeaderResponse);
}
message PrepareRequest {
  int64 proposal_id = 1; // proposal number
  int64 slot_id = 2; // proposal slot number
}

enum CommandType{
  UNKNOWN = 0; // unknown command
  SET = 1; // command to set a value
  DELETE = 2; // command to delete a value
}

message Command{
  CommandType type = 1; // command type
  string key = 2; // key of the value to be manipulated
  bytes value = 3; // value to be manipulated, if applicable
  int64 proposal_id = 4; // proposal number associated with the command
}

message PrepareResponse {
  bool success = 1;
  string error_message = 2; // error message, if any
  int64 accepted_proposal_id = 3; // accepted proposal number
  Command accepted_command = 4; // accepted command, if any
  int64 current_proposal_id = 5; // current proposal number
}

message AcceptRequest {
  int64 proposal_id = 1; // proposal number
  int64 slot_id = 2; // proposal slot number
  Command command = 3; // command to be accepted
}

message AcceptResponse {
  bool success = 1; // indicates if acceptance was successful
  int64 current_proposal_id = 2; // accepted proposal number
  string error_message = 3; // error message, if any
}
```

In this case, the Prepare function corresponds to **PHASE 1** and the Accept function corresponds to **PHASE 2**, with PrepareRequest being **1a** and PrepareResponse being **1b**, and likewise for accept.

## Leader Election

To elect the leader of a Paxos system, we can use Paxos itself, but it's a bit more complex due to some peculiarities.

How does a node decide if it should try to elect itself as leader?

The simplest way we could think of is through heartbeats, where the current leader periodically sends a heartbeat message to all network nodes. If a node goes more than a certain time without receiving a heartbeat from the leader, it assumes the leader is offline and after a certain random delay (to avoid multiple nodes initiating election at the same time) it tries to propose itself as leader.

For this part, we used the following RPC functions:

```proto
service Paxos{
  rpc ProposeLeader(ProposeLeaderRequest) returns (ProposeLeaderResponse);
  rpc SendHeartbeat(LeaderHeartbeat) returns (LeaderHeartbeatResponse);
}

message ProposeLeaderRequest{
  int64 proposal_id = 1; // Proposal number for leader election
  string candidate_address = 2; // Leader candidate address
}

message ProposeLeaderResponse{
  bool success = 1;
  string error_message = 2; // Error message, if any
  int64 current_highest_leader_proposal_id = 3; // Current leader's proposal number
  string current_leader_address = 4; // Current leader's address
  int64 highest_decided_slot_id = 5; // Highest slot decided so far
}

message LeaderHeartbeat{
  string leader_address = 1; // Leader address
  int64 current_proposal_id = 2; // Leader's current proposal number
  int64 highest_decided_slot_id = 3; // Highest slot decided so far
}

message LeaderHeartbeatResponse{
  bool success = 1; // Indicates if heartbeat was successful
  string error_message = 2; // Error message, if any
  int64 known_highest_slot_id = 3; // Highest slot the Acceptor/Learner knows
}
```

Additionally, to avoid problems we encountered during testing, if more than one node perceives the lack of a leader and they start an election, if the node receives any indication that an election is already happening with an ID greater than its own election, whether by receiving the proposal directly or by receiving a denial from another node containing this higher ID, it aborts its own election.

## Client

To be able to use the KV-Store without having direct access to the nodes and a way to execute these functions through them, all nodes also implement the following gRPC definition:

```proto
service KVStore{
  rpc Get(GetRequest) returns (GetResponse);
  rpc Set(SetRequest) returns (SetResponse);
  rpc Delete(DeleteRequest) returns (DeleteResponse);
  rpc List(ListRequest) returns (ListResponse);
  rpc ListLog(ListRequest) returns (ListLogResponse);
  rpc TryElectSelf(TryElectRequest) returns (TryElectResponse);
}

message GetRequest {
  string key = 1;
}

message GetResponse {
  string value = 1;
  bool found = 2;
  string error_message = 3;
}

message SetRequest {
  string key = 1;
  string value = 2;
}

message SetResponse {
  bool success = 1;
  string error_message = 2;
}

message DeleteRequest {
  string key = 1;
}

message DeleteResponse {
  bool success = 1;
  string error_message = 2;
}

message ListRequest{}

message ListResponse {
  repeated KeyValuePair pairs = 1;
  string error_message = 3; // Error message, if any
}

message KeyValuePair {
  string key = 1;
  string value = 2;
}

message ListLogResponse {
  repeated LogEntry entries = 1;
  string errorMessage = 2;
}

message LogEntry {
  int64 slot_id = 1;
  Command command = 2; // Command associated with the log
}

message Command{
  paxos.CommandType type = 1; // command type
  string key = 2; // key of the value to be manipulated
  string value = 3; // value to be manipulated, if applicable
  int64 proposal_id = 4; // proposal number associated with the command
}

message TryElectRequest {
}

message TryElectResponse {
  bool success = 1; // Indicates if election was successful
  string error_message = 2; // Error message, if any
}
```

Thus, it's possible to use a gRPC client that makes these calls to a specific node.

For better user interaction, an HTTP server (also in Golang) was also developed, which acts as a gRPC client for the KVStore service. Thus, accompanied by the frontend developed in React, we can visualize and send commands to each system node.

## Registry

Right, but how do the nodes and HTTP server know the node addresses to be able to make gRPC requests?

We also developed a simple registry service, also through gRPC, in which each node registers upon startup and allows the addresses of all system nodes to be queried. In addition to using heartbeats for knowledge of which node is active or not.

The registry is defined by the following protobuf:

```proto
service Registry {
  rpc Register(RegisterRequest) returns (RegisterResponse);
  rpc ListAll(google.protobuf.Empty) returns (ListResponse);
  rpc Heartbeat(HeartbeatRequest) returns (HeartbeatResponse);
}
```

## Synchronization

The synchronization part concerns the synchronization of the decided commands state between all nodes.

If we have a system with 50 nodes running, we insert some values, and then 10 new nodes are connected, they need to know the history of values decided by the other 50. For this, we also have the following function:

```proto
service Paxos{
  rpc Learn(LearnRequest) returns (LearnResponse);
}

message LearnRequest{
  int64 slot_id = 1; // Slot number being learned
}

message LearnResponse{
  bool decided = 1; // Indicates if the slot was decided
  Command command = 2; // Command associated with the slot, if decided
}
```

When a node receives a HeartBeat, the number of the last slot decided by the system is sent along, and if it's greater than what the node has in its internal state, it executes **Learn** until it synchronizes its entire state.

This synchronization is also done during the election of a new leader if necessary.

# Conclusion

The implementation of this project was challenging and brought a lot of learning about concurrency problems and distributed systems, in addition to enabling greater practice with Golang, which is a very interesting language for concurrent systems and the gRPC communication protocol.

The code for the project can be found in our [GitHub Repository](https://github.com/luizgustavojunqueira/KV-Store-Paxos?tab=readme-ov-file#key-value-store-with-paxos) along with instructions on how to run all the services involved.
