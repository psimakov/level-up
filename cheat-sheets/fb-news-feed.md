<cheat-sheet>

# Cheat Sheet: Design Facebook News Feed

## 1. Problem Statement & Scope

The goal is to design Facebook's News Feed, showing recent stories from users in one's social graph. The focus is on uni-directional "follow" relationships and a chronologically ordered feed.

### Functional Requirements

*   **Core:**
    *   Users can create posts.
    *   Users can follow other users.
    *   Users can view a chronological feed of posts from people they follow.
    *   Users can page through their feed (infinite scroll).
*   **Out of Scope:**
    *   Likes and comments.
    *   Private or restricted visibility posts.
    *   User authentication (assume it exists).

### Non-Functional Requirements

*   **High Availability:** Prioritize availability over strong consistency.
*   **Eventual Consistency:** Tolerate up to 1 minute of post staleness.
*   **Low Latency:** Posting and viewing the feed should be fast (< 500ms).
*   **Scalability:** Handle a massive user base (e.g., 2 billion users) with unlimited follows/followers.

## 2. Core Entities & Data Model

*   **User:** A user in the system.
*   **Post:** Content created by a user.
    *   `postID` (PK)
    *   `creatorID`
    *   `content`
    *   `createdAt` (Timestamp)
*   **Follow:** A uni-directional link between two users.
    *   `userFollowing` (Partition Key)
    *   `userFollowed` (Sort Key)

## 3. API Design

*   **Create Post:** `POST /posts`
    *   Body: `{ "content": { ... } }`
    *   Response: `200 OK` with `{ "postId": "..." }`
*   **Follow User:** `PUT /users/[id]/followers`
    *   Idempotent action to create a follow relationship.
*   **Get Feed:** `GET /feed?pageSize={size}&cursor={timestamp}`
    *   Response: `{ "items": Post[], "nextCursor": "..." }`
    *   Cursor-based pagination using the timestamp of the oldest post seen.

## 4. High-Level Design & Solution Walkthrough

### Initial (Naive) Approach

This approach works for a small scale but has significant performance bottlenecks that need to be addressed.

1.  **Services:**
    *   **Post Service:** Handles post creation, writing to a `Posts` table.
    *   **Follow Service:** Manages follow relationships, writing to a `Follows` table.
    *   **Feed Service (Read-Heavy):** Generates the user's feed.
2.  **Databases (DynamoDB / Cassandra):**
    *   **Posts Table:**
        *   Primary Key: `postID`.
        *   GSI (Global Secondary Index): `creatorID` (Partition Key), `createdAt` (Sort Key). This allows efficiently fetching all posts by a specific user, sorted by time.
    *   **Follows Table:**
        *   Primary Key: `userFollowing` (PK), `userFollowed` (SK). Allows fetching everyone a user follows.
        *   GSI: `userFollowed` (PK), `userFollowing` (SK). Allows fetching all followers of a user.
3.  **Feed Generation (on-read):**
    *   For a given user, the **Feed Service** first queries the `Follows` table to get all followed user IDs.
    *   Then, for each followed user ID, it queries the `Posts` table GSI to get their recent posts.
    *   Finally, it merges all these posts, sorts them by `createdAt`, and returns the result.

### Identifying Bottlenecks (The Fan-Out Problem)

*   **High Follow Count:** If a user follows thousands of people, the feed service makes thousands of database calls, which is slow and resource-intensive ("fan-out on read").
*   **High Follower Count:** If a user with millions of followers creates a post, updating all their feeds becomes a challenge ("fan-out on write").

### Scalable (Hybrid) Approach

This approach pre-computes feeds for most users to optimize read latency and uses the naive approach as a fallback for specific cases.

1.  **Precomputed Feeds Table:**
    *   A new table (`PrecomputedFeed`) stores a ready-made feed for each user.
    *   Partition Key: `userId`.
    *   Value: A list of the most recent ~200 `postIDs`.
    *   **Storage Check:** 2B users * 200 postIDs/user * 10 bytes/postID ≈ 4TB. This is reasonable.

2.  **Async Feed Fan-out on Write:**
    *   When a user creates a post, the **Post Service** sends a message to a queue (e.g., SQS, Kafka).
    *   A pool of **Feed Workers** consumes from the queue.
    *   For each new post, a worker fetches the creator's followers from the `Follows` table GSI.
    *   The worker then injects the new `postID` into the `PrecomputedFeed` list for each follower.
    *   For creators with millions of followers, this single job can be broken down into smaller sub-jobs and re-queued to distribute the load.

3.  **Hybrid Strategy for Mega-Accounts (e.g., celebrities):**
    *   Writing to millions of feeds for a single post is inefficient.
    *   **Solution:** Don't precompute feeds for followers of mega-accounts.
    *   When a user requests their feed, the **Feed Service** will:
        1.  Fetch the user's precomputed feed from the `PrecomputedFeed` table.
        2.  Separately, fetch posts from the few mega-accounts the user follows (using the initial on-read approach).
        3.  Merge the two lists at runtime.
    *   This caps the number of writes on post creation and limits the on-read work to a small, manageable number of high-follower accounts.

## 5. Deep Dive: Common Problems & Pitfalls

### Hotkeys / Hot Shards

*   **Problem:** A viral post from a popular account will be requested by millions of users simultaneously, overwhelming the database partition (or cache shard) holding that `postID`.
*   **Incorrect Solution:** A standard distributed cache (like Redis) sharded by `postID`. This simply moves the hotkey problem from the database to the cache.
*   **Correct Solution:**
    1.  Use a cache layer (e.g., Redis, Memcached) to shield the database.
    2.  Instead of a single sharded cache, use **multiple independent cache replicas**.
    3.  The **Feed Service** randomly selects a cache replica to query.
    4.  This distributes the read load for a single hotkey across `N` cache instances. If `N=10`, only 1/10th of the requests result in a cache miss that hits the database, effectively solving the hotkey issue.

### Common Candidate Mistakes

*   **Getting Lost in Micro-Optimizations:** Trying to perfect a small part of the system before having a working, end-to-end high-level design. It's better to start with a simple (even if not scalable) design and iterate.
*   **Over-engineering:** Choosing complex technologies like graph databases when a simpler key-value store with indexes is sufficient for the stated requirements.
*   **Not Quantifying:** Failing to put numbers on non-functional requirements (e.g., "fast" vs. "<500ms") or doing back-of-the-envelope calculations for storage/traffic.
*   **Not Proactively Identifying Bottlenecks:** A key senior-level signal is the ability to identify the "fan-out" and "hotkey" problems in the initial design and propose solutions.

## 6. Interviewer's Guide: Leveling Expectations

*   **Mid-Level (E4):**
    *   Focus on breadth. Should define APIs, data models, and a functional high-level design.
    *   May identify some scaling issues but won't be expected to solve all of them proactively. The interviewer may need to guide them toward the deep dives.
*   **Senior (E5):**
    *   Balances breadth and depth. Should quickly produce the high-level design.
    *   **Must proactively identify and discuss solutions for at least two deep dives** (e.g., fan-out on read/write, hotkeys).
    *   Should clearly articulate trade-offs (e.g., on-read vs. on-write computation, hybrid model).
*   **Staff+ (E6+):**
    *   Focus on depth and experience. Should drive the conversation, identifying and solving multiple bottlenecks with practical, experience-backed solutions.
    *   The interviewer should only need to focus the conversation, not steer it.
    *   Expected to discuss nuances like sharding strategies, cache eviction policies, and potential product limitations (e.g., capping follows) as a valid engineering strategy.

</cheat-sheet>