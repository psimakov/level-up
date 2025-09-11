<cheat-sheet>

# Cheat Sheet: Design a URL Shortener (like Bit.ly)

This cheat sheet provides a structured guide for an interviewer assessing a candidate on the URL shortener system design problem. It is derived from the expert `<challenge />` materials.

---

## Interviewer Guide

Your goal is to assess the candidate's ability to navigate a classic system design problem from requirements to a scalable, production-ready architecture.

**What to look for:**
*   **Structured Approach:** Does the candidate follow a logical framework (e.g., requirements, high-level design, deep dives)?
*   **Clarity on Requirements:** Do they clarify both functional and non-functional requirements before designing? Do they ask clarifying questions about scale?
*   **Trade-off Analysis:** Can the candidate discuss the pros and cons of different approaches (e.g., short code generation, caching strategies, redirect types)?
*   **Pragmatism:** Do they avoid premature optimization or unnecessary complexity? For example, do they correctly identify that the database might not need complex sharding for this use case?
*   **Core Concepts:** Do they demonstrate understanding of key system design concepts like caching, load balancing, database indexing, and stateless services?

---

## Phase 1: Requirements Gathering

A strong candidate will start by defining the problem space.

### Functional Requirements (Core Features)
*   **URL Shortening:** Users can submit a long URL and receive a short URL.
*   **URL Redirection:** Users accessing a short URL are redirected to the original long URL.
*   **Custom Aliases (Optional):** Users can suggest a custom short name (e.g., `bit.ly/my-name`).
*   **Expiration (Optional):** Short URLs can have an expiration time.

### Non-Functional Requirements (System Qualities)
*   **Low Latency:** Redirects must be fast (e.g., under 200ms) as they are user-facing.
*   **High Availability:** The service must be resilient to failures. If a server goes down, users should still be able to create and use short URLs.
*   **Scalability:** The system must handle a large number of users and URLs (e.g., 100 million daily active users, 1 billion total URLs).
*   **Uniqueness:** Every short URL must be unique and map to exactly one long URL.
*   **Consistency Model:** The system does not require strong read-after-write consistency. Eventual consistency is acceptable, making the system favor **Availability** over Consistency (in the CAP theorem sense).

### Common Pitfall: Premature Calculations
Candidates might immediately start doing "back-of-the-envelope" calculations for storage and traffic. A better approach, as highlighted in the challenge, is to state that such calculations should be done *when they are needed to make a design decision*, not just for the sake of it.

---

## Phase 2: High-Level Design & API

### Core Entities & Data Model
The candidate should identify the main entities. A simple, unified `URL` table is sufficient.

**URL Table:**
*   `short_url` (string, Primary Key): The generated unique short code.
*   `long_url` (string): The original URL.
*   `user_id` (string, Foreign Key): The user who created the link.
*   `creation_time` (timestamp): When the link was created.
*   `expiration_time` (timestamp, nullable): When the link should expire.

### API Endpoints
The API should map directly to the functional requirements.

1.  **Create Short URL:**
    *   `POST /api/v1/urls`
    *   **Body:**
        ```json
        {
          "long_url": "https://...",
          "custom_alias": "my-alias", // optional
          "expiration_time": "2025-12-31T23:59:59Z" // optional
        }
        ```
    *   **Response (Success):**
        ```json
        {
          "short_url": "http://short.ly/aB1x9Z"
        }
        ```

2.  **Redirect URL:**
    *   `GET /{short_code}`
    *   **Response:** A `302 Found` redirect to the `long_url`.

### Initial Architecture
A simple, non-scalable design is a great starting point.
*   **Client:** A web browser or mobile app.
*   **Server:** A single application server handling API requests.
*   **Database:** A single relational database (e.g., PostgreSQL) storing the URL table.

---

## Phase 3: Deep Dives

This is where the candidate evolves the simple design to meet the non-functional requirements.

### Deep Dive 1: Uniqueness of Short Codes

This is the core of the problem. How do you generate a short, unique code?

