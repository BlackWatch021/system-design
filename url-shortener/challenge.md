# Exercise 1: Design a URL Shortener (like bit.ly)

## The problem

Users paste a long URL and get a short one back:

```
https://www.example.com/articles/2026/11/system-design-for-beginners?ref=newsletter
        ↓
https://sho.rt/aZ3x9Q
```

When anyone opens `https://sho.rt/aZ3x9Q`, they are redirected to the original long URL.

---

## Requirements

### Functional (what it must do)

1. Given a long URL → return a unique short URL.
2. Opening the short URL → **redirect** to the original URL.
3. Short links **expire** after a default time (e.g. 1 year); users can optionally set their own expiry.
4. _(Optional / stretch)_ Users can choose a **custom alias**, e.g. `sho.rt/my-blog`.
5. _(Optional / stretch)_ Show **click count** per link.

### Non-functional (how well it must do it)

- **Highly available:** if redirects go down, every link on the internet that uses us breaks.
- **Low latency** for redirects (users are waiting on a click).
- Short codes should **not be easily guessable** (no simple 1, 2, 3...).
- **Read-heavy:** links are clicked far more often than they are created.

### Out of scope

- User accounts / login, analytics dashboards, spam/malicious URL detection.

---

## Numbers to work with (assumptions)

- **100 million** new short URLs created per month
- **Read:write ratio = 100:1** (100 redirects for every 1 URL created)
- Links stored for **5 years**
- Average long URL size ≈ **500 bytes**

---

## Your task: work through these steps in order

### Step 1: Clarify requirements

Write down 2–3 questions you'd ask the interviewer, and the assumptions you'll make.

