# System Design Glossary

You often understand an idea but don't know its name. Use the **quick lookup** first: find the situation as you'd describe it, get the term, then read the details below.

## Quick lookup: "I'd describe it as…" → term

| You'd describe it as… | It's called |
|---|---|
| Many requests want the same missing data → let **one** fetch it, the rest wait and share its answer | **Request coalescing** |
| A popular cache entry expires and **everyone hits the DB at once** | **Cache stampede / thundering herd** |
| Retry later, waiting **longer each time** (1 s, 2 s, 4 s…) | **Exponential backoff** |
| Add a **little randomness** to wait times so retries don't all happen together | **Jitter** |
| Failed clients all retrying together and making the outage worse | **Retry storm** |
| Provider keeps failing → **stop calling it for a while**, then test it again | **Circuit breaker** |
| Doing the same thing twice has the **same result as doing it once** | **Idempotency** |
| A unique ID sent with a request so a **repeat is recognised and ignored** | **Idempotency key** |
| A message might be delivered **more than once**, but never lost | **At-least-once delivery** |
| Two processes **read the same value at once** and both act on it (both see 99) | **Race condition** (check-then-act) |
| An operation that **can't be interrupted halfway** (read + change + write as one) | **Atomic operation** |
| Copies are briefly out of date but **catch up a moment later** | **Eventual consistency** |
| The replica **hasn't received the latest write yet** | **Replication lag** |
| **Copy** all the data to more machines | **Replication** |
| **Split** the data across machines | **Sharding / partitioning** |
| A way to split data so that **adding a machine moves only a little data** | **Consistent hashing** |
| A sorted lookup structure so the DB **jumps to matching rows instead of scanning every row** | **Database index** (B-tree) |
| A middle table linking two tables (users ↔ groups), one row per pair, holding **data about the pair** (role, joined_at) | **Junction table** (many-to-many) |
| A backup machine **takes over** when the main one dies | **Failover** |
| **One box** whose failure takes everything down | **Single point of failure (SPOF)** |
| Server keeps **no memory of the user** between requests, so any server can handle any request | **Stateless** |
| Add **more machines** vs make **one machine bigger** | **Horizontal vs vertical scaling** |
| One key/user/link gets **far more traffic** than the rest | **Hot key / hot partition** |
| App checks the cache, on a miss reads the DB **and fills the cache** | **Cache-aside** |
| Write to the DB **and the cache at the same time** | **Write-through** |
| Write to the cache now, **save to the DB later in batches** | **Write-back (write-behind)** |
| The cache holds an **old copy** after the real data changed | **Stale data / cache invalidation problem** |
| Pre-loading the cache **before** users ask | **Cache warming** |
| Serve the old copy **now** and refresh it **in the background** | **Stale-while-revalidate** |
| Limit how many requests a client can send per minute | **Rate limiting** |
| A bucket of tokens that **refills steadily**; each request uses one | **Token bucket** |
| 100 requests at 12:00:59 + 100 at 12:01:00 = 200 in 2 seconds | **Boundary burst** (fixed-window flaw) |
| Limiter is down → **let everyone through** / **block everyone** | **Fail open / fail closed** |
| Consumers are slower than producers → **tell producers to slow down** | **Backpressure** |
| Overloaded → **deliberately drop low-priority work** to save the rest | **Load shedding** |
| Keep the core working when part of the system fails (show **less** rather than nothing) | **Graceful degradation** |
| One heavy user/job **slows everyone sharing** the same resource | **Noisy neighbour** |
| Give each part **its own resources** so one failure can't sink the others | **Bulkhead pattern** |
| Hand work to a queue and **reply immediately**; do the work later | **Asynchronous processing** |
| Where messages go **after all retries fail** | **Dead letter queue (DLQ)** |
| A message that **always fails**, however often it's retried | **Poison message** |
| One event → **many** independent receivers | **Fan-out** |
| Save to the DB and publish to the queue **without them getting out of sync** | **Transactional outbox** |
| Guessing other users' links by counting 1, 2, 3… | **Enumeration** |
| Trying thousands of passwords on `/login` | **Brute-force attack** |
| Always send the same user to the **same server / region** | **Sticky routing (session affinity)** |
| Users reach the **nearest server** automatically by the same IP address | **Anycast / geo-routing** |
| A lookup table of **which server each online user is connected to** | **Connection registry / session store** |
| Servers post to a named channel; whoever **subscribed** to it gets the message | **Pub/Sub (publish–subscribe)** |
| A small "I'm still alive" signal sent **every few seconds** | **Heartbeat** |

