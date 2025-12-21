# From Paper to Reality: What I Learned Implementing Raft from Scratch

## Overview

Unlike most engineering programs, my college dedicated the entire 6th semester to internships. I learned a ton, both professionally and personally, and honestly loved the experience (a stark contrast to my college life :D).

While most of my internship focused on Machine Learning, I found myself curious about distributed systems as well. Some of my team members were working on Spark, and the sheer scale they handled fascinated me. I wanted to build something from scratch to really understand how things worked under the hood.

I actually started with the Bayou paper, but I hit a wall pretty fast. It's a great paper, but without a community or a mentor to bounce ideas off of, the implementation details felt too opaque, and online resources were scarce.

That's when I switched to Raft. The paper ("In Search of an Understandable Consensus Algorithm") was a satisfying and great experience. It was easy to read and digest. It was elegant, modular, and most importantly, approachable. Leader Election, Log Replication, Safety. It all seemed to click. Plus, the internet is full of Raft discussions, so whenever I got stuck, I knew I could find help. It felt like something I could actually build.

I decided to build **PecanRaft** to prove I understood it. The process, however, quickly highlighted the gap between reading a paper and building a working system. 

This blog isn't about a production-ready system—my implementation was far from it. It's about the specific lessons I learned when the clean abstractions of the paper met the reality of race conditions and network failures.

*This blog was written five years after I learned that distributed systems are mostly about handling the unknown. The code hasn't changed since then.*


---

## Concurrency Challenges

When reading the paper, everything seemed so sequential:
- *Timeout happens → Start election*
- *Get votes → Become leader*
- *Receive log → Replicate → Commit*

But when I tried to code this in Java, I realized that the paper completely ignores the hardest part: **Concurrency**.

### The Threading Reality

The paper might give you an illusion that it runs in a straight line. But in my code, I had:
- **gRPC threads** receiving incoming requests
- **Timer threads** firing election timeouts
- **Main logic thread** processing state transitions
- **Client request handlers** reading/writing data

All of these were trying to access the same shared state (`currentTerm`, `votedFor`, `log`) simultaneously.

### My Naive Approach

I thought I could just use `synchronized` keywords on my methods and be safe. I was wrong.

```java
// What I thought would work
public synchronized void handleVote() {
    currentTerm++;
    nodeState = CANDIDATE;
}

public synchronized void handleHeartbeat() {
    resetElectionTimer();
    nodeState = FOLLOWER;
}
```

The problem? These methods sometimes needed to call each other, or acquire multiple locks in different orders.

### What Actually Happened

```java
// Thread 1: In appendEntries()
node.logLock.writeLock().lock();      // Lock A
node.nodeLock.writeLock().lock();     // Then Lock B

// Thread 2: In updateStatus() 
node.nodeLock.writeLock().lock();     // Lock B
node.logLock.readLock().lock();       // Then Lock A

// Result: DEADLOCK
```

I had multiple threads—timers firing elections, gRPC handlers processing heartbeats, and client requests reading data—all trying to touch the same state. Slapping locks on everything caused the system to deadlock and freeze randomly. Using too few locks corrupted the state. 

**The Lesson**: Understanding the algorithm is straightforward. Implementing it in a way that handles concurrency correctly without deadlocking is the actual engineering challenge.

### What I Should Have Done (But Didn't)

Looking back now, here's what I *should* have done:

1. **Consistent Lock Ordering**: Always acquire locks in the same order across all methods
2. **Minimize Critical Sections**: Hold locks for the shortest time possible
3. **Use Read-Write Locks Strategically**: Distinguish between reads and writes
4. **Consider Lock-Free Structures**: Use `AtomicReference` for frequently-read state

But I never actually fixed these issues. The deadlocks still happen occasionally in my code.

---

## Implementation Choices

### Why I Used gRPC Streaming (And Why It Was Overkill)

I chose **gRPC bidirectional streaming** for the AppendEntries RPC.

**The Idea**: I wanted to allow for a flexible back-and-forth conversation between the leader and follower, enabling the leader to fetch missing logs if the follower fell behind. It seemed elegant—one persistent connection, continuous communication.

