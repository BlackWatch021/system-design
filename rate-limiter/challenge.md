# Exercise 2: Design a Rate Limiter

## The problem

A **rate limiter** controls how many requests a client may send in a period of time. When a client goes over the limit, extra requests are **rejected** before they reach our servers.

```
User sends 101st request this minute
        ↓
Rate limiter: "limit is 100/min" → ❌ rejected (the API servers never see it)
```

You already met this in Exercise 1: `429 Too Many Requests` on the create-URL endpoint. Now you'll design the thing that produces that 429.

Why do we need one?

- Stop one user (or a bot) from overloading the system for everyone else.
- Block brute-force attacks (e.g. someone trying 10,000 passwords on `/login`).
- Control cost (e.g. every request calls a paid third-party API).

---

## Requirements

### Functional (what it must do)

1. Limit requests per **client** according to a **rule**, e.g.:
   - Free users: **100 requests / minute**
   - Paid users: **1,000 requests / minute**
   - `/login`: **5 attempts / minute per IP address**
2. Requests within the limit are passed on to the API servers; requests over it are **rejected**.
3. A rejected client should be **told** that they were limited, and ideally **when they can try again**.
4. Rules can be **changed** without redeploying the code.

### Non-functional (how well it must do it)

- **Low latency:** every request passes through the limiter, so it must add only a few milliseconds.
- **Accurate:** limits must hold even though we have **many API servers** behind a load balancer.
- **Highly available:** if the rate limiter breaks, it must not take the whole API down with it.
- **Memory efficient:** we have millions of users.

### Out of scope

- Detecting _which_ traffic is malicious (bot detection, DDoS scrubbing).
- Billing users for usage.

---

## Numbers to work with (assumptions)

- **10 million** daily active users
- **20,000 requests / second** at peak, across all users
- **50 API servers** behind the load balancer
- Each rule applies **per user** (or per IP for `/login`)

---

## Your task: work through these steps in order

### Step 1: Clarify requirements

Write down 2–3 questions you'd ask the interviewer, and the assumptions you'll make.

Questions

1. Is the application global or local
2. If user hits the quota per minute, then what will happen, will they get a warning and time limit only. Or the punishment will chance with the time its done.
   Like first time, a cool down
   Second time Increased cool down
   Third time bigger cool down
   Fourth time, complete ban on account etc

Assumptions

1. One session per user, no two or more login at a time
2. Considering rest of the system is working perfectly, will mostly focus on the rate limiting part only.

### Step 2: Back-of-the-envelope estimation

Calculate:

- How much **memory** is needed to keep a counter for every active user? (Decide what you store per user, estimate its size in bytes, multiply.)
- How many **operations per second** will the counter storage have to handle?

Memory
1.Considering we will be storing user name(Unique for each user) followed by a counter and time of first request to maintain the session of one minute. That will be around 1kb, that is more than required but still considering in future we may wanna add something. That's why
1kb \* 10000000 = 10 gb at peak time. So we can keep it as it is.
Cause not every user will be active at the same time plus not everyone is going to exploit the system.

2. Counter storage function needs to handle 20,000 operations per second to update the counter against every request. But its a simple ++ operation against each user.

> Tip: Round aggressively. The point is to find out whether this fits on **one machine** or needs many.

### Step 3: Where does the rate limiter live?

Possible places:

- In the **client** (browser / mobile app)
- **Inside each API server** (in its own memory)
- As a **separate layer** in front of the API servers (middleware / API gateway)

For each one: would it work, and why or why not? Pick one.

So the rate limiter will live at the CDN layer as its is the closest to the user(edge computing) plus it will act as gate keeper, will not allow the request to pass to the next stage if it isn't correct or its has reached the limit.

Then at the db layer, we can store the data regarding the particular user if they have reached the limit, so if user thinks that they can change the location/ip address to fool the system, then it will be catched easily, will save this list on redis for easy and fast look up.

### Step 4: The algorithm (the core problem)

Look up **at least two** of these (looking up concepts is allowed) and explain each one **in your own words, with a small example**:

- Fixed window counter
- Sliding window log
- Sliding window counter
- Token bucket
- Leaky bucket

For each one: How does it work? What does it store per user? Pros and cons?

Things to think about:

- With a fixed window of "100 per minute", what happens if a user sends 100 requests at **12:00:59** and another 100 at **12:01:00**?
- Should a user be allowed a short **burst** (e.g. 20 requests in one second) if they stay under the per-minute limit?

