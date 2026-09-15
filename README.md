# Byzantine Fault-Tolerant Replication System

A **Byzantine fault-tolerant state machine replication system** implemented in **Go** using a linear variant of **Practical Byzantine Fault Tolerance (PBFT)**.

The system runs across **7 replicas** and tolerates up to **2 Byzantine failures**, satisfying the PBFT requirement:

```text
n = 3f + 1
```

where:

```text
n = 7 replicas
f = 2 Byzantine replicas
```

The project uses a replicated banking workload to demonstrate Byzantine consensus, authenticated message exchange, deterministic state machine execution, leader recovery, checkpointing, and resilience against malicious replica behavior.

## Key Features

* **Linear PBFT** consensus across 7 replicas
* Tolerates up to **2 Byzantine failures**
* **State Machine Replication** for deterministic transaction execution
* Authenticated **PRE-PREPARE, PREPARE, and COMMIT** phases
* **PBFT view change** for faulty or malicious leader recovery
* **Ed25519 digital signatures** for message authentication
* **SHA-256 digests** for transaction integrity
* **2f + 1 quorum validation**
* **Checkpointing** for stable replica state and recovery
* Byzantine attack simulation including:

  * Invalid signatures
  * Crash/silent replicas
  * Selective message withholding
  * Timing attacks
  * Equivocation
* Read-only quorum processing
* Duplicate request detection
* Client retries and leader forwarding
* Redis-backed replicated account state
* SmallBank-style benchmarking with configurable workload skew

---

# Architecture

The system consists of one replicated group containing **7 replicas**.

```text
                         Clients
                      A, B, C ... J
                           |
                           v
                        Primary
                           |
          +----------------+----------------+
          |        |       |       |        |
          v        v       v       v        v
         n2       n3      n4      n5       n6
                           |
                           v
                          n7

                Replicated State Machine
                           |
                           v
                     Account Balances
```

Every correct replica maintains the same banking state.

The system contains **10 banking clients**, identified as:

```text
A B C D E F G H I J
```

Each account initially has:

```text
balance = 10
```

---

# Byzantine Fault Model

Unlike crash-only systems, Byzantine systems must tolerate replicas that can behave arbitrarily.

A Byzantine replica may:

* Send incorrect messages
* Send different messages to different replicas
* Corrupt signatures
* Stop responding
* Selectively communicate with only part of the cluster
* Delay protocol execution
* Attempt to disrupt consensus

For PBFT:

```text
n >= 3f + 1
```

With 7 replicas:

```text
7 = 3(2) + 1
```

the system can tolerate:

```text
f = 2
```

Byzantine replicas.

The primary quorum threshold used by the protocol is:

```text
2f + 1 = 5
```

Therefore, at least **5 matching authenticated replica responses** are required for important protocol decisions.

---

# Linear PBFT

The system implements a **leader-driven linear variant of PBFT**.

Classic PBFT uses all-to-all communication during the PREPARE and COMMIT phases, leading to approximately:

```text
O(n²)
```

communication.

This implementation uses a collector-style design in which replicas send signed responses to the primary, which then distributes quorum evidence.

The normal protocol flow is:

```text
Client
   |
   | Signed transaction
   v
Primary
   |
   | PRE-PREPARE
   +------------------------> Replicas
   |
   | <---- Signed responses
   |
   | Collect 2f+1 responses
   |
   | PREPARE + quorum proof
   +------------------------> Replicas
   |
   | <---- Signed responses
   |
   | Collect 2f+1 responses
   |
   | COMMIT + quorum proof
   +------------------------> Replicas
   |
   v
Ordered Execution
```

This reduces unnecessary replica-to-replica communication while preserving Byzantine quorum guarantees.

---

# PBFT Protocol Phases

## 1. Client Request

A client creates a transaction such as:

```text
A -> B : 3
```

The request includes identifying information such as:

```text
sender
receiver
amount
timestamp
```

The client signs the transaction using its **Ed25519 private key**.

The receiving replica verifies the client's signature before processing the transaction.

---

## 2. PRE-PREPARE

The primary:

1. Verifies the client request
2. Assigns a sequence number
3. Computes a SHA-256 digest
4. Creates a PRE-PREPARE message
5. Signs the message
6. Broadcasts it to replicas

The message contains information such as:

```text
View Number
Sequence Number
Transaction Digest
Transaction
Primary Signature
```

Each backup validates:

* The current view
* Primary identity
* Primary signature
* Transaction digest
* Sequence number
* Existing log state

