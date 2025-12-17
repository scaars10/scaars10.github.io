# From Paper to Reality: What I Learned Implementing Raft from Scratch

---

## 1. Overview

Unlike most engineering programs, my college dedicated the entire 6th semester to internships. I learned a ton—both professionally and personally—and honestly loved the experience (a stark contrast to my college life :D).

While most of my internship focused on Machine Learning, I found myself curious about distributed systems as well. Some of my team members were working on Spark, and the sheer scale they handled fascinated me. I wanted to build something from scratch to really understand how things worked under the hood.

I actually started with the Bayou paper, but I hit a wall pretty fast. It’s a great paper, but without a community or a mentor to bounce ideas off of, the implementation details felt too opaque, and online resources were scarce.

That’s when I switched to Raft. The paper ("In Search of an Understandable Consensus Algorithm") was a breath of fresh air—easy to read and digest. It was elegant, modular, and most importantly, approachable. Leader Election, Log Replication, Safety—it all seemed to click. Plus, the internet is full of Raft discussions, so whenever I got stuck, I knew I could find help. It felt like something I could actually build.

I decided to build **PecanRaft** to prove my understanding. The reality, however, was a humbling lesson in the difference between theoretical comprehension and engineering practice. This blog isn't about a production-ready system—my implementation was far from it. Instead, it’s about the specific, often painful lessons I learned when the clean abstractions of the paper met the messy reality of code, threads, and network failures.

---

## 2. The Illusion of Simplicity

When reading the paper, everything seemed so sequential.
*   *Timeout happens -> Start election.*
*   *Get votes -> Become leader.*

But when I tried to code this in Java, I realized that the paper completely ignores the hardest part: **Concurrency**.

### Threads are Confusing
The paper describes logic as if it runs in a straight line. But in my code, I had gRPC threads receiving messages, a main thread processing logic, and timer threads ticking in the background.

I thought I could just use `synchronized` keywords on my methods and be safe. I was wrong. I ended up with weird bugs where a node would vote for a candidate *after* it had already received a heartbeat from a leader, just because the threads interleaved in a way I didn't expect.

---

## 3. Naive Implementation Choices

### 3.1 Why I used gRPC Streaming
I chose **gRPC bidirectional streaming** for the logs.
*   **The Idea**: I wanted to allow for a flexible back-and-forth conversation between the leader and follower, enabling the leader to fetch missing logs if the follower fell behind.
*   **The Reality**: I didn't actually use it for long-lived connections. I ended up creating a new stream for every single heartbeat cycle. It was inefficient (creating a new connection every 70ms!), but ironically, it saved me from the complex error handling of persistent streams. It worked, but it definitely wasn't "optimal."

### 3.2 Using MongoDB for Everything
I used MongoDB because it was the only database I was comfortable with at the time.
*   **The Limitation**: I didn't understand the performance cost. Every time a node received a log entry, I wrote it to MongoDB before responding. This made my "high throughput" system crawl. I learned later that real systems use optimized write-ahead logs (WALs), not a full document database for every single entry.

---

## 4. Stuff the Paper Doesn't Tell You

### 4.1 Randomized Timers
The paper says: *"Randomize your election timeouts to prevent split votes."*

I expected this to be tricky, but it turned out to be one of the smoother parts of the implementation. Java’s `ScheduledExecutorService` handled the heavy lifting. On every heartbeat, I simply cancelled the previous future and scheduled a new one with a random delay. It worked effectively to prevent split votes without much fuss.

### 4.2 Testing Fault Tolerance
I ran each node in its own terminal window to simulate a real cluster. The "fun" part was testing fault tolerance: I would manually kill a Leader node's process and watch the other terminals to see if they noticed.

It was satisfying to see a Follower timeout, become a Candidate, and win the election in real-time, and see the old leader become a follower once it came back and sync the data from new leader. But it also meant I had to trace why a specific node *didn't* vote the way I expected during a partition. Some bugs would take days to find and fix and I am pretty sure there must be plenty still left to find as I barely got it to work and I am sure there are still some edge cases I haven't covered. I learned that just standard logs weren't enough—I needed detailed context (Current Term, Voted For, etc.) in every log line to make sense of the distributed state across my screens.

---

## 5. What I Learned

| Concept | The "Paper" Understanding | The Reality |
| :--- | :--- | :--- |
| **RPCs** | You send a message, you get a reply. | You send a message, and maybe it gets there, or maybe your wifi blips and your code crashes. |
| **State** | You are either Follower, Candidate, or Leader. | You are confusingly both because you haven't updated your variable yet. |
| **Concurrency** | "Handle messages sequentially". | Using `synchronized` is easy to write but hard to get right. |

---

## 6. Summary

**PecanRaft** is definitely not production-ready. It has bugs, it's slow, and it relies on MongoDB in a way that makes no sense for a real consensus engine.

But building it taught me that **Distributed Systems are hard not because the algorithms are complex, but because the world is messy.** Threads race, networks fail, and simple logic gets complicated fast.

If you really want to learn Raft, don't just read the paper. Try to build it. You'll fail a lot, but you'll actually understand it.
