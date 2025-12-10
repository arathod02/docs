---
slug: offline-online-indicator
title: Design an Online/Offline Indicator (Presence Service)
sidebar_label: Presence Service
description: Designing a scalable presence system for 1 billion users.
tags: [system-design, distributed-systems, websockets, redis]
---

Designing a system to indicate if a user is **Online** or **Offline** (and their last seen timestamp) sounds simple on the surface, but becomes a massive engineering challenge when scaling to **1 Billion users**.

In this post, we will breakdown the design of a Presence Service suitable for a massive social network or chat application like WhatsApp or Facebook Messenger.

## Problem Statement

We need to build a service that indicates the availability status of a user's connections.

**The Functional Requirements:**
1.  Indicate if a friend/connection is currently **Online**.
2.  If offline, display the **Last Seen** timestamp.

**The Scale:**
* **Total Users:** ~1 Billion.
* **Concurrent Online Users:** ~500 Million.
* **Latency:** The status needs to be near real-time.

## High-Level Strategy: Push vs. Pull

How do we keep the server updated about the client's status?

### Option 1: Pull (Polling)
The server periodically connects to the client to ask, "Are you there?"
* **Verdict:** ❌ **Impossible.**
* **Reasoning:** In a mobile/NAT environment, servers cannot initiate connections to clients easily. Furthermore, polling 1B users is computationally wasteful.

### Option 2: Push (Heartbeat)
The client sends a signal to the server periodically saying, "I am alive."
* **Verdict:** ✅ **Selected.**
* **Reasoning:** This is the standard pattern for presence. If the server stops receiving heartbeats (after a timeout threshold), the user is marked offline.

---

## The Protocol: HTTP vs. WebSockets

We established a "Push" model, but how should the client push this data?

### 1. REST/HTTP (The Polling Approach)
The client sends a `POST /health` request every $N$ seconds.

**Why this fails at scale:**
* **Overhead:** HTTP is stateless. Every heartbeat requires a full 3-way TCP handshake (if not using keep-alive efficiently), SSL handshake overhead, and heavy HTTP headers.
* **Traffic:** With 500M concurrent users sending a heartbeat every 10 seconds, that is **50 Million requests per second**. Most of the data transferred would be HTTP headers, not the actual status payload.

### 2. Persistent WebSockets
The client opens a long-lived, bi-directional connection with the server.

**Why this is the winner:**
* **Reduced Overhead:** Once the connection is established, data frames have minimal overhead (just a few bytes). There are no repeated headers or handshakes.
* **Real-time:** The server knows *immediately* if a connection is severed (TCP FIN or RST).
* **Bi-directional:** It allows the server to push status updates of *friends* back to the user over the same channel.

:::caution The "Disconnect" Fallacy
A common misconception is that WebSockets natively handle all disconnects via an `onDisconnect` event.
* **Clean Disconnect:** If a user clicks "Logout", the client sends a TCP FIN. The server knows immediately.
* **Dirty Disconnect:** If the user loses 4G signal or their battery dies, **the server receives nothing.**
* **The Fix:** We must implement an **Application-Level Heartbeat**. If the server doesn't receive a "Ping" frame or message within $N$ seconds, it forcibly closes the socket and marks the user offline.
  :::

:::info Scaling WebSockets
Scaling persistent connections is harder than scaling stateless HTTP.
1.  **OS Limits:** You must tune the kernel to allow >65k open file descriptors (ephemeral ports) per server.
2.  **Load Balancing:** You need a Layer 7 Load Balancer that supports "Sticky Sessions" effectively, though for a pure presence service, state can be externalized.
3.  **Memory:** Holding 500M open connections requires massive RAM across your fleet of connection handlers (Gateway Service).
    :::

---

## Database Design & Estimation

The compute layer is simple (Gateway servers holding connections). The complexity lies in the storage layer.

### Query Patterns
1.  **Write (Heavy):** Update User $A$'s timestamp (Heartbeat).
2.  **Read (Heavy):** Get status for User $A$ (and their 500 friends) when User $B$ opens the app.

### Data Schema
We need a simple Key-Value pair.

| Field | Type | Size |
| :--- | :--- | :--- |
| `UserID` | Integer | 4 Bytes |
| `LastSeen` | Epoch (Int) | 4 Bytes |
| **Total** | | **8 Bytes** |

### Capacity Planning
With 1 Billion users, do we need massive storage?