If valid, the replica records the request as:

```text
PREPREPARED
```

and returns a signed acknowledgement.

---

## 3. PREPARE

The primary collects enough signed responses from replicas.

Once it obtains a quorum:

```text
2f + 1 = 5
```

it distributes the corresponding quorum proof in the PREPARE phase.

Each replica validates the signed evidence.

If enough valid matching responses exist, the request transitions from:

```text
PREPREPARED
      |
      v
   PREPARED
```

The replica then returns a signed PREPARED acknowledgement.

---

## 4. COMMIT

The primary collects another quorum of PREPARED responses and sends the corresponding proof in the COMMIT phase.

Each replica verifies:

```text
view
sequence number
digest
replica signatures
quorum size
```

If the certificate is valid, the entry becomes:

```text
COMMITTED
```

A committed transaction becomes eligible for state machine execution.

---

# State Machine Replication

Each replica maintains an ordered log of transactions.

For example:

```text
Seq 1 -> T1
Seq 2 -> T2
Seq 3 -> T3
```

Transactions must execute strictly in sequence order.

A replica maintains:

```text
LastExecutedSequenceNumber
```

and executes only:

```text
LastExecutedSequenceNumber + 1
```

Suppose:

```text
Seq 10 -> COMMITTED
Seq 11 -> PREPARED
Seq 12 -> COMMITTED
```

The replica executes sequence 10 and waits for sequence 11.

It does not execute sequence 12 early.

This ensures that all correct replicas execute transactions in the same deterministic order and therefore maintain identical state.

---

# Banking State Machine

A transaction has the form:

```text
Transfer(Sender, Receiver, Amount)
```

For:

```text
A -> B : 3
```

a replica checks:

```text
Balance[A] >= 3
```

If valid:

```text
Balance[A] -= 3
Balance[B] += 3
```

The resulting state is persisted in Redis.

Because every correct replica processes the same ordered transaction log, their banking states remain consistent.

---

# Cryptographic Authentication

The project uses **Ed25519 digital signatures** to authenticate both client and replica messages.

Each node has its own public/private key pair.

Example:

```text
node1.priv
node1.pub
...
node7.priv
node7.pub
```

Clients similarly maintain individual key pairs.

This provides:

* Sender authentication
* Message integrity
* Protection against forged protocol messages

A malicious replica cannot successfully impersonate another correct replica without access to its private key.

---

# SHA-256 Digests

Transactions are identified using SHA-256 digests.

Conceptually:

```text
Digest = SHA256(transaction)
```

A digest allows replicas to verify that different protocol messages refer to exactly the same transaction.

If a Byzantine node changes:

```text
A -> B : 3
```

into:

```text
A -> B : 8
```

the transaction digest changes, allowing correct replicas to reject the conflicting message.

---

# Leader Selection

PBFT operates using numbered **views**.

Each view has one primary.

The leader is selected deterministically:

```text
leader = ((view - 1) % 7) + 1
```

For example:

```text
View 1 -> n1
View 2 -> n2
View 3 -> n3
View 4 -> n4
```

This means that when the current primary becomes faulty, replicas already know which node should become the primary of the next view.

---

# PBFT View Change

If the current primary prevents the protocol from making progress, correct replicas initiate a **view change**.

Examples include:

* Primary crash
* Message withholding
* Excessive delay
* Malicious protocol behavior

The flow is:

```text
Faulty Primary
      |
      v
Execution timeout
      |
      v
VIEW-CHANGE
      |
      v
Next Primary
      |
      v
Collect View-Change Messages
      |
      v
Recover Safe Prepared Requests
      |
      v
NEW-VIEW
      |
      v
Consensus Continues
```

---

## View-Change Proofs

A replica includes information about transactions that have reached sufficiently safe protocol states.

This may include:

```text
Sequence Number
Digest
Transaction
Previous View
Prepared Proof
Status
```

These proofs ensure that transactions which may already have been safely accepted cannot simply be replaced by a different transaction after a leader change.

---

# Safe Log Reconstruction

The new primary collects view-change messages and reconstructs the safe transaction history.

For each sequence number, the new primary selects the valid prepared request associated with the highest relevant previous view.

Example:

```text
Replica 1:
Seq 8 -> T1, View 2

Replica 2:
Seq 8 -> T1, View 2

Replica 3:
Seq 8 -> T2, View 1
```

The new primary preserves:

```text
Seq 8 -> T1
```

because it represents the safer prepared history.