**The Reality**: I didn't actually use it for long-lived connections. I ended up creating a new stream for every single heartbeat cycle. 

```java
// Every 150ms, this ran:
ManagedChannel channel = ManagedChannelBuilder
    .forAddress(address, port)
    .usePlaintext()
    .build();
    
// Send heartbeat, then:
channel.shutdown();
```

It was inefficient (reconnecting every 150ms is poor practice), but it avoided the complexity of managing long-lived streams. If a connection died, I just made a new one. It worked, but it wasn't elegant.

**Takeaway**: Sometimes the "wrong" solution teaches you more. I learned about gRPC overhead specifically because I saw the performance impact of doing it this way.

### Using MongoDB for Everything

I used MongoDB because it was the only database I was comfortable with at the time.

**The Limitation**: I didn't understand the performance cost. Every time a node received a log entry, I wrote it to MongoDB before responding:

```java
public void addToUncommittedLog(int key, int value) {
    LogEntry entry = new LogEntry(currentTerm, key, value, lastIndex+1);
    logs.add(entry);
    db.writeLog(entry);  // MongoDB write - SLOW!
}
```

This made the system slow. Each write involved serializing to BSON, a network call to Mongo, index updates, and the actual insert. For a consensus system that needs to fsync every log entry before acknowledging, this was a major bottleneck.

Real systems use optimized write-ahead logs (WALs) like LevelDB/RocksDB or custom append-only files designed for sequential writes. MongoDB is not designed for this use case.

## Specific Implementation Details

### Randomized Timers & The "Reset" Problem

The paper is clear: *"Randomize your election timeouts to prevent split votes."*

I implemented this using Java's `ScheduledExecutorService`:

```java
void startElectionTimer() {
    Random rand = new Random();
    int randomTime = (int) (leaderTimeout + rand.nextDouble() * 150);
    electionFuture = electionExecutor.schedule(
        this::startElection, 
        randomTime, 
        TimeUnit.MILLISECONDS
    );
}

void restartElectionTimer() {
    stopElectionTimer();  // Cancel old task
    startElectionTimer(); // Create new task
}
```

On every heartbeat received, I would cancel the existing timeout task and schedule a new one with a random delay (e.g., 150-300ms).

**The Trade-off**: While clean, this approach meant constant object creation (Future tasks) and cancellation overhead every 50-100ms.

**What would have been better**:
```java
class ElectionTimer {
    private volatile long lastHeartbeatTime;
    
    // Single background thread checks periodically
    public void run() {
        while (true) {
            Thread.sleep(50);
            if (System.currentTimeMillis() - lastHeartbeatTime > randomTimeout) {
                startElection();
            }
        }
    }
    
    public void reset() {
        lastHeartbeatTime = System.currentTimeMillis();
    }
}
```

This avoids the constant schedule/cancel churn and is more efficient. But my implementation never got there—the schedule/cancel approach was "good enough" for learning purposes.

### Visual Inspection & Debugging Hell

Running a distributed system on one laptop means staring at 5 terminal windows simultaneously. I manually killed processes to simulate crashes.

**The Problem**: Standard logs were useless:
```
INFO: Received vote
INFO: Timeout occurred
INFO: Sending heartbeat
```

I couldn't tell *which* term a vote belonged to, *which* node sent it, or *why* a node was rejecting logs.

**The Fix**: I moved to structured context in log lines:
```
[Node 1 | Term 5 | CANDIDATE] - Election timeout, starting election
[Node 2 | Term 5 | FOLLOWER]  - Received vote request from Node 1
[Node 2 | Term 5 | FOLLOWER]  - Granted vote to Node 1
[Node 1 | Term 5 | LEADER]    - Won election with 3/5 votes
```

Suddenly, I could trace the flow of events across the cluster by reading logs side-by-side.


### The "Catch-Up" Mechanism

When a follower comes back online after a crash, it's missing potentially thousands of log entries.

**My Approach**: Since I was using gRPC streaming, I simply let the Leader detect the mismatch in `nextIndex` and start pushing entries one by one in the response stream.