Question 1
How long can be the short url, cuase the smaller the url is, smaller will be the amount of url we can generate.
Question 2
Is it necessary to keep the sho.rt in the url and after that we have that unique code, can I do something like sho.rt/aksdhg/wlkejs
Here first unique section is for the site url and second unique section is for the details followed by the site (https://linkedin/user/profile/section/certification)
So aksdhg = linkedin
wlkejs=user/profile/section

### Step 2: Back-of-the-envelope estimation

Calculate:

- Writes per second (new URLs / sec)
- Reads per second (redirects / sec)
- Total number of URLs over 5 years
- Total storage needed over 5 years

> Tip: 1 month ≈ 2.5 million seconds. Round numbers aggressively; being roughly right is the goal.

Rough Estimation
Writes = 100 mil per months
Then per second == 100000000/30*24*60\*60 = 38.5 ~ 40 writes per second

Reads = Writes and reads are in ratio Read:Write::100:1 + 40*100= 4000 reads per second
Total number of URLs = 100 mil url per month *12*5=6000000000
Tolat storage needed = average size of a url = size of actual url(500 bytes) + short url(100 bytes) = 600 bytes * 100 mil*12 (months) *5(years)= around 3.6 TB

### Step 3: API design

Define the endpoints. For each one: method, path, request body, response.

- Creating a short URL
- Redirecting
- Think: which **HTTP status code** should a redirect return? (Look up `301` vs `302`; the choice matters!)

EndPoints

1. Post Request
   status - 301 (as its cached compare to 302)
   path - /api/v1/createUrl
   Method - POST
   Body - {data:{url:'user pasted url'}}
   Response - {data:{message:'success'/'failed',shortUrl:"newly created url"}}

2. GET Request
   status - 201
   path - /api/v1/getUrl/{short url slug}
   Method - GET
   Body - {}
   Respone - {data:{message:'success'/'failed', url:'Actual url'}}

### Step 4: Data model

- What fields does each record need?
- SQL or NoSQL? Why?

SQL as the data is mostly structured and we aren't chanding the data in any way plus it will be faster for reads too as the application is read heavy
Single Table

\_id , url, shortUrl, timeStemp, expiry

timeStemp to keep record when it was created
expiry, we will generate it while generating url as the cron jobs won't need to calcualte expiry for each entry through timestemp, it will reduce the load on server/db.

### Step 5: Generating the short code (the core problem)

- How long should the code be? (Hint: with characters `a-z A-Z 0-9` = 62 options per character, how many codes do 6 or 7 characters give you? Is it enough for Step 2's total?)
- How do you generate it? Think of at least **two** approaches and their trade-offs.
- What happens if two long URLs get the **same** short code (a collision)?

6 characters will give as approx 56 billion options, which is way more then our need
-GENERATION
approach 1 - generate the short url in incremental way, like first will be 000000 then 000001 and so on, but the issue here will be easy of finding out urls, as hackers will get the upper ceiling of the url then they can hit each and every url in sequential way. Which compromises our value of anonymity and the url should not be easy to guess.

approach 2
Generate random code each time, this way it will be way harder to guess the next url plus we will be preserving the privacy of user. Then the issue is to ensure that the url isn't in use, for this we will have to check with the db each time, which will increase atleast one call per request, this is minimum. What if every generated code is taken, then for a single short url , we will hit the whole db which isn't ideal.

In case of same short code, the last written url will get preference but it will never happen as we will be ensuring that no two or more urls gets the same short url.

### Step 6: High-level design (draw it!)

Draw the components and how a request flows through them, for:

- **Create flow:** user → ... → saved
- **Redirect flow:** user clicks → ... → redirected

Consider: load balancer, app servers, database, cache, CDN. **Which of these do you really need, and where?**

Done with the flow, it in diagram.drawio file

### Step 7: Deep dive / scaling

Answer at least two of these:

- How do you make redirects fast? (Where does caching help? Can a **CDN** help here? Think about what we learned!)
- What happens if a single short link goes **viral** (millions of clicks in a minute)?
- How do you scale the database when it grows too large?
- How do you clean up **expired** links?

---

1. For quick response, caching is done at two places, first is at CDN level then in redis.
2. In that case it will be stored inside CDN and if its a miss then it will be in the redis cache, so no matter how much the traffic is, the architecture will handle it.
3. We can do sharding, although there will be a master db for handeling the write responses, so in most of the cases we will be handeling one single master DB for the data, which is for writing mostly then there are separate DBs for read requests.
4. Every night or the time when the load will be less on the server, that time we will be running the cron job to check the expiry of the links and remove them once its over

## Rules

- Write your answer in a new file in this folder: `my-solution.md`
- Add your diagram (export as an image or put the link) in this folder too.
- Don't look up full solutions online. It's fine to look up individual concepts (e.g. "what is base62").
- Stuck on a step? Write what you think, mark it with ❓, and move on. I'll help with those parts during review.

---

# 📌 Review notes: missed points & answers (reference)

## Step 1: Clarify requirements

Good extra questions to ask:

- If two users shorten the **same** long URL, should they get the **same** short link or different ones?
- Can a link be **edited or deleted** after it's created?
- Do we need **custom aliases** and **click counts** in version 1 (they're marked optional)?

Assumptions to write down:

- The short code uses only `a-z A-Z 0-9` (base62).
- Each create request gets a new short code (no de-duplication).
- A link can't be changed after it's created.
- The default expiry is 1 year, and the maximum is 5 years (our storage window).

## Step 3: API design (full answer)

### 1. Create a short URL

```
POST /api/v1/urls
Content-Type: application/json

{
  "longUrl":     "https://www.example.com/articles/2026/11/...",
  "customAlias": "my-blog",               // optional
  "expiresAt":   "2027-11-11T00:00:00Z"   // optional, default = now + 1 year
}
```

Success: **`201 Created`**

```
{
  "code":      "aZ3x9Q",
  "shortUrl":  "https://sho.rt/aZ3x9Q",
  "longUrl":   "https://www.example.com/articles/2026/11/...",
  "expiresAt": "2027-11-11T00:00:00Z"
}
```

Errors, using status codes (not `message: 'failed'`):

| Status                  | When                                                                            |
| ----------------------- | ------------------------------------------------------------------------------- |
| `400 Bad Request`       | The long URL is invalid, or the expiry is in the past or more than 5 years away |
| `409 Conflict`          | The custom alias is already taken                                               |
| `429 Too Many Requests` | One client is creating too many links (rate limiting)                           |

### 2. Redirect

```
GET /{code}            e.g.  GET https://sho.rt/aZ3x9Q
```

- The path is the **short URL itself**. It's not under `/api/v1/`, because the URL has to stay short and must never change.
- Success: **`302 Found`**, with the header `Location: https://www.example.com/articles/...` and an empty body. The **browser** reads `Location` and follows it automatically. That's the "redirect".
- `40isted but has **expired**. 410 means "it was here, and it's gone for good".
4 Not Found`: the code doesn't exist.
- `410 Gone`: the code ex

### 3. (Stretch) Click count

```
GET /api/v1/urls/{code}/stats   →   200 OK   { "code": "aZ3x9Q", "clicks": 1523 }
```

### 301 vs 302: the decision

|                               | 301 Moved Permanently                                          | 302 Found (temporary)      |
| ----------------------------- | -------------------------------------------------------------- | -------------------------- |
| Browser caches it?            | Yes, possibly forever                                          | No (unless told to)        |
| Next click from the same user | Goes **straight to the long URL**, so our server never sees it | Comes to **us** every time |
| Click counting (req. 5)       | ❌ Repeat clicks are lost                                      | ✅ Every click is counted  |
| Expiry (req. 3)               | ❌ The browser keeps redirecting even after the link expires   | ✅ We can return 410       |
| Load on our servers           | Lower                                                          | Higher                     |

**Choose 302**, because it keeps the expiry and click-count requirements working. Handle the extra load with the CDN and Redis (Step 7).

Bonus: the `Cache-Control: s-maxage=60` header lets a **shared cache (the CDN)** keep the redirect for 60 seconds, while browsers don't cache it. That way you get most of the speed of 301, and expiry still takes effect within about a minute.

## Step 4: Data model

| Field        | Type              | Notes                                                                                     |
| ------------ | ----------------- | ----------------------------------------------------------------------------------------- |
| `code`       | string(7)         | **Primary key** (unique index). Store only the code; the domain is the same for every row |
| `longUrl`    | string(~2 KB max) |                                                                                           |
| `createdAt`  | timestamp         |                                                                                           |
| `expiresAt`  | timestamp         | **Indexed**, so the cleanup job can find expired rows quickly                             |
| `clickCount` | integer           | Optional (stretch requirement)                                                            |

**SQL vs NoSQL:** decide based on the **access pattern**, not on "SQL is faster".

- Our access pattern is one lookup by a single key, with no joins and no relationships, across 6 billion rows and about 3.6 TB.
- A **key-value / NoSQL store** (DynamoDB, Cassandra) matches that exactly and spreads across machines (sharding) by itself.
- **SQL** (Postgres or MySQL) also works, but at this size you'd have to shard it yourself. Either answer is fine if you give this reasoning.

## Step 5: Short code generation

**Length:**

- 62^6 ≈ 56 billion codes and 62^7 ≈ 3.5 trillion codes.
- 6 characters is enough for 6 billion links, but **7 is safer**. With random codes, a full table means 10% of new codes collide at 6 characters, but only about 0.17% at 7.

**Approach 1: counter + base62**

- ✅ Never collides, so no "does this exist?" check is needed.
- ❌ Sequential codes are guessable.
- Fix: put the number through a **reversible shuffle** (a bijection, meaning every number maps to exactly one different number) before base62-encoding it. Codes then look random but still never collide.
- ❌ All servers need the same counter, which makes it a bottleneck.
- Fix: each server reserves a **block** of numbers (e.g. 1,000 at a time) from a central counter (Redis `INCRBY`, or a DB row), then hands them out from memory.

**Approach 2: random code**

- ✅ Hard to guess, and simple.
- Collisions are possible but rare, and checking one code is a **single indexed lookup**, not a scan of the DB.

**Approach 3: hash the long URL** (e.g. MD5 → base62 → take the first 7 characters)

- ✅ The same long URL always gets the same code (free de-duplication).
- ❌ It still collides sometimes, and two users can't get different expiries for the same URL.

**Collisions: the correct handling**

- Don't "check, then insert": two servers can both check, both see "free", and both insert.
- Do this instead: **insert directly** and let the DB's **unique constraint** reject a duplicate. On a duplicate-key error, generate a new code and retry (give up after about 3–5 tries).
- **Never overwrite** an existing link: someone's link would silently start pointing somewhere else.
- For a custom alias that's already taken, return `409 Conflict` (don't retry).

## Step 6: Diagram fixes

1. **The CDN goes in front of the load balancer**, not beside the client:
   - Redirect flow: `Client → CDN → (miss) → Load Balancer → App Server → Redis → (miss) → Replica DB`
   - Create flow: `Client → Load Balancer → App Server → Master DB` (writes skip the CDN)
2. **Cache-aside:** the **app server** asks Redis. On a miss, the app server reads the DB, then stores the result in Redis. Redis doesn't call the DB itself.
3. **Replication lag bug:** when a link is created, also **write it into Redis** (called "write-through"). Then an immediate click finds it in Redis, even if the replica hasn't caught up yet.
4. **No single point of failure:** run at least **2 load balancers** (one active, one standby) and several app servers. "Highly available" means no single box can take us down.

## Step 7: Deep dive

### 1. Fast redirects

The layers, from closest to the user to furthest:

1. **CDN edge** (a short `s-maxage`, e.g. 60 s)
2. **Redis** (in memory, under 1 ms)
3. **Replica DB** (indexed lookup on `code`)

With about 4,000 reads/sec and most clicks going to popular links, the CDN and Redis absorb the large majority of reads.

### 2. Viral link

- The CDN serves almost all of the traffic at the edge.
- **Cache stampede:** before the link is cached, thousands of requests miss at the same moment and all hit the DB. Fixes:
  - **Request coalescing:** only one request fetches from the DB while the rest wait for that answer. CDNs call this "collapsed forwarding".
  - **A lock per key** in Redis.
- **Hot key:** a single Redis node gets hammered. Fixes:
  - Add Redis read replicas.
  - Keep a tiny in-memory cache on each app server for a few seconds.
- **Click counting:** don't run `UPDATE ... clickCount + 1` on one DB row millions of times. Instead:
  - Increment a counter in Redis (`INCR`) and save it to the DB every minute or so.
  - Or send click events to a queue (e.g. Kafka) and count them in batches.

### 3. Scaling the DB: replication ≠ sharding

|                    | Replication                              | Sharding                                              |
| ------------------ | ---------------------------------------- | ----------------------------------------------------- |
| What it does       | **Copies all** the data to more machines | **Splits** the data across machines                   |
| Solves             | Too many **reads**; a machine dying      | Data **too big** for one machine; too many **writes** |
| Each machine holds | The full 3.6 TB                          | Only its part (e.g. 3.6 TB ÷ 8 shards ≈ 450 GB)       |

- **Shard key = `code`**, using `shard = hash(code) % N`. Every redirect already knows the code, so it goes to **exactly one** shard.
- Use **consistent hashing**, so adding a shard moves only a small part of the data.
- Each shard still has its own **master + replicas**. Real systems use both techniques together.

### 4. Cleaning up expired links

- **Check at read time (lazy):** if `expiresAt < now`, return `410 Gone`. A link then stops working the moment it expires, not when the next nightly job runs.
- **A nightly cron job removes the rows**, in **batches** (e.g. 1,000 rows at a time using the `expiresAt` index), so the DB isn't locked for a long time.
- **Caches:** set the Redis **TTL** (time to live) to the time remaining until `expiresAt`, and keep the CDN `s-maxage` short.
- **Don't reuse expired codes** (or only after a long time): people may still have the old link in emails or posts, and it would suddenly lead to someone else's site.