Missing sequence numbers can be represented by:

```text
NULL
```

requests.

A NULL request changes no application state but allows ordered state machine execution to continue without sequence gaps.

---

# Checkpointing

The system periodically creates checkpoints after a configured number of executed requests.

Checkpointing provides a stable replica state from which older protocol history can be safely summarized or discarded.

Conceptually:

```text
Executed Requests

1
2
3
...
100
 |
 v
CHECKPOINT
 |
 v
Stable State
```

Replicas exchange signed checkpoint messages and collect matching checkpoint proofs.

Checkpointing helps:

* Establish stable state boundaries
* Reduce retained protocol history
* Support synchronization across replicas
* Improve protocol recovery efficiency

---

# Byzantine Attack Simulation

The project explicitly supports several malicious replica behaviors to test protocol resilience.

## Invalid Signature Attack

A Byzantine replica intentionally corrupts its signature before sending a protocol message.

Correct replicas verify Ed25519 signatures and reject the invalid message.

```text
Replica
   |
valid signature
   |
tamper signature
   |
   v
Correct Replica
   |
signature verification fails
   |
REJECT
```

---

## Crash / Silence Attack

A faulty replica stops participating in the protocol.

It may ignore:

```text
PRE-PREPARE
PREPARE
COMMIT
VIEW-CHANGE
NEW-VIEW
```

Consensus can still progress as long as the number of faulty nodes does not exceed:

```text
f = 2
```

---

## Selective Message Withholding

A Byzantine replica may communicate with some replicas while ignoring others.

Example:

```text
Primary -> n2 ✓
Primary -> n3 ✓
Primary -> n4 ✗
Primary -> n5 ✓
```

This simulates asymmetric malicious behavior rather than a simple crash.

---

## Timing Attack

A malicious replica deliberately delays protocol messages.

This tests whether timeout and view-change mechanisms can recover when the leader remains partially responsive but prevents timely progress.

---

## Equivocation

A Byzantine primary may send conflicting protocol information to different replicas.

Example:

```text
n2 receives:
Seq 10 -> Transaction X

n3 receives:
Seq 10 -> Transaction X

n4 receives:
Seq 10 -> Transaction Y
```

Signatures, transaction digests, quorum validation, and PBFT safety rules prevent conflicting transactions from obtaining valid commit certificates simultaneously.

---

# Read-Only Operations

Balance reads do not require the full PBFT consensus path.

A client broadcasts a signed balance request to replicas.

```text
Client
  |
  +------ READ ------> n1
  +------ READ ------> n2
  +------ READ ------> n3
  +------ READ ------> ...
  +------ READ ------> n7
```

Replicas verify the client signature and return signed balance responses.

The client accepts a result after receiving:

```text
2f + 1 = 5
```

matching authenticated responses.

This avoids the overhead of running full consensus for read-only requests.

---

# Duplicate Request Handling

Clients may retry requests if responses are delayed or lost.

The system identifies transactions using their transaction metadata and digest.

Already executed transactions are tracked so that a repeated client request does not execute the same banking transfer twice.

Instead, the previously computed result can be returned.

---

# Client Retry and Leader Forwarding

Clients normally send requests to the primary for the current view.

If the request cannot progress:

1. The client retries the request
2. Other replicas may receive the request
3. Backups can forward requests to the current primary
4. A view change may occur if the primary is faulty

This allows the system to continue processing requests during leader transitions and partial failures.

---

# Redis Storage

Redis is used as the backing store for each replica's local banking state.

Each replica maintains its own copy of account balances.

Consensus is **not provided by Redis**.

Instead:

```text
PBFT
 |
 v
determines transaction order
 |
 v
State Machine Replication
 |
 v
updates local state
 |
 v
Redis persistence
```

This separation is important because PBFT provides distributed agreement, while Redis stores each replica's resulting application state.

---

# Benchmarking

The project includes a configurable benchmarking framework based on a banking workload.

Benchmark parameters include:

* Number of concurrent workers
* Read/write ratio
* Request rate
* Benchmark duration
* Workload skew

---

## Workload Skew

The benchmark supports skewed account access using a Zipf-style distribution.

Low skew produces approximately uniform access:

```text
A  ████
B  ████
C  ████
D  ████
```

Higher skew creates hot accounts:

```text
A  ███████████████████
B  ███████████
C  ████
D  ██
```

This helps evaluate the system under realistic contention patterns.

---

## Performance Metrics

The benchmarking framework measures:

* Throughput
* Read latency
* Write latency
* P50 latency
* P95 latency
* P99 latency
* Successful operations
* Failed operations

These metrics help analyze the performance cost of Byzantine consensus under different workloads.

---

# Technology Stack

| Component     | Technology                |
| ------------- | ------------------------- |
| Language      | Go                        |
| Consensus     | Linear PBFT               |
| Replication   | State Machine Replication |
| RPC           | gRPC                      |
| Serialization | Protocol Buffers          |
| Signatures    | Ed25519                   |
| Hashing       | SHA-256                   |
| Storage       | Redis                     |
| Failure Model | Byzantine                 |
| Benchmarking  | SmallBank-style workload  |

---

# Project Structure

```text
.
├── cmd/
│   ├── node/                 # Starts PBFT replica nodes
│   ├── client/               # Banking transaction client
│   ├── client-funcns/        # Inspection/debugging utilities
│   ├── bench/                # Benchmark runner
│   └── genKeys/              # Cryptographic key generation
│
├── internal/
│   ├── node/
│   │   ├── node.go           # Replica state
│   │   ├── helper.go         # PBFT helper logic
│   │   └── view_change.go    # View-change implementation
│   │
│   ├── client/
│   │   ├── hub.go            # Client reply aggregation
│   │   ├── hub_logic.go      # Quorum and response processing
│   │   ├── server.go         # Client RPC handling
│   │   └── csv_parser.go     # Workload parsing
│   │
│   ├── crypto/
│   │   └── crypto.go         # Ed25519 signing and verification
│   │
│   └── bench/
│       ├── workload.go       # Benchmark workload generation
│       ├── runner.go         # Benchmark execution
│       ├── metrics.go        # Latency/throughput metrics
│       └── loader.go
│
├── database/
│   └── redis.go              # Replica-local persistent state
│
├── gRPC/
│   ├── server.go             # PBFT RPC server
│   └── helper.go
│
├── proto/
│   └── pbft.proto            # Protocol Buffer definitions
│
├── cluster/
│   └── manifest.json         # Cluster configuration
│
├── keys/
│   ├── node*.priv
│   ├── node*.pub
│   ├── client*.priv
│   └── client*.pub
│
├── go.mod
└── go.sum
```

---

# Protocol Summary

The complete normal transaction path is:

```text
Client
  |
  | sign transaction
  v
Primary
  |
  | verify signature
  | assign sequence number
  | compute SHA-256 digest
  |
  v
PRE-PREPARE
  |
  v
Replica Validation
  |
  v
2f+1 Signed Responses
  |
  v
PREPARE
  |
  v
2f+1 Signed Responses
  |
  v
COMMIT
  |
  v
Ordered State Machine Execution
  |
  v
Update Account Balances
  |
  v
Persist to Redis
  |
  v
Signed Client Replies
```

---

# View-Change Summary

When the primary is faulty:

```text
Primary fails or behaves maliciously
              |
              v
      Progress timeout
              |
              v
         VIEW-CHANGE
              |
              v
      New Primary Selected
              |
              v
 Collect Safe Prepared History
              |
              v
        Rebuild Log
              |
              v
          NEW-VIEW
              |
              v
      Consensus Resumes
```

---

# Distributed Systems Concepts Demonstrated

This project combines several important distributed systems and fault-tolerance concepts:

* Byzantine Fault Tolerance
* Practical Byzantine Fault Tolerance
* Linear PBFT
* State Machine Replication
* Byzantine quorum systems
* Quorum certificates
* Digital signatures
* Cryptographic hashing
* Leader-based consensus
* View change
* Safe log reconstruction
* Deterministic execution
* Checkpointing
* Replica synchronization
* Client retries
* Duplicate suppression
* Byzantine attack simulation
* Tail-latency benchmarking
* Skewed workloads

---

# Summary

This project demonstrates how a replicated transaction-processing system can remain safe and available even when some replicas behave maliciously.

The core design combines:

```text
               Client Authentication
                       |
                       v
                  Linear PBFT
                       |
         +-------------+-------------+
         |                           |
         v                           v
 Byzantine Consensus         PBFT View Change
         |                           |
         +-------------+-------------+
                       |
                       v
             State Machine Replication
                       |
                       v
                 Checkpointing
                       |
                       v
               Replicated State
```

With **7 replicas and f = 2**, the system maintains correct replicated execution despite Byzantine nodes attempting to crash, equivocate, delay communication, send invalid signatures, or selectively withhold protocol messages.
