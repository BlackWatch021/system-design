# CDN (Content Delivery Network)

## What it is
A network of cache servers (**PoPs: Points of Presence**) spread across many cities, close to users ("**the edge**").
It stores copies of content so users get it from nearby instead of from your far-away data center.

- **Edge** = outer boundary of the internet, physically closest to users.
- **Origin** = where the original content lives (S3 / object storage for media, app server / LB for APIs).
- The **load balancer** is the front door of *your data center*, not the closest point to the user.

## Where it sits in the architecture
The CDN sits **beside** your system at the edge, not in a chain with Redis. It only speaks HTTP and never talks to Redis or the DB.

```
                ┌──── Static / public content ────────────────────────────┐
   ┌──────┐     ▼                                                         │
   │ User │ ──► CDN Edge (HIT → return) ── MISS ──► Origin (S3 / LB+App) ─┘
   └──────┘
       │  Personalized requests (login, cart, balance)
       ▼
   Load Balancer ──► App Servers ──► Redis ──► Database
```

Two common setups:
- **Separate domains:** `cdn.myapp.com` → CDN, `api.myapp.com` → LB directly.
- **Everything behind the CDN:** personalized requests **pass through** the CDN uncached (you still get DDoS protection, fast TLS and a hidden origin IP).

## How a user reaches the CDN
1. The HTML points to `https://cdn.myapp.com/cat.jpg`.
2. DNS / **Anycast** routes the user to the **nearest PoP**.
3. **HIT** → served instantly. **MISS** → the edge fetches from the origin, stores a copy and returns it.
4. On a miss, the response comes back along the same path; the edge **streams** it to the user while storing it.

## What can be cached
| ✅ Cache on CDN | ❌ Never cache on CDN |
|---|---|
| Images, video, JS, CSS, fonts | Bank balance, cart, "my orders" |
| Public API responses (catalog, blog, trending) | Anything behind a login / per user |
| Same response for everyone | Must always be exact and current |

- The CDN caches the **whole HTTP response**, whatever your server did to build it (Redis, DB, etc.).
- **Cache key** = usually the full URL + query string. Same URL means the same cached copy, so private data must not be cached.
- Controlled by **headers**:
  - `Cache-Control: public, s-maxage=300` → the CDN may cache it for 300 s
  - `Cache-Control: private, no-store` → the CDN passes it through without caching

## Pull vs. push
| Pull (most common) | Push |
|---|---|
| The CDN fetches from the origin on the first request | You upload to the CDN yourself |
| The first user per region gets a slow miss | You manage uploads and updates |

## Freshness: the cache invalidation problem
A cache holds a **copy**. When the original changes, **the CDN doesn't know**; it serves the old copy until the TTL expires.

Fixes (best first):
1. **Versioned filenames:** `logo.v2.png` / `app.a8f3c.js` → new URL = new cache key. You can cache for a year.
2. **Purge / invalidate** through the CDN's dashboard or API.
3. **Shorter TTL:** fresher content, but more origin traffic (a trade-off).

`stale-while-revalidate` → serve the old copy instantly while refreshing in the background.

> "There are only two hard things in CS: cache invalidation and naming things."

## DNS vs. CDN (don't mix them up)
| | Caches |
|---|---|
| DNS | Name → IP address |
| CDN | Actual files / responses |

## Handling misses at scale
- **Request coalescing:** many users miss on the same key at once → the edge sends **one** request to the origin, and the others wait and receive the same response.
  - Built in and on by default for most managed CDNs (Nginx needs `proxy_cache_lock on`).
  - Implemented with an in-memory **hash map of in-flight fetches** (key → waiting requests), a fast lookup by key.
  - Works per edge/PoP, and only for cacheable content.
- **Origin shield (tiered cache):** edges → regional shield → origin, so the origin sees ~1 request per file instead of one per edge.
- Same problem with Redis = **cache stampede / thundering herd**, same fix (one request rebuilds, the others wait; e.g. Go `singleflight`).

## Spikes
- A CDN doesn't "auto-scale" like AWS. It has **massive pre-built shared capacity** across hundreds of PoPs.
- Cache hits are cheap (no DB, no app logic), and traffic is spread across PoPs.
- ⚠️ It only protects **cacheable** traffic. Personalized requests still hit your origin, so you still need auto-scaling, LBs and Redis.

## Benefits
- **Lower latency** (distance: Delhi→Virginia ≈ 200+ ms vs. a local edge ≈ 5–20 ms)
- **Origin offload** → fewer servers
- **DDoS protection**, and it **hides the origin IP** (lock the origin firewall to CDN IPs only)
- Absorbs **traffic spikes**
- Cheaper bandwidth (egress)

## Cost trade-off
- You pay the CDN per GB and per request, but origin egress and compute go down.
- **The cache hit ratio decides it:** 95% hits = big savings; 10% hits = the CDN may cost more than it saves.
- Interview line: *"A CDN adds cost, but for media-heavy apps it usually reduces total cost by offloading bandwidth and compute. The savings depend on the cache hit ratio."*

## When you might skip it
A tiny app with few users in one region, mostly personalized content → extra cost and complexity for little gain.

## CDN vs. Redis
| | CDN | Redis |
|---|---|---|
| Location | Edge, near users | Your data center, near app servers |
| Caches | Files / HTTP responses | Data (query results, sessions) |
| Talks to | The user's browser | Your app server |
| Saves | Network distance (latency) | Database load |