```java
if (value.getResponseCode() == ResponseCodes.MORE) {
    long index = value.getMatchIndex();
    List<LogEntry> logs = node.getLogs((int) index, -1);
    // Send logs starting from index...
}
```

This worked, but it was essentially **"Stop-and-Wait"**. The Leader sends entry N, waits for an ack, then sends N+1. For a node that's 10,000 entries behind, this is incredibly slow.

Strategies like pipelining (sending batches without waiting) or snapshotting (sending the entire state) would be the correct approach, but I didn't implement them at the time.

### The "Poor Man's Quorum" (CountDownLatch)

The Raft algorithm requires the leader to wait for a majority of followers to confirm a log entry before committing it. This is inherently asynchronous.

**My Hack**: I used Java's `CountDownLatch` initialized to `(ClusterSize / 2) + 1`:

```java
ResettableCountDownLatch consensusLatch = new ResettableCountDownLatch(1);

// In systemService (client request handler):
public void systemService(ClientRequest request, 
                         StreamObserver<ClientResponse> responseObserver) {
    if (nodeState != LEADER) {
        responseObserver.onNext(ClientResponse.newBuilder()
            .setSuccess(false)
            .setLeaderId(leaderId)
            .build());
        return;
    }
    
    // Block until consensus is reached
    boolean result = consensusLatch.await(1000, TimeUnit.MILLISECONDS);
    
    if (result && nodeState == LEADER) {
        node.addToUncommittedLog(request.getKey(), request.getValue());
        responseObserver.onNext(ClientResponse.newBuilder()
            .setSuccess(true)
            .build());
    }
}

// In appendEntries broadcast:
void allAppendEntries() {
    AtomicInteger successCount = new AtomicInteger(1);
    
    // Broadcast to all followers...
    
    // Wait for majority
    for (int i = 0; i < 20; i++) {
        Thread.sleep(70);
        if (successCount.get() > peerId.length / 2) {
            node.setCommitIndex(currentSize);
            break;
        }
    }
}
```

**Why it worked**: It made the code look sequential: `Send -> Wait -> Commit`. It turned a distributed problem into a local threading problem.

**Why it's bad**: It holds a thread hostage for every single client request. If a follower is slow, the leader's thread is blocked doing nothing. It doesn't scale.

**What a better implementation would look like** (that I never built):
```java
// Asynchronous handling with CompletableFuture
CompletableFuture<Boolean> waitForQuorum(LogEntry entry) {
    return CompletableFuture.supplyAsync(() -> {
        AtomicInteger acks = new AtomicInteger(1);
        
        followers.parallelStream().forEach(follower -> {
            if (appendEntry(follower, entry)) {
                acks.incrementAndGet();
            }
        });
        
        return acks.get() > (clusterSize / 2);
    });
}

// Then chain it:
waitForQuorum(entry)
    .thenAccept(success -> {
        if (success) commitEntry(entry);
        respondToClient(success);
    });
```

This doesn't block threads and scales much better. But for my purposes—learning the algorithm on a 3-node cluster running on localhost—the CountDownLatch hack was sufficient.

---

## Comparison

| Concept | The "Paper" Understanding | The Reality |
|:--------|:--------------------------|:------------|
| **RPCs** | You send a message, you get a reply. | Messages get lost, replies are delayed, or the network fails entirely. |
| **State** | You are either Follower, Candidate, or Leader. | Threads might read stale state while another thread is updating it. |
| **Concurrency** | "Handle messages sequentially". | `synchronized` blocks are easy to add but difficult to coordinate without causing deadlocks. |
| **Timers** | "Set a random timeout". | Creating and canceling timers constantly introduces overhead. |
| **Log Replication** | "Append entries to your log". | Synchronous disk writes for every entry kills performance. |
| **Crashes** | "Node restarts and catches up". | Catch-up is slow if you re-send the log one entry at a time. |
| **Testing** | "Run it and see if it works". | It works in simple cases but fails under specific timing conditions or network delays. |

---

## Retrospective

Looking back at PecanRaft five years later, here's what I would change if I ever revisited this project. But to be clear: **I haven't touched this code in 5 years**. These are lessons learned from reading other implementations, working on production systems, and understanding what I got wrong.