Then **pick one** for our system and say why.

I looked up for these and couldn't understand them properly but I have my idea, tell me if it fits any of them or not.
So the time user's first request hits the CDN, it will be stored with user's time, counter to be 1 and user name, then the request will go on, now second time we will check the time, compare it with the stored time and check if the counter is below 100/1000 or not. If yes, then allow the request and increase the counter other wise straign away deny the request, send another request to the server to add the user's name under block list in db and redis, so in case the user tries to change his location in order to fool the cdn, it can still be caught by the data and redis. And update it similarly in the CDN too but with expiry to be the cool down of the user, so we can handle it at the edge only. Saving precious bandwidth and data.
For distinguishing between paid and unpaid user, we can attach the tag on request body that the user is paid or not from the client side only, once they login in into their account.
And since 20 requests in a second sits still inside the minute rule, there fore we will have to let it pass, other wise we can also check for this in the function too. Minor tweek in the function and we can mitigate this issue too.

### Step 5: The response and the rules

- What **status code** does a limited client get? What **headers** would help the client know its limit, how many requests it has left, and when to retry?
- Where are the **rules** (100/min, 1,000/min, 5/min…) stored, and how does the limiter find out when they change?

Status code - 429 Too many requests for both users, paid and unpaid.
If you ask me, its best to not let the user know what the limit is or how many requests they are left with, just to keep things confidential. Yet if you want it, then we can pass down all these data through the body of the response
{data:{
limit:'100/1000 request per minute'
requestLeft:'45',
timeToRetry:'2145 hours, 14 november 2026'

}}

The timeToRetry is just the example, we will use the actual time formate for it, then it will be used to calculate the difference between current and this time to show a counter to the user.

### Step 6: High-level design (draw it!)

Draw the components and the request flow for:

- **Allowed request:** client → ... → API server → response
- **Rejected request:** client → ... → 429

Consider: load balancer, rate limiter, where the counters live, where the rules live, API servers.
Done, check the diagram.drawio file for the design

### Step 7: Deep dive

Answer at least two of these:

- We have **50 API servers**. If each one counted in its own memory, what would go wrong? Where should the counters live instead?
- Two requests from the same user arrive at **exactly the same time** on different servers. Both read "99", both allow it, both write "100". The user got 101 requests through. How do you prevent this?
- The counter store (e.g. Redis) **goes down**. Do you let all requests through, or block all of them? What are the trade-offs?
- The service runs in **several regions** (US, EU, Asia). How do you limit a user globally?

---

1. That will not be efficient, as we have add the counting feature in each server and even after that, load will be distributed. means they aren't used to their full potential. Just a wastage of resource. Instead we need to implement the feature in some centralised place, like CDN.
2. We are monitoring each request inside the CDN, so it will still count them one after another.
3. This is a nice question, didn't thought about it. But we can use a cluster of CDNs to ensure in case of a failed edge, we can switch to another one without a fail.
4. First assumption I made was a single session per user. So a single user can't access the app at two places simultaneously to metigate this problem only. Other wise we have to maintain a separate DB just to log each users request which isn't ideal plus not a good idea too.

## Rules

- Write your answers under each step (like last time) or in a new `my-solution.md` in this folder.
- Add your diagram to this folder (`diagram.drawio` is fine).
- Don't look up full solutions online. It's fine to look up individual concepts (e.g. "what is token bucket").
- Stuck on a step? Write what you think, mark it with ❓, and move on. I'll help with those parts during review.

---

# 📌 Review notes: improved solution (reference)

## Step 1: Clarify requirements

Questions:

1. Is the app **global or single-region**? (This decides Step 7.4.)
2. **What identifies a client:** user ID, IP address or API key? (The whole design depends on this.)
3. Should the limits be **hard** (always reject) or **soft** (allow a small burst over the limit)?
4. If the rate limiter itself fails, should the API **stay open** or **block everything**?

Assumptions:

- Logged-in requests are limited per **user ID**; `/login` is limited per **IP** (there's no user yet).
- A user can have **many sessions** at once (phone + laptop). Counting per user ID handles that, so no session rule is needed.
- Going over the limit is **not a punishment**: the user just waits until they have quota again. Escalating bans are abuse handling (out of scope).
- Being slightly inaccurate (a few extra requests) is acceptable; the goal is protecting the system, not exact billing.

## Step 2: Estimation

**Memory:** store only what the algorithm needs.

| Field | Size |
|---|---|
| Key (e.g. `rl:user:123`) | ~20 bytes |
| Token count / counter | 4–8 bytes |
| Last refill time / window start | 8 bytes |
| Redis overhead per key | ~50 bytes |
| **Total** | **~100 bytes per user** |

```
10 million users × 100 bytes = 1 GB   → fits in the RAM of ONE Redis machine
```

- Keys **expire** (TTL) after the user goes quiet, so inactive users don't use memory.
- Don't pad the estimate to 1 KB "for the future". Estimation is about realistic sizes; future growth is handled by scaling (Redis Cluster), not by guessing high.

**Operations:**

```
20,000 requests/sec × 1 Redis operation each (atomic INCR or one Lua script) ≈ 20,000–40,000 ops/sec
One Redis node handles ~100,000 ops/sec   → one primary per region is enough
```

## Step 3: Where does the rate limiter live?

| Place | Works? | Why |
|---|---|---|
| **Client** (browser / app) | ❌ | The user controls the client and can remove or change it. Never trust the client |
| **Inside each API server** | ❌ | The load balancer spreads one user's requests over 50 servers; each sees only ~1/50 of them (see 7.1) |
| **CDN edge** | 🟡 First filter only | Hundreds of edges, each with its own memory, and they don't know users or plans. Good for blocking obvious floods per IP, not for accurate per-user limits |
| **API gateway** (separate layer) | ✅ **Chosen** | Every request passes through it, it knows the user (it checks the login token), and all gateway instances share one Redis |

An **API gateway** is the single front door to the backend. It handles authentication, rate limiting, routing (`/login` → auth service, etc.) and logging. It runs in **our region, next to our servers**, as several identical, stateless instances.

## Step 4: The algorithm

**Your idea = fixed window counter.** Store a counter and a window start time; allow requests while the counter is under the limit; reset after 60 s.

Its flaw is the **boundary spike**:

```
12:00:00   1 request     → window starts, count = 1
12:00:59   99 requests   → count = 100 ✅
12:01:00   100 requests  → new window, count reset → 100 more ✅
           = 199 requests in ~1 second on a "100/min" limit
```

The algorithms:

| Algorithm | How it works | Stores per user | Pros / cons |
|---|---|---|---|
| Fixed window counter | Count requests in each fixed minute | 1 counter + start time | ✅ Simplest. ❌ Boundary spike (up to 2× the limit) |
| Sliding window log | Save every request's timestamp; count those in the last 60 s | Up to 100 timestamps | ✅ Exact. ❌ Much more memory |
| Sliding window counter | `previous minute's count × overlap% + current count` (e.g. at 12:01:15: `80 × 0.75 + 30 = 90`) | 2 counters | ✅ Fixes the spike, tiny memory. ❌ An approximation |
| **Token bucket** ⭐ | A bucket holds up to N tokens and refills at a steady rate. Each request takes 1 token; empty bucket → 429 | Tokens + last refill time | ✅ Tiny memory, allows **controlled bursts**. Very widely used |
| Leaky bucket | Requests join a queue that is processed at a fixed rate | A queue | ✅ Perfectly smooth output. ❌ Bursts wait or get dropped |

**Choice: token bucket.**

- Free plan: bucket size 100, refill 100/min (~1.7 tokens per second).
- Paid plan: bucket size 1,000, refill 1,000/min.
- Bursts: the bucket size *is* the burst setting. A user who has been quiet can send 20 requests in one second, because they're spending saved tokens.

**Paid vs free:** the gateway looks up the user's plan **on the server side** (from the user ID in their login token, cached for a few minutes). Never trust a `paid: true` tag sent by the client: anyone can edit it.

## Step 5: The response and the rules

**Rejected request:**

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1731500000
```

- Send the `X-RateLimit-*` headers on **every** response (also successful ones), so well-behaved clients slow down **before** hitting the limit.
- Use **headers**, not the body: every client can read them, whatever the response body format is.
- `Retry-After` in **seconds** avoids problems with clocks set to different times.
- Hiding the limit doesn't keep it secret (anyone can find it by sending requests until they get a 429); it only hurts honest developers. Requirement 3 also asks us to say when to retry.

**Rules:**

- Rules live in a **config store** (a DB table or config service), e.g. `{ plan: "free", limit: 100, per: "minute" }`, `{ endpoint: "/login", key: "ip", limit: 5, per: "minute" }`.
- Each gateway keeps a copy **in its own memory** (no extra network call per request) and **refreshes it every 30 s – 5 min**, or gets notified when a rule changes.
- Rules change without redeploying any code (requirement 4).

## Step 6: High-level design

```
                     ┌─ Gateway 1 ─┐
Client → CDN → LB →  ├─ Gateway 2 ─┼──→ Redis primary (counters) ──→ Replica (AZ-b)
                     └─ Gateway 3 ─┘         │                      └→ Replica (AZ-c)
                           │                 │
                           │          (INCR / Lua script → new count)
                           │
              ┌────────────┴─────────────┐
       within limit                 over limit
              ↓                          ↓
   Load balancer → API servers    429 back to the client
                                  (API servers never see it)

Gateways ← refresh every 30 s – 5 min ← Rules config store
```

- **Allowed:** Client → CDN → gateway → Redis says OK → load balancer → API servers → response (with `X-RateLimit-*` headers).
- **Rejected:** Client → CDN → gateway → Redis says over the limit → **429** straight back.
- **No block list in the DB.** Counters that expire do the "blocking" automatically. If you limit by user ID, changing IP or location changes nothing.

## Step 7: Deep dive

### 1. Why not count inside each of the 50 API servers?

- The load balancer spreads one user's requests across all 50 servers, so each server sees about 2 of them.
- The user could send **50 × 100 = 5,000 requests per minute** before any single server notices.
- The problem is **wrong answers**, not wasted resources.
- Fix: one **shared counter store (Redis)** that every gateway checks. The gateways themselves stay stateless.

### 2. Race condition (both read 99, both allow)

Requests are handled **in parallel**, by many gateways at once, so this really happens. The fix is an **atomic** operation (one that can't be interrupted halfway):

- **Fixed window:** Redis `INCR rl:user:123` adds 1 *and* returns the new value in a single step. If it returns 101, reject. (Set an `EXPIRE` of 60 s on the key so it resets.)
- **Token bucket:** it needs several steps (read tokens → refill → take one → save), so run them as one **Lua script** inside Redis. Redis runs a script all at once, with nothing in between.

Every check goes to the **Redis primary**. Replicas are copied asynchronously and lag slightly behind, so reading counts from them could wrongly allow requests.

### 3. Redis goes down

| Choice | Result |
|---|---|
| **Fail open** (let everything through) | ✅ The API stays up. ❌ No protection for a short while |
| **Fail closed** (block everything) | ✅ Safe. ❌ The whole API is down, which breaks the "must not take the API down" requirement |

- **Choose fail open** for normal API traffic.
- Fallback while Redis is down: each gateway enforces a rough limit in its **own memory** (e.g. limit ÷ number of gateways).
- **Make failure rare:** Redis primary + replicas in **different availability zones** (separate buildings in the same region). If the primary dies, a replica is promoted within seconds (**failover**). A few recent counts may be lost, which is fine.

### 4. Several regions (US, EU, Asia)

- A **region** = a cluster of data centers in one metro area that serves a large part of the world (e.g. Mumbai serves India; Ireland serves much of Europe). Inside it are **availability zones** a few km apart, kept in real-time sync.
- Each region has its **own gateways + Redis primary**, close to its users. One shared Redis for the whole world would add ~100+ ms to every request.
- **The loophole:** a user who deliberately routes requests through several regions (e.g. VPNs) could get up to `limit × number of regions`. Options:

| Option | Trade-off |
|---|---|
| Accept it | Simple; worst case 3× with 3 regions |
| Split the limit (100 ÷ 3 each) | Never over, but normal users who only use one region get just 33 ❌ |
| Home region per user | Exact, but requests may cross an ocean (+~100 ms) ❌ |
| **Periodic sync** ⭐ | Each region counts locally and sends **batched** updates to the others every ~1 s ("user 123: +5"). Small overshoot (~1 s worth), cheap (~600 KB/s) |

- **Choose periodic sync.** The counts are **eventually consistent** (a second or so behind, then caught up), which is fine for a rate limiter.
- For strict limits (e.g. failed logins on an account), keep that counter in **one place** (where the account lives) and accept the cross-region delay. Logins are rare, so it doesn't matter.

### Key takeaways

1. **Never trust the client**, for limits or for plan information.
2. **Shared state needs a shared store, and updates must be atomic** (Redis `INCR` / Lua).
3. **A CDN is many separate copies, not one central place.**
4. **Token bucket** is the algorithm most often expected.
5. Rate limiting is allowed to be **slightly inaccurate**; that trade buys speed and availability.