---

## Caching

### Cache hit / miss, hit ratio
*"Was it already in the cache?"*
- **Hit:** found in the cache (fast). **Miss:** not found, so fetch it from the origin or DB (slow).
- **Hit ratio** = hits ÷ total. A high ratio means the cache is doing its job (e.g. 95%).
- **Seen in:** [cdn.md](cdn.md), URL shortener (Step 7).

### TTL (time to live)
*"How long a copy may be kept before it's thrown away."*
- Set per cache entry or key (Redis `EXPIRE`, CDN `s-maxage`).
- Short TTL = fresher data but more misses; long TTL = faster but more stale.
- **Seen in:** CDN, URL shortener (expiry), rate limiter (counters reset automatically).

### Cache stampede / thundering herd
*"A popular entry expires and thousands of requests miss at once → all hit the DB."*
- **Problem:** the DB gets a sudden flood exactly when the cache stops protecting it.
- **Fixes:** **request coalescing** (below), a lock per key, refreshing the entry slightly before it expires, stale-while-revalidate, and adding jitter to TTLs so keys don't all expire together.
- **Seen in:** URL shortener (viral link), [cdn.md](cdn.md).

### Request coalescing
*"Hold all the identical requests, let only one through, then give its answer to everyone."*
- The first miss for a key fetches from the origin; later requests for the same key **wait** for that fetch instead of starting their own.
- How it's built: an in-memory map of `key → in-flight fetch`, which waiting requests subscribe to.
- **Also called:** collapsed forwarding (CDNs), `singleflight` (Go), dogpile prevention.
- **Seen in:** [cdn.md](cdn.md), URL shortener (viral link).

### Cache-aside (lazy loading)
*"App checks the cache; on a miss it reads the DB itself and fills the cache."*
- The app talks to both the cache and the DB; the cache never talks to the DB.
- Most common pattern. Only data that's actually requested gets cached.
- **Seen in:** URL shortener (diagram fix).

### Write-through
*"On every write, update the DB and the cache together."*
- The cache is always fresh; writes are slightly slower.
- **Seen in:** URL shortener (a link must be readable right after creation, despite replication lag).

### Write-back (write-behind)
*"Write to the fast store now, save to the DB later in batches."*
- Very fast writes, far fewer DB writes; you can lose recent data if the cache dies before flushing.
- **Seen in:** URL shortener click counts (Redis `INCR`, flushed to the DB every minute).

### Cache invalidation / stale data
*"The original changed but the cache still has the old copy."*
- **Fixes:** versioned names (`app.a8f3c.js`), purge through the CDN's API, shorter TTL, delete the cache entry on write.
- *"There are only two hard things in CS: cache invalidation and naming things."*
- **Seen in:** [cdn.md](cdn.md).

### Stale-while-revalidate
*"Serve the old copy instantly, refresh it in the background."*
- Users never wait for a refresh; data may be a few seconds old.
- **Seen in:** [cdn.md](cdn.md).

### Cache warming
*"Pre-load popular data so the first users don't get slow misses."*
- Used after deploys, cache restarts, or before a planned traffic spike (sale, campaign).

### Hot key / hot partition
*"One key gets massively more traffic than the others."*
- **Problem:** one Redis node or DB shard is overloaded while the others sit idle.
- **Fixes:** replicas for reads, a small in-memory cache on each app server, splitting the key into several copies.
- **Also called:** the celebrity problem.
- **Seen in:** URL shortener (viral link).

---

## Scaling and data

### Horizontal vs vertical scaling
*"More machines vs a bigger machine."*
- **Vertical (scale up):** more CPU/RAM on one machine. Simple, but there's a ceiling and it's still one box.
- **Horizontal (scale out):** more machines behind a load balancer. No ceiling, but needs stateless servers or shared state.

### Stateless
*"The server remembers nothing between requests."*
- Any server can handle any request, so you can add or remove servers freely.
- State lives somewhere shared (DB, Redis).
- **Seen in:** rate limiter (gateways stateless, counters in Redis).

### Replication (primary / replica)
*"Copy all the data to more machines."*
- **Primary** takes writes; **replicas** copy it and serve reads / stand by as backups.
- Solves **too many reads** and **machine failure**; doesn't solve "data too big".
- **Seen in:** URL shortener, rate limiter (Redis replicas as backups).

### Replication lag
*"The replica hasn't caught up with the latest write yet."*
- Replication is usually **asynchronous**, so a read from a replica right after a write may miss it.
- **Fixes:** read your own writes from the primary, write-through cache.
- **Seen in:** URL shortener (new link → 404), rate limiter (why counters are read from the primary only).

### Sharding / partitioning
*"Split the data so each machine holds only part of it."*
- A **shard key** decides where each row lives, e.g. `hash(code) % N`.
- Pick a key that every request already knows and that spreads the load evenly.
- Solves **data too big** and **too many writes**. Each shard usually still has its own replicas.
- **Seen in:** URL shortener (Step 7.3).

### Junction table (many-to-many)
*"A user can be in many groups and a group has many users, so put the link itself in its own table."*
- `group_members(group_id, user_id, role, joined_at, muted_until, ...)`: one row per user per group, primary key `(group_id, user_id)`.
- Holds data that belongs to **the pair**, not to either side (Bob is an admin in group A, a member in group B).
- Better than a member list inside the group row: per-member data has a home, "Bob's groups" is an index lookup, and concurrent adds don't overwrite each other.
- **Seen in:** chat app (groups).

### Database index (B-tree)
*"Like the index at the back of a book: look up the word, jump straight to the page."*
- A separate structure, kept **sorted** by the chosen column(s), that points to where each row is stored.
- No index → the DB scans every row: **O(n)**. B-tree index → a few hops down a tree: **O(log n)** (about 3–4 disk reads even for a billion rows).
- Hash index → **O(1)** for exact matches only (`id = 42`), no ranges or sorting.
- **Composite index** `(chat_id, seq)`: sorted by chat, then by seq, so "chat X, seq > 1041, latest 50" is one jump plus reading neighbouring rows.
- **Cost:** extra storage, and every write must update the index too. Index only the columns you actually query.
- **Seen in:** url-shortener (`short_code`), chat app (`(chat_id, seq)`).

### Consistent hashing
*"A way to assign keys to machines so adding or removing a machine moves only a small part of the data."*
- With plain `hash % N`, changing N moves almost every key; consistent hashing moves only about 1/N of them.
- Used by Cassandra, DynamoDB and Redis Cluster-style systems.

### Failover
*"A standby takes over when the main machine dies."*
- e.g. a Redis replica is promoted to primary within seconds; the last few writes may be lost.
- **Seen in:** rate limiter.

### Single point of failure (SPOF)
*"One component whose failure takes everything down."*
- **Fix:** redundancy, meaning at least two of everything important (load balancers, gateways, DB replicas), across availability zones.

### Region / availability zone (AZ)
*"Where the machines physically are."*
- **Region:** a cluster of data centers in one metro area serving a large part of the world (e.g. Mumbai, Ireland).
- **AZ:** separate buildings inside a region, a few km apart, with their own power and network; close enough for real-time sync.
- **Edge / PoP:** CDN locations in hundreds of cities, much closer to users than regions.
- **Seen in:** rate limiter (Step 7.4).

### Eventual consistency vs strong consistency
*"Copies may disagree for a moment but catch up."* vs *"Everyone always sees the latest value."*
- Eventual: faster, more available, slight staleness (e.g. rate-limit counts synced across regions every second).
- Strong: always correct, but slower (needs coordination) and less available during failures.

### Anycast / geo-routing
*"Users automatically reach the nearest location."*
- Anycast: many locations share **one IP address**; the internet routes each user to the closest one. GeoDNS: DNS returns a nearby location's IP.
- **Seen in:** [cdn.md](cdn.md).

### Sticky routing (session affinity)
*"Always send the same user to the same server or region."*
- Keeps per-user state in one place; the downside is uneven load, and that user is affected if that server fails.
- **Seen in:** rate limiter ("home region" option).

---

## Concurrency and correctness

### Race condition (check-then-act)
*"Two processes read the same value at the same moment and both act on it."*
- e.g. two gateways both read 99, both allow the request, both write 100 → 101 requests got through.
- e.g. two servers both check that a short code is free, and both insert it.
- **Fixes:** atomic operations, unique constraints, locks.
- **Seen in:** rate limiter (Step 7.2), URL shortener (collisions).

### Atomic operation
*"Read + change + write happen as one step that can't be interrupted."*
- Redis `INCR`, a Redis **Lua script**, a DB transaction, `INSERT` with a unique constraint.
- **Seen in:** rate limiter, URL shortener.

### Unique constraint
*"The DB itself refuses a duplicate."*
- Instead of "check, then insert" (racy), just insert, and handle the duplicate error by retrying or returning `409`.
- **Seen in:** URL shortener (collisions), message queues (idempotency).

### Idempotency / idempotency key
*"Doing it twice has the same effect as doing it once."*
- The caller or producer attaches a unique key (`order-4521-shipped`, `event_id`); the receiver records keys it has handled and ignores repeats.
- Needed because networks time out and clients and queues retry.
- **Seen in:** notification system (API + workers), [message-queues.md](message-queues.md).

### Delivery guarantees
*"How many times can a message be delivered?"*

| Guarantee | Meaning | Risk |
|---|---|---|
| At-most-once | 0 or 1 times | Messages can be **lost** |
| **At-least-once** | 1 or more times | **Duplicates** (fix with idempotency) |
| Exactly-once | Exactly 1 time | Generally impossible end to end when systems can crash |

- Real systems usually choose **at-least-once + idempotency**.
- **Seen in:** notification system, [message-queues.md](message-queues.md).

---

## Traffic control and failure handling

### Rate limiting
*"Cap how many requests a client may send in a period of time."*
- Rejected requests get `429 Too Many Requests` + `Retry-After`.
- Usually in an **API gateway** with counters in a shared **Redis**.
- **Seen in:** rate limiter exercise.

### Rate-limiting algorithms
| Name | One-line idea | Weak spot |
|---|---|---|
| Fixed window counter | Count per calendar minute | **Boundary burst**: up to 2× at the edge between windows |
| Sliding window log | Store every timestamp, count the last 60 s | Memory |
| Sliding window counter | `previous minute's count × overlap% + current count` | Approximate |
| **Token bucket** | Tokens refill steadily; each request uses one; bucket size = allowed burst | Most common choice |
| Leaky bucket | Queue drained at a fixed rate | Bursts wait or get dropped |

### Exponential backoff + jitter
*"After each failure, wait longer before retrying, plus a little randomness."*
- 1 s → 2 s → 4 s → 8 s…, each ± a random amount.
- Backoff gives the failing service time to recover; **jitter** stops thousands of clients retrying at the same instant.
- Set a **maximum** number of attempts; after that → DLQ or "failed".
- **Seen in:** notification system, [message-queues.md](message-queues.md).

### Retry storm
*"Every failed client retries together, so the struggling service gets even more load."*
- **Fixes:** backoff + jitter, a retry limit, circuit breaker.

### Circuit breaker
*"If a dependency keeps failing, stop calling it for a while, then test it with one request."*
- **Closed** (normal) → too many failures → **open** (fail immediately, don't call) → after a timeout → **half-open** (let one test request through) → success → closed again.
- Protects your threads and the struggling provider; callers get a fast error instead of a slow timeout.
- Fits the notification system: SendGrid down → open the circuit, leave messages in the queue.

### Fail open / fail closed
*"When the safety component breaks, let everything through or block everything?"*
- **Fail open:** stay available, lose protection briefly. Typical for rate limiters.
- **Fail closed:** stay safe, become unavailable. Typical for security checks like authentication.
- **Seen in:** rate limiter (Step 7.3).

### Backpressure
*"The consumer can't keep up → signal the producer to slow down."*
- Ways to do it: bounded queues (reject when full), `429` responses, producers pausing when queue depth is high.
- **Seen in:** notification system (10M-message campaign).

### Load shedding
*"Overloaded → deliberately drop low-priority work so the important work survives."*
- e.g. drop marketing traffic and keep logins and payments.

### Graceful degradation
*"Part of the system is down → show less, but don't show an error page."*
- e.g. recommendations fail → show the page without them; click counter down → redirects still work.

### Noisy neighbour / bulkhead pattern
*"One heavy workload slows everyone sharing the resource."* → *"Give each part its own resources."*
- Named after a ship's watertight compartments: one floods, the ship still floats.
- **Seen in:** notification system (separate queues and workers per channel and priority, so a 10M campaign can't delay login codes).

---

## Async and messaging (details in [message-queues.md](message-queues.md))

| Term | One-liner |
|---|---|
| **Asynchronous processing** | Hand the work off, reply immediately (`202 Accepted`), do the work later |
| **Producer / consumer (worker)** | Puts messages in / takes them out and does the work |
| **Message broker** | The queue system itself (RabbitMQ, Kafka, SQS) |
| **ACK** | Worker → broker: "done, delete it". No ACK → the message is delivered again |
| **Publisher confirm** | Broker → producer: "stored safely" |
| **Dead letter queue (DLQ)** | Where messages go after all retries fail; inspected and replayed by engineers |
| **Poison message** | A message that always fails (bad data); without a retry limit it loops forever, which is why DLQs exist |
| **Fan-out** | One event delivered to many queues or services |
| **Transactional outbox** | Write the data + an outbox row in one DB transaction; a relay publishes from the outbox |
| **Decoupling** | Producer and consumer don't need to know about or wait for each other |
| **Pub/Sub (publish–subscribe)** | Publishers send to a *channel*; every current subscriber receives it. Nothing is stored, so if no one is listening it's gone (unlike a queue). e.g. each chat server subscribes to its own channel `server-8` |
| **Connection registry / session store** | Key-value map `user → server they're connected to` (usually Redis, with a TTL refreshed by heartbeats). Answers "where is Bob?"; the message itself never passes through it. Seen in: chat app |
| **Heartbeat** | Periodic "still alive" ping (e.g. every 30 s). If heartbeats stop, the TTL expires and the user or server is treated as gone. Seen in: chat app (presence, registry) |

---

## API and HTTP

| Term | Meaning |
|---|---|
| `201 Created` | Something was created and is finished |
| `202 Accepted` | Accepted; it will be done later (async) |
| `301` vs `302` | Permanent redirect (browsers cache it) vs temporary redirect (every click comes back to you) |
| `404` vs `410 Gone` | Never existed / not found vs existed and has been removed permanently |
| `409 Conflict` | Clashes with existing data (e.g. alias already taken) |
| `429 Too Many Requests` | Rate limited; check `Retry-After` |
| **Headers vs body** | Headers = information *about* the response (limits, caching, redirect target); body = the content itself |
| **API gateway** | The single front door: authentication, rate limiting, routing, logging |
| **Load balancer** | Spreads traffic across servers |

---

## Estimation

| Term | Meaning |
|---|---|
| **Back-of-the-envelope** | Rough calculations to decide what the design needs (1 machine or 100?) |
| **QPS / RPS** | Queries / requests per second |
| **Peak vs average** | Traffic bursts (e.g. 30× the average) → size the system, or a queue, for bursts |
| **Read-heavy / write-heavy** | Which one dominates decides caching, replicas, DB choice |
| Handy numbers | 1 day ≈ 100k s (really 86,400) · 1 month ≈ 2.5M s · 62⁶ ≈ 56B · 62⁷ ≈ 3.5T |

---

## Security basics

| Term | Meaning | Fix |
|---|---|---|
| **Never trust the client** | Anything the client sends (a `paid: true` tag, a client-side limit) can be faked | Decide on the server |
| **Enumeration** | Guessing IDs by counting (`000001`, `000002`…) | Random-looking IDs (random codes, or a bijection before base62) |
| **Brute force** | Trying many passwords or codes quickly | Rate limit by IP/account, lockouts |
| **Least data** | A service stores only what it needs (no passwords in the notification DB) | Keep data in the service that owns it |