*   **Bad Idea: Prefix of Long URL.** Fails uniqueness due to common domains.
*   **Option 1: Random Number Generator + Base62 Encoding.**
    *   **How:** Generate a large random number and convert it to a Base62 string (`[a-zA-Z0-9]`). A 6-character string gives ~56 billion possibilities.
    *   **Problem:** High probability of collisions (see "Birthday Paradox").
    *   **Solution:** Requires checking the database for existence before writing, adding a read operation to the write path.
*   **Option 2: Hash of Long URL.**
    *   **How:** Hash the long URL (e.g., MD5, MurmurHash), Base62 encode it, and take the first 6-7 characters.
    *   **Problem:** Same collision issue as the random number generator. Also requires a DB check.
*   **Option 3 (Recommended): Global Counter.**
    *   **How:** Use a distributed, atomic counter (like one in Redis). Increment the counter for each new URL, and Base62 encode the resulting number.
    *   **Pros:** **Guaranteed uniqueness**. No need to check the database for collisions, resulting in a faster write path.
    *   **Security Con:** The resulting URLs are predictable (`/1`, `/2`, etc., when decoded). This allows competitors to scrape data.
    *   **Security Solution:** Use a **bijective function** (e.g., using a library like `squids.java`) to obfuscate the counter's output into a random-looking but still unique string.

### Deep Dive 2: Low Latency on Redirects

*   **Database Indexing:** The `short_url` column must be the **primary key**. This creates a B-Tree index, making lookups extremely fast (logarithmic time).
*   **Caching Layer:**
    *   Introduce an in-memory cache (like Redis or Memcached) to store `short_url` -> `long_url` mappings.
    *   **Strategy:** Use a **Read-Through, Least Recently Used (LRU)** policy.
        1.  Client requests a short URL.
        2.  Service checks the cache.
        3.  **Cache Hit:** Return the long URL immediately (very fast).
        4.  **Cache Miss:** Fetch from the database, populate the cache, and then return.
*   **Trade-off: 301 vs. 302 Redirects.**
    *   **301 (Permanent):** Can be heavily cached by browsers and CDNs. This reduces load on our servers but means we lose analytics/visibility into redirect traffic.
    *   **302 (Temporary):** Forces the client to always check with our server. This is better for collecting analytics and detecting if the service is down, at the cost of higher server load. **302 is generally preferred.**

### Deep Dive 3: Scalability

*   **Application Servers:**
    *   The single server is a bottleneck. **Horizontally scale** the application servers and place them behind a **Load Balancer**.
    *   The servers must be **stateless**.
    *   (Advanced) Separate into **Read Service** (for redirects) and **Write Service** (for creation), which can be scaled independently. An **API Gateway** would route requests appropriately.
*   **Global Counter:** The counter cannot live on a single application server. It must be moved to a centralized, distributed store like **Redis** (using its `INCR` command for atomicity).
*   **Database:**
    *   A quick calculation shows that 1 billion rows at ~500 bytes/row is ~500 GB. This fits on a single modern database server.
    *   The primary bottleneck would be read throughput, but the caching layer handles most of that.
    *   Therefore, **sharding the database is likely not necessary** initially. A single primary with read replicas would be sufficient for availability.

### Deep Dive 4: High Availability

*   **Services:** Running multiple instances of each service behind a load balancer already provides high availability.
*   **Cache:** If the Redis cache goes down, the system degrades gracefully (slower lookups from the DB) but doesn't fail. Redis can also be set up in a high-availability cluster.
*   **Database:** Use a primary-replica setup. If the primary DB fails, a replica can be promoted to primary. Regular snapshots to a blob store (like S3) should be taken for disaster recovery.

---

## Final Architecture

The final design should look something like this:
`Client -> API Gateway -> [Read Service Cluster | Write Service Cluster]`
*   **Read Service:** `-> Redis Cache -> (on miss) -> Database`
*   **Write Service:** `-> Redis Counter -> Database`

This architecture is scalable, highly available, and low-latency for the critical redirect path.

</cheat-sheet>