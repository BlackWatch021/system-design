# Exercise 3: Design a Notification System

## The problem

Many parts of a company need to tell users something:

```
Order service:     "Your order #4521 has shipped"            → push notification
Auth service:      "Your login code is 482913"               → SMS
Marketing team:    "50% off this weekend!" to 10M users      → email + push
```

Instead of every team building its own sending code, we build **one Notification Service** that all other services call. It sends notifications through three **channels**:

- **Push** (phone notifications), via Apple / Google push services
- **SMS**, via an SMS provider (e.g. Twilio)
- **Email**, via an email provider (e.g. SendGrid)

We **don't** build the actual delivery to phones or inboxes; we call these **third-party providers**. They are outside our control: they can be slow, go down, or limit how fast we call them.

---

## Requirements

### Functional (what it must do)

1. Any internal service can ask us to send a notification to a user through **push, SMS, and/or email**.
2. Notifications use **templates** with variables, e.g. `"Hi {name}, your order {orderId} has shipped"`.
3. Users can **opt out** per channel and per type (e.g. "no marketing emails", but still receive security SMS).
4. _(Stretch)_ **Scheduled** notifications, e.g. "send at 9:00 am in the user's time zone".
5. _(Stretch)_ Track the **status** of each notification: queued → sent → delivered / failed.

### Non-functional (how well it must do it)

- **Reliable:** notifications must not get lost. (A lost login code means the user can't log in.)
- **No duplicates:** a user shouldn't get the same SMS twice. It's annoying, and every SMS **costs money**.
- **Priority:** a login code must not wait behind 10 million marketing emails.
- **Handles bursts:** marketing campaigns send millions of notifications at once.
- **Doesn't slow down the callers:** the order service shouldn't have to wait for an email to actually be delivered.

### Out of scope

- Building our own SMS / email / push delivery (we use providers).
- An in-app notification inbox.
- Analytics dashboards (open rates, click rates).
  /

---

## Numbers to work with (assumptions)

- **50 million** notifications per day in total:
  - 40 million push
  - 8 million email
  - 2 million SMS
- A marketing campaign can send **10 million** push notifications within **10 minutes**.
- Our email provider accepts at most **1,000 requests / second** from us.
- One SMS costs about **$0.01**.
- We keep a log of every notification for **30 days**. Each log record ≈ **500 bytes**.

---

## Your task: work through these steps in order

### Step 1: Clarify requirements

Write down 2–3 questions you'd ask the interviewer, and the assumptions you'll make.

questions

1. Do notifications has to be synchronous to server
2. Do all notifications should reach each user at the same time or a little bit of up and down will work
3. Do we have an upper limit to send the notificaitons

assumptions

1. I am assuming this is some sort of ordering platform
2.

### Step 2: Back-of-the-envelope estimation

Calculate:

- **Average** notifications per second.
- **Peak** notifications per second during a campaign. How does it compare to the average?
- How long would the 8 million daily emails take if we could only send 1,000 per second?
- **Storage** for 30 days of notification logs.
- **SMS cost** per day. What would it cost per day if a bug sent every SMS **twice**?

> Tip: 1 day ≈ 100,000 seconds (really 86,400). Round aggressively.

Notifications per seconds = 578.7 ~ 579 notifications per second - Push Notification 462.9~463 push notifications per second - Email -~93 eamils per second - SMS - 23 sms per second
Peak notifications - 16,666.66~ 16,667 notifications per seconds
Way above our avg notifications per second

If we are sending emails continuous, then it will take around 2.22 hours to send 8 million emails. other wise the average is 93 emails per seconds

storage= 50 million * 30 days *500 bytes for each notification storage = 50000000*30*500= 7,50,00,00,00,000 bytes = around 750 gb or 0.75 tb

SMS cost = $20,000 per day.
in case of a bug, it will double our expense on SMS ~~ $40,000

### Step 3: API design

How does another service (e.g. the order service) ask us to send a notification?

- Method, path, request body, response.
- Should we respond **after** the notification is actually delivered, or **immediately**? (Think: what if the SMS provider takes 3 seconds to answer, or is down?)
- Which **status code** fits the answer you chose? (Look up `202`.)

### Step 4: Data model

- What do we need to store? Think about: user contact details (phone, email, device tokens), user **preferences** (opt-outs), **templates**, and the **notification log**.
- SQL or NoSQL for each? Why? (They don't all have to be the same.)

Refer data model diagram

### Step 5: Sending: the core problem

Imagine the simplest design: the order service calls the Notification Service, which calls the SMS provider, waits for the answer, then replies.

```
Order service → Notification Service → SMS provider
              (waits...)           (waits...)
```

- What goes wrong when the provider is **slow**? When it's **down**? When a campaign sends **10 million** at once?
- Look up **message queue** (concept lookup is allowed). What is it, and how would it change this design?
- Should push, SMS and email share **one** queue or have **separate** queues? Why?
- Who reads from the queue and actually calls the providers? (Look up **worker** / **consumer**.)

-The whole architecture can't be in synchronous pattern, as one service calling other and waiting for its response, it will make our app bugging and less resposive.
-That's why we need to use message queue, as it takes the load from the server and completed the task/event in asynchronous manner. This way server only does the heavy lifting like taking the order, and passes down the notification part to MQ.

- So, inside the MQ we can create separate queues as per our need, and here we can clearly see that we are heavly relying on push notificadtions, 400+ notifications a second, so its a best thing to choose separate queues for each service as this will also keep things separate and will improve the efficiency.
- So, inside a MQ we have a queue, and there we have workers(provider), who takes the events and sends the request to the provider(consumer)

### Step 6: High-level design (draw it!)

Draw the components and the flow for:

- **Happy path:** order service → ... → the user's phone buzzes.
- **Failure path:** the SMS provider is down. What happens to the notification?

Consider: the Notification Service, queues, workers, databases (preferences, templates, log), cache, third-party providers. Label arrows with **what actually happens** (e.g. "check opt-out", "put on SMS queue"), not just "REQ/RES".

### Step 7: Deep dive

Answer at least two of these:

- **Retries:** the email provider is down for 10 minutes. How do you retry without hammering it the moment it comes back? (Look up **exponential backoff**.) What if a notification keeps failing forever? (Look up **dead letter queue**.)
- **Duplicates:** a worker sends an SMS, the provider sends it, but the worker crashes before marking it "sent". Another worker picks it up and sends it again. How do you prevent the double SMS? (Look up **idempotency**.)
- **Priority:** a 10M marketing campaign is in progress and someone requests a login code. How does the login code still arrive within seconds?
- **Provider limits:** the email provider only accepts 1,000 requests per second. How do you make sure you never send faster than that, even with many workers? (Think back to Exercise 2!)

---

-Retries- You already mentioned the answer, we can use exponential backoff, where we will delay each retry with certain amount, like for 1st time it will be 4 seconds, then for 2nd it will be 10 seconds etc. Just an example. and for the case of failing forever, we hahve DLQ, which sends the details back to the server.

- To mitigate duplicates, we can use a cache or something similar, where we will store the event id plus user id, then will be checking if it exist, if yes, mean we already send the event.
- Priotity- Really nice question, for this we will be using priority queue concept, in this each event gets a number depicting its priority, using this we can move the urgent notifications/email first.
- For this we can ensure that we are doing operation in batches where a particular batch will never exceed the limit.

## Rules

- Write your answers under each step (like before) or in a new `my-solution.md` in this folder.
- Add your diagram to this folder (`diagram.drawio` is fine).
- Don't look up full solutions online. It's fine to look up individual concepts (e.g. "what is a message queue").
- Stuck on a step? Write what you think, mark it with ❓, and move on. I'll help with those parts during review.

---

# 📌 Review notes: improved solution (reference)

## How a message queue works (read this first)

Think of the ticket rail in a restaurant kitchen:

- The **waiter** clips an order ticket to the rail and goes straight back to customers, without waiting for the food → the **producer**.
- The **rail** holds tickets in order; if 50 orders arrive at once, they simply wait → the **queue**.
- **Cooks** each take a ticket, cook the dish, and only then throw the ticket away → the **workers / consumers**.

```
Order service ──POST──→ Notification Service ──puts message──→ [ SMS queue ] ──→ SMS workers ──→ Twilio
                ←202────  (replies immediately)                 (stored on disk)   (our code)     (provider)
```

1. The **Notification Service** (producer) puts a message such as `{notificationId, userId, phone, text}` on the queue and replies immediately.
2. The **queue** is a separate system (**Kafka**, **RabbitMQ**, **AWS SQS**). It **stores messages on disk**, so a crash doesn't lose them.
3. A **worker** (our own code, on its own servers) takes a message, calls the provider, updates the log, then sends an **ack** (acknowledgement: "done, delete it").
4. If a worker **crashes before acking**, the queue gives the message to another worker after a timeout. Nothing is lost, but this is also where duplicates come from (Step 7.2).

What a queue gives us:

| Benefit | Why |
|---|---|
| Fast callers | They get an answer immediately, not after delivery |
| No lost notifications | Messages wait safely on disk |
| Absorbs bursts | 16,667/s go in; workers take them out at their own speed |
| Survives provider outages | Messages wait in the queue until the provider is back |

Terms:

- **Producer:** puts messages on the queue (the Notification Service).
- **Consumer / worker:** takes messages off and does the work (our code).
- **Provider:** the third party that actually delivers (Twilio, SendGrid, Apple/Google push).

## Step 1: Clarify requirements

Questions:

1. Should the caller wait for delivery, or just get "accepted"? (→ sync vs async)
2. Is a rare **duplicate** acceptable, or must every notification be sent exactly once?
3. Can users opt out of **security** messages (login codes)? (Usually no.)
4. Do we need multiple **languages** for templates?

Assumptions:

- This is a **shared** service that every team calls (orders, auth, marketing, …), not a feature of one app.
- Callers are **internal services** inside our network, not end users.
- Delivery is **at-least-once**: never lose a notification; a rare duplicate is acceptable.
- Security notifications ignore opt-outs; marketing respects them.

## Step 2: Estimation

```
Average:   50M / 86,400 s ≈ 580/s   (push ≈ 460, email ≈ 90, SMS ≈ 25)
Peak:      10M / 600 s   ≈ 16,700/s → about 30× the average
Emails:    8M / 1,000 per s = 8,000 s ≈ 2.2 hours
Storage:   50M × 30 days × 500 B = 750 GB ≈ 0.75 TB
SMS cost:  2M × $0.01 = $20,000/day; duplicated = $40,000/day (+$600k a month)
```

What the numbers tell us:

- A peak **30× the average** means we shouldn't size servers for the peak. A **queue** absorbs the burst, and workers drain it.
- Even with no limits on our side, a campaign's emails take **hours** because of the provider's limit, so campaigns must not block urgent messages (Step 7.3).
- Duplicates cost real money, so idempotency matters (Step 7.2).

## Step 3: API design

The API is between the **caller** and the **Notification Service**. The queue is internal; callers never see it.

### Send a notification

```
POST /api/v1/notifications
{
  "userId": "u_123",
  "type": "order_shipped",                    // decides priority and opt-out rules
  "channels": ["push", "email"],
  "templateId": "order_shipped_v2",
  "data": { "name": "Riya", "orderId": "4521" },
  "idempotencyKey": "order-4521-shipped"      // the same key sent twice → only one notification
}
```

Response: **`202 Accepted`**

```
{ "notificationId": "n_789", "status": "queued" }
```

- **202** = "accepted, will be done later". It's honest, because delivery hasn't happened yet. (200 = done; 201 = created and finished.)
- Before replying, the service:
  1. Checks the **idempotency key** (seen before? → return the earlier `notificationId`, don't send again).
  2. Loads the user's contact details and **preferences**, and drops opted-out channels.
  3. Fills in the **template**.
  4. Writes a log row per channel with status **"queued"**.
  5. Puts one message on each channel's (and priority's) queue.

Errors:

| Status | When |
|---|---|
| `400` | Invalid input (unknown channel, missing template data) |
| `404` | Unknown user or template |
| `429` | The caller is sending too many requests (Exercise 2) |

### (Stretch) Check status

```
GET /api/v1/notifications/{notificationId}  →  200 OK
{ "notificationId": "n_789", "channels": { "push": "delivered", "email": "sent" } }
```

## Step 4: Data model

Store **only what the notification service needs**:

- No orders or addresses; the caller passes the needed text in `data`.
- **Never passwords.** If this database leaked, passwords would leak with it.

| Table | Fields | Store |
|---|---|---|
| `users_contact` | `userId` (PK), `email`, `phone`, `timezone`, `language` | SQL |
| `devices` | `deviceId` (PK), `userId`, `pushToken`, `platform` (ios/android), `updatedAt` | SQL |
| `preferences` | `userId`, `channel`, `type`, `allowed` (PK = `userId + channel + type`) | SQL + cache in Redis |
| `templates` | `templateId`, `channel`, `language`, `subject`, `body` with `{placeholders}`, `version` | SQL + cached in memory |
| `notification_log` | `notificationId` (PK), `userId`, `channel`, `type`, `templateId`, `status`, `retryCount`, `error`, `providerMessageId`, `idempotencyKey`, `createdAt`, `updatedAt` | **Cassandra** |
| `campaigns` (optional) | `campaignId`, `templateId`, `audience`, `scheduledAt`, `status` | SQL |

Notes:

- **Device tokens** are a separate table, because one user can have several phones or tablets. Push is impossible without them.
- **Preferences as rows**, not one column per combination: a new type or channel needs no schema change.
- **`notificationId` per channel** as the log's key. `campaignId + userId` breaks for messages without a campaign (orders, login codes), and for one campaign sent over both email and push.
- **Status lifecycle:** `queued → sent → delivered`, or `→ failed` (after retries, → DLQ).

**SQL vs NoSQL:**

- **Users, devices, preferences, templates → SQL.** They're small, structured and read-heavy, and they change rarely. Cache them (Redis, or in memory) so each notification doesn't hit the DB.
- **Logs → Cassandra (NoSQL).** About 50M writes per day, always looked up by `notificationId` or `userId`, with no joins, and old rows can **expire automatically** (TTL of 30 days).

## Step 5: Sending: sync vs async

**The synchronous design breaks in three ways:**

| Situation | What goes wrong |
|---|---|
| Provider is **slow** (3 s) | The order service waits 3 s on every call; its own users wait too |
| Provider is **down** | The notification is **lost** (or every caller has to build its own retry logic) |
| **10M at once** | ~16,700/s overwhelms our servers, and the email provider's 1,000/s limit rejects most of them |

**With a queue:** the Notification Service puts messages on queues and replies **202** immediately. **Workers** read from the queues at a safe speed and call the providers. When a provider is down, messages wait (and retry); when a burst arrives, the queue holds it.

**Separate queues, mainly for isolation:**

- If Twilio is down, only the SMS queue fills up; email and push keep flowing.
- Each channel gets its own number of workers and its own speed limit (email ≤ 1,000/s).
- Split further **by priority**: `sms-high`, `sms-low`, `email-high`, `email-low`, … (Step 7.3).

**Who calls the providers:** **workers** (consumers), our own code running on separate servers, for example a group of SMS workers, a group of email workers and a group of push workers. Add more workers when a queue grows.

## Step 6: High-level design

```
Order / Auth / Marketing services
        │  POST /notifications
        ▼
  Notification Service ──check──→ Idempotency (Redis)
   (several instances) ──read───→ Preferences + Templates (SQL, cached)
        │              ──write──→ Notification log: "queued" (Cassandra)
        │
        ├─ put → [ sms-high ]  [ sms-low ]   → SMS workers   ──→ Twilio
        ├─ put → [ email-high ][ email-low ] → Email workers ──→ SendGrid
        └─ put → [ push-high ] [ push-low ]  → Push workers  ──→ Apple / Google push
                                                  │   │
                     shared token bucket (Redis) ←┘   └→ update log: sent / failed
                                                  │
                         failed 5 times ──────────┴──→ Dead letter queue → engineers inspect / replay
```

**Happy path:**

1. The order service calls `POST /notifications` and immediately gets **202 Accepted**.
2. The Notification Service checks the idempotency key and preferences, fills in the template, logs "queued", and puts the message on `push-high`.
3. A push worker takes the message, takes a token from the rate limiter, and calls Apple/Google.
4. The provider accepts it; the worker logs "sent" and **acks** the message.
5. The user's phone buzzes.

**Failure path (SMS provider down):**

1. An SMS worker calls Twilio and gets an error or a timeout.
2. The worker does **not** ack; it schedules a retry with backoff (1 s, 2 s, 4 s, …).
3. Meanwhile, SMS messages wait in the queue. Email and push are unaffected.
4. After about 5 failures, the message moves to the **dead letter queue** and its log status becomes "failed".

## Step 7: Deep dive

### 1. Retries

- **Exponential backoff:** the wait **doubles** after each failure: 1 s → 2 s → 4 s → 8 s → 16 s.
- **Jitter:** add a small random amount to each wait (e.g. 4 s ± 1 s). Without it, thousands of failed messages all retry at the same second and hit the provider again the moment it recovers.
- **Max retries** (e.g. 5) → move the message to the **dead letter queue (DLQ)**. The DLQ doesn't send anything anywhere: it **holds** failed messages so engineers or tools can inspect them, fix the cause, and **replay** them.
- Not every error should be retried: "invalid phone number" will never succeed → straight to failed. Only retry temporary errors (timeouts, 5xx, 429).

### 2. Duplicates (idempotency)

There are two different sources:

| Source | Fix |
|---|---|
| The **caller** sends the same request twice (it timed out and retried) | Idempotency key in the API: store `idempotencyKey → notificationId` in Redis (with a TTL of e.g. 24 h); if the key was seen before, return the earlier result and send nothing |
| A **worker crashes** after the provider sent the message but before recording "sent" | Write status **"sending"** before calling the provider. If another worker later finds "sending", it can ask the provider's API whether the message went out, or pass an idempotency key to the provider (if it supports one) so the provider rejects the repeat |

- **Exactly-once delivery is generally impossible** across systems that can crash. Real systems choose **at-least-once** (never lose a message, occasionally repeat one) and make the duplicate window as small as possible.
- For SMS, each duplicate costs money, so it's worth the extra "sending" step there. For push, a rare duplicate is usually acceptable.

### 3. Priority

- **Separate queues per priority**, each with **its own workers**:
  - `high`: login codes, password resets, security alerts
  - `low`: marketing, newsletters
- High-priority workers never touch the low queue, so a login code never waits behind 10M campaign messages.
- Why not one priority queue: many queue systems (Kafka, SQS) don't support priorities. Even where they do, a single queue still shares workers with the campaign.
- **Reserve provider capacity:** e.g. give `email-high` 200 of the 1,000/s, and let the campaign use the other 800.

### 4. Provider limits (1,000/s for email)

- **Each worker limiting itself doesn't work:** 10 workers × 1,000/s each = 10,000/s.
- **Use a shared token bucket in Redis** (Exercise 2): the bucket refills at 1,000 tokens/s, and **every** email worker takes a token (an atomic Lua script) before calling SendGrid. If there are no tokens, the worker waits a moment.
- If SendGrid still answers `429`, respect its `Retry-After` header and back off.
- The rest of the 8M emails simply **wait in the queue**. That's exactly what the queue is for.

### Key takeaways

1. **Slow or unreliable work goes behind a queue:** producer → queue → workers → ack, and the caller gets **202 Accepted**.
2. **Separate queues** give isolation (one provider down doesn't stop the others) and priority (login codes skip marketing).
3. **Retries:** exponential backoff + jitter, then a dead letter queue.
4. **At-least-once + idempotency keys**, because exactly-once is generally impossible.
5. **A service stores only its own data:** no orders or passwords in the notification database.
6. **A limit shared by many workers needs a shared counter** (a Redis token bucket).