### Architecture Changes

**Separate State Machine Interface**

My current code tightly couples the consensus logic with the key-value store. A cleaner approach would be:

```java
public interface StateMachine {
    void apply(LogEntry entry);
    byte[] snapshot();
    void restore(byte[] snapshot);
}

public class KeyValueStore implements StateMachine {
    private Map<Integer, Integer> data = new ConcurrentHashMap<>();
    
    @Override
    public void apply(LogEntry entry) {
        data.put(entry.getKey(), entry.getValue());
    }
}
```

This would decouple the consensus logic from the application logic, making the code more modular and testable. But my implementation has these concerns mixed together.

**Event-Driven Design**

Instead of having RPC handlers directly modify state, a better approach would be to use an event queue:

```java
class RaftNode {
    BlockingQueue<Event> eventQueue = new LinkedBlockingQueue<>();
    
    void mainLoop() {
        while (running) {
            Event event = eventQueue.take();
            handleEvent(event);  // Single-threaded processing
        }
    }
}
```

This would eliminate most concurrency issues by processing events sequentially. My current implementation has multiple threads all trying to modify state, which is where the deadlocks come from.

**Proper WAL Implementation**

Using LevelDB or RocksDB instead of MongoDB would look like:

```java
Options options = new Options();
options.createIfMissing(true);
DB db = factory.open(new File("raft-log"), options);

// Fast sequential writes
db.put(bytes("entry-" + index), serialize(entry));
```

This would be orders of magnitude faster than my current MongoDB approach.

### Elaborate Testing Strategy

**Chaos Testing**: Building test framework to inject:
- Network partitions
- Clock skew
- Process crashes
- Message reordering

I only tested the "happy path" with manual process kills. Real chaos testing would have found so many more bugs.

**Deterministic Simulation**: Testing with simulated time instead of real sleeps:

```java
class SimulatedClock {
    private long currentTime = 0;
    
    void advance(long millis) {
        currentTime += millis;
        // Trigger all timers that should fire
    }
}
```

This would make tests fast and reproducible instead of taking real wall-clock time.

**Property-Based Testing**: Verifying invariants like:
- At most one leader per term
- Committed entries never disappear
- Log matching property holds

I never tested these properties systematically—I just ran the system and hoped it worked.

### Observability

**Metrics I Should Have Tracked**:
- Leader election frequency (should be rare)
- Commit latency (time from log entry to commit)
- Log replication lag per follower
- RPC success/failure rates

**Structured Logging**:
```java
logger.info("state_transition", 
    Map.of(
        "node_id", nodeId,
        "term", currentTerm,
        "from_state", oldState,
        "to_state", newState,
        "trigger", trigger
    )
);
```

This would make logs machine-parseable for analysis instead of just human-readable.

But again—I never implemented any of this. PecanRaft remains in its original, flawed state as a time capsule of what I knew (and didn't know) during my internship.

---

## Takeaways

**Distributed Systems are hard.** It's not just the algorithms, but the implementation details. Dealing with race conditions, network failures, and component interactions is complex.

**Performance is a feature.** Choices like using MongoDB as a WAL can make a correct implementation unusable. 

**Testing needs to be rigorous.** If you aren't simulating network partitions or crashes, you aren't fully testing the system.

**Just Start.** Building this, however inefficiently, taught me more than reading the paper alone. Build it, measure it, and then fix it.

---

## Final Thoughts

**PecanRaft is not production-ready.** It has bugs I never fixed, performance issues I never resolved, and architectural decisions that make me cringe looking back. The deadlocks still happen. The MongoDB writes are still slow. The catch-up mechanism is still inefficient.

I haven't touched this code in 5 years, and likely never will. It serves as a snapshot of what I knew at the time.

Building this taught me that **reading papers is not the same as understanding systems**. 

If you want to learn Raft, I recommend **trying to build it**. You will likely encounter bugs and performance issues, just as I did. But debugging those issues—deadlocks, race conditions, and bottlenecks—is where you truly learn how the system works.

---

**Repository**: [github.com/scaars10/PecanRaft](https://github.com/scaars10/PecanRaft)  
