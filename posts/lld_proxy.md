# Simple Interface, Elegant Internals: Designing a Rate-Limiter for API Interaction

## Overview

When building systems that interact with third-party APIs like brokerage platforms, we often face a strict set of constraints: hard limits on request rates (RPM) and the need for decent reliability.

This post explores the Low-Level Design behind an order execution and price treacking module. The goal was to create a module that is **clean, maintainable, and respectful of API boundaries** while handling the concurrency required for managing multiple positions and tracked stocks.

I utilized several classic design patterns—**Proxy, Producer-Consumer, Decorator and Read-Through Caching** to abstract away the complexity of rate limiting, network calls and flow control from the core model and business logic.

---

## The Design Challenge

My requirements were straightforward but conflicting:
1.  **Multiple Workers**: I have multiple threads/processes running in parallel processing cpu/gpu intensive tasks while trying to place orders or check prices for different positions and stocks that I want to track simultaneously.
2.  **Strict Quotas**: The broker API allows only 10 requests/second and 1 request/second for quotes.
3.  **Simplicity**: The trading model should focus solely on crunching huge tensors of data and generating profitable signals, remaining completely agnostic to retries, rate limits, or thread locks.

A naive approach of scattering `time.sleep()` calls throughout the codebase leads to "spaghetti code" and unpredictable behavior. I needed a structural solution.

---

## Pattern 1: The Proxy Pattern (Transparent Wrapper)

To ensure the trading logic remains clean, I didn't want to change how I call the API. If the original call was `kite.place_order(...)`, I wanted to keep it exactly that way, transparently adding rate-limiting behavior.

I implemented a **Proxy** class `KiteRateLimiter` that wraps the actual API client.

```python
class KiteRateLimiter:
    def __init__(self, kite_instance):
        self.kite = kite_instance
        # ... logic ...

    def __getattr__(self, name):
        # Intercept all method calls
        return lambda *args, **kwargs: self._proxy(name, *args, **kwargs)
```

**Why it's elegant**:
*   **Decoupling**: The business logic is unaware it's talking to a wrapper.
*   **Centralized Control**: All API interaction goes through one funnel, making it the perfect place to inject logging, retries, and rate limiting.

---

## Pattern 2: Producer-Consumer with Queues

Rate limits are essentially a flow control problem. To manage this, I adopted a **Producer-Consumer** model using thread-safe queues.

I separated the API traffic into two distinct "lanes" or channels:
1.  **Transactional Lane**: For placing/modifying orders and general api calls.
2.  **Data Lane**: For fetching market quotes.

Each lane has its own `Queue` and a dedicated worker thread.

```python
# Producer: The main thread pushes a request to the queue
def _proxy(self, func_name, *args, **kwargs):
    q = self.channels['default']['queue']
    result_queue = queue.Queue()
    q.put((func_name, args, kwargs, result_queue))
    return result_queue.get()  # Block until result logic completes

# Consumer: The worker thread processes requests at a controlled pace
def _worker(self, task_queue, interval):
    while True:
        task = task_queue.get()
        # ... execute API call ...
        time.sleep(interval)  # Enforce rate limit (e.g., 0.2s)
```

**Why it's elegant**:
*   **Serialized Execution**: Even if 50 threads trigger `place_order` simultaneously, the queue serializes them, ensuring the system never exceeds the N req/sec limit.
*   **Prioritization**: By separating "Quotes" into their own channel, a heavy data-fetch operation never blocks a critical trade execution.

---

## Pattern 3: Read-Through Cache via Decorator

Fetching the live price (LTP) is the most frequent operation. However, calling the API for every single position check is inefficient and quickly exhausts quotas.

I implemented a `QuoteMaintainer` class ensuring I respect the "Single Responsibility Principle". It acts as a **Read-Through Cache**.

**The Logic**:
1.  **Check Cache**: Is the price for `INFY` already in memory and fresh (TTL < 5s)?
2.  **On Miss**: If not, fetch quotes for *all* tracked instruments in a single batch call.
3.  **Update**: Populate the cache and return the value.

```python
class QuoteMaintainer:
    def get_quote(self, instrument):
        # 1. Check in-memory TTL cache
        if instrument in self._quote_cache:
            return self._quote_cache[instrument]
            
        # 2. Fetch fresh data (Read-Through)
        updated_quotes = self._kite.quote(self.instruments)
        self._quote_cache.update(updated_quotes)
        
        return updated_quotes[instrument]
```

**Why it's elegant**:
*   **Batching**: It automatically converts N individual requests into 1 batch request.
*   **Abstraction**: The strategy simply asks "What is the price?", completely agnostic to whether it came from RAM or the network.

---

## Conclusion

By applying standard Low-Level Design patterns, I solved the rate-limiting problem without complicating the business logic.

*   **Proxy** allowed me to inject middleware without changing client code.
*   **Producer-Consumer** queues turned chaotic concurrency into ordered sequential execution.
*   **Caching** reduced API load by an order of magnitude.

The result is a system that abstracts away the complexity of rate limiting, network calls and flow control from the core model and business logic.