$$
1,000,000,000 \text{ users} \times 8 \text{ bytes} \approx 8 \text{ GB}
$$

:::tip Epiphany
We only need **~8 GB** of storage to hold the state of every user on the planet. This entire dataset can fit into the RAM of a single modern server instance.
:::

---

## Managing "Online" State Lifecycle

How do we decide when to switch a user from Online to Offline? We have three strategies.

### 1. The Cron Job Reaper
A background process scans the database every few minutes and deletes entries older than $N$ minutes.
* **Pros:** Keeps DB clean eventually.
* **Cons:** **Terrible at scale.** Scanning a table of 500M rows every minute creates massive read pressure and locking issues. The "Offline" status will always be laggy.

### 2. Connection Events (Explicit Disconnect)
Leverage WebSocket callbacks (`onConnect`, `onDisconnect`) to update the DB.
* **Pros:** extremely efficient. Writes only happen on state changes.
* **Cons:** Unreliable. If a user loses network (enters a tunnel) or the app crashes, the `onDisconnect` event might never fire sent to the server. The user will appear "Online" forever (a Zombie session).

### 3. Database TTL (Time-To-Live)
Use the database's native feature to auto-expire keys. The Heartbeat simply resets the TTL.
* **Pros:** Handles "unclean" disconnects gracefully. If the heartbeat stops, the key vanishes automatically. No manual cleanup required.
* **Cons:** Moderate write load (every heartbeat is a write to reset the TTL).

**Verdict:** We will use **Option 3 (TTL)** as the primary mechanism, potentially optimized by Option 2 (explicitly deleting the key on a clean logout to avoid the TTL wait).

---

## Technology Selection: Redis vs. DynamoDB

We need a Key-Value store that handles massive write throughput.

**The Math:**
* 500 Million concurrent users.
* Heartbeat interval: 30 seconds.
* Throughput = $500,000,000 / 30 \approx$ **16.6 Million Writes/Second**.

### Candidate 1: Amazon DynamoDB
* **Pros:** Serverless, high durability, multi-region replication (Global Tables).
* **Cons:** **Cost and Hot Partitions.**
    * Cost: DynamoDB charges by **Write Capacity Units (WCUs)**.
      * 16.6 Million writes/sec = **16.6 Million WCUs**.
      * Cost per WCU (Provisioned) $\approx \$0.00065$ / hour.
      * **Hourly Cost:** $\$10,833$.
      * **Monthly Cost:** **~$7.9 Million / Month**.
    * Hot Partition: In DynamoDB, a single partition is strictly limited to **1,000 WCUs**. If 2,000 users map to the same partition key, requests get throttled.

### Candidate 2: Redis (The Winner)
Redis is an in-memory store. We are limited by CPU/Network throughput per node.
* **Pros:**
    * **In-Memory Speed:** Sub-millisecond reads/writes.
    * **Native TTL:** Redis handles key expiration natively and efficiently.
    * **Cost Effective:**
      * Redis is an in-memory store. We are limited by CPU/Network throughput per node.
      * A single robust Redis node (e.g., AWS `r7g.xlarge`) can handle **~600,000 writes/sec**. (**Benchmark**: https://aws.plainenglish.io/aws-elasticache-a-performance-and-cost-analysis-of-redis-7-1-vs-valkey-7-2-bfac4fb5c22a)
      * Nodes required: $16,600,000 / 600,000 \approx$ **28 Shards**.
      * Cost per node $\approx \$0.30$ / hour.
      * **Monthly Cost:** $28 \times \$0.30 \times 730 \text{ hours} \approx$ **~$6132 / Month**.
* **Cons:**
    * **Persistence:** If Redis crashes, we lose "Last Seen" data (unless AOF mode is enabled, which slows performance).
* **Mitigation:** For a Presence system, *ephemeral* data loss is acceptable. If Redis crashes, users briefly appear offline until their next heartbeat (seconds later).

:::tip Cost Epiphany
By choosing Redis over DynamoDB for this high-throughput/ephemeral workload, we save the company roughly **$7.89 Million per month**.
:::

## Final Architecture

```mermaid
flowchart TD
    Client[User Client] -->|WebSocket Connection| LB[Load Balancer]
    LB -->|Sticky Session| WS[WebSocket Server]
    
    subgraph Data Layer
    WS -->|HEARTBEAT every 10s| Redis[(Redis Cluster)]
    end
    
    note[Redis Key: UserID <br/> Value: Timestamp <br/> TTL: 30s] -.-> Redis