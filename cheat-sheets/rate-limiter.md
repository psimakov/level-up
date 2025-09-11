<cheat-sheet>

# **Cheat-Sheet: Design a Rate Limiter**

This document is a `<cheat-sheet />` for the `<coach />` to conduct an interview on designing a distributed rate limiter.

---

## **1. Problem Statement**

Present the candidate with a vague, high-level problem:

> "Design a rate limiter to protect our backend services from being overwhelmed by too many requests. The rate limiter will be used for a large-scale social media application."

The candidate's first task is to ask clarifying questions to establish the scope and requirements.

---

## **2. Interview Framework & Guiding Questions**

### **Step 1: Requirements & Scope**

The candidate must define the functional and non-functional requirements.

*   **Functional Requirements:**
    *   **Identify Clients:** How do we identify who is making the request? (IP address for anonymous users, user ID for authenticated users, API key for developers).
    *   **Limit Requests:** The core functionality. The system must limit requests based on configurable rules (e.g., 100 requests per minute per user).
    *   **Return Proper Errors:** When a user is rate-limited, the system should return a clear error code (`429 Too Many Requests`) and helpful headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`).

*   **Non-Functional Requirements:**
    *   **Scalability:** The system must handle a large volume of requests. Push the candidate for a specific number. (e.g., **1 million requests per second (RPS)** for a social media app with 100 million DAU).
    *   **Low Latency:** The rate limiter should add minimal overhead to the underlying request. A good target is **<10ms** of added latency.
    *   **High Availability:** The system should be fault-tolerant. If the rate limiter goes down, what happens? (Discuss fail-open vs. fail-close). The system should favor availability over strict consistency.
    *   **Accuracy:** The rate limiting should be reasonably accurate. Perfect accuracy is not required; a small margin of error is acceptable.

### **Step 2: High-Level Design & Placement**

The candidate should propose a high-level architecture. A key discussion point is **where the rate limiter should live**.

*   **Option 1: In the Microservice (Red Flag):**
    *   *Why it's bad:* No global state. A user could bypass the limit by hitting different services. Fast but incorrect.
*   **Option 2: As a Separate Service:**
    *   *Trade-offs:* Centralizes the logic and state, which is good. However, it adds a network hop for every single request, increasing latency.
*   **Option 3: At the API Gateway / Edge (Baseline Solution):**
    *   *Why it's best:* It's the first point of entry for all requests. It can reject requests early without overwhelming downstream services. This is the most common and effective placement.

### **Step 3: Deep Dive - Core Algorithm**

This is the heart of the interview. The candidate should discuss different algorithms and their trade-offs.

*   **Fixed Window Counter:**
    *   *How it works:* A counter for each user in a fixed time window (e.g., 100 requests per minute). The counter resets at the start of each minute.
    *   *Pros:* Simple to implement, low memory footprint.
    *   *Cons:* **Boundary effect**. A user can send a burst of 2x the limit by sending requests at the end of one window and the beginning of the next.

*   **Sliding Window Log:**
    *   *How it works:* Stores a timestamp for each request in a list/log. When a new request comes in, it discards all timestamps older than the window and counts the remaining ones.
    *   *Pros:* Perfectly accurate, solves the boundary effect.
    *   *Cons:* High memory usage. Storing every timestamp for every user is not scalable.

*   **Sliding Window Counter:**
    *   *How it works:* A hybrid approach. It keeps a counter for the current and previous window and uses a weighted sum to approximate the count in the sliding window.
    *   *Pros:* Good balance between accuracy and memory efficiency.
    *   *Cons:* It's an approximation and can be complex to implement correctly.

*   **Token Bucket (Baseline Solution):**
    *   *How it works:* Each user has a "bucket" with a certain capacity (burst limit) and a refill rate. A request consumes one token. If the bucket is empty, the request is rejected.
    *   *Pros:* Handles bursts naturally, memory efficient (only need to store token count and last refill time), and is widely used in practice.
    *   *Cons:* Requires careful tuning of bucket size and refill rate.

### **Step 4: Deep Dive - Data Storage & Scalability**

Once the algorithm is chosen (likely Token Bucket), ask how to store the state (token counts, timestamps).

*   **Storage:** An in-memory cache like **Redis** is the ideal choice due to its speed.
*   **Race Conditions:** What happens if a user sends two requests at the same time that hit different gateway instances? Both might read the same token count, and both might be allowed, violating the limit.
    *   *Solution:* Use atomic operations. Redis provides **Lua scripting** to perform the read-modify-write cycle as a single, atomic transaction.
*   **Scaling:** A single Redis instance cannot handle 1 million RPS.
    *   *Solution:* **Sharding**. Distribute the user keys across multiple Redis nodes. The candidate should be able to explain how to shard (e.g., consistent hashing on the user ID) and that **Redis Cluster** handles this automatically.

### **Step 5: Deep Dive - Availability & Fault Tolerance**

*   **Failure Modes:** What happens if a Redis shard goes down?
    *   **Fail-Open:** Allow all traffic through. Risks overwhelming backend services.
    *   **Fail-Close:** Reject all traffic. Protects the backend but causes an outage for users hitting that shard.
    *   *Discussion:* The candidate should discuss the trade-offs. For a social media app, `fail-close` is often preferred to prevent a cascading failure, but it's a business decision.
*   **High Availability Solution:**
    *   **Replication:** Use **leader-follower replication** for each Redis shard. If a primary node fails, a replica can be promoted to take its place, minimizing downtime.

---

## **3. Acceptable Solutions**

*   **Expected Baseline:**
    *   **Placement:** Rate limiter at the API Gateway.
    *   **Algorithm:** Token Bucket.
    *   **Storage:** A sharded Redis cluster for storing token counts and timestamps.
    *   **Atomicity:** Use of Lua scripts or transactions to prevent race conditions.
    *   **Availability:** Leader-follower replication for Redis shards.

*   **Acceptable Variants:**
    *   **Algorithm:** Sliding Window Counter is also a very strong choice if the candidate can justify the trade-offs.
    *   **Storage:** Other in-memory stores like Memcached could be used, but the candidate should acknowledge the lack of built-in data structures and scripting compared to Redis.

*   **Red-Flag Approaches:**
    *   Placing the rate limiter in the application microservices.
    *   Using a disk-based database (like PostgreSQL or MySQL) for the counters (too slow).
    *   Ignoring race conditions.
    *   Failing to address the single point of failure of a single Redis instance.

---

## **4. Common Mistakes & Probing Questions**

*   **Mistake: Not clarifying requirements.**
    *   *Probe:* "What kind of scale are we talking about? How many requests per second?"
*   **Mistake: Choosing a naive algorithm without discussing trade-offs.**
    *   *Probe:* "You chose the Fixed Window algorithm. What happens if a user sends a lot of requests right at the boundary of a window?"
*   **Mistake: Forgetting about race conditions.**
    *   *Probe:* "Imagine we have multiple instances of our gateway. What could go wrong if a user sends two requests at the exact same time?"
*   **Mistake: Designing for a single server.**
    *   *Probe:* "Our Redis server can only handle 100k requests per second, but we need to handle 1 million. How can we scale it?"
*   **Mistake: Not considering fault tolerance.**
    *   *Probe:* "What happens if the Redis instance holding a user's data crashes? Should we let their requests through or block them?"

</cheat-sheet>
