# Message Queues (MQ)

## What it is
A **message queue** holds tasks ("messages") so that one part of the system can hand off work and move on, while another part does the work later at its own pace.

- **Producer** = puts messages in (your app server / Notification Service).
- **Message broker** = the queue system itself (**RabbitMQ**, **Kafka**, **AWS SQS**, Google Pub/Sub). "The MQ" usually means the broker.
- **Queue** = one named line of messages **inside** the broker (e.g. `email_queue`). It only **stores** messages; it doesn't run anything.
- **Worker / consumer** = **your own program** that takes messages off a queue and does the actual task.
- **Provider** = the third party a worker may call (SendGrid, Twilio, FCM). It isn't part of the queue system.

> Post office analogy: **RabbitMQ** = the post office, **queue** = an incoming mail tray, **worker** = an employee, **message** = a task, **ACK** = the employee telling the post office "done".

```
App server ──publish──→ [ RabbitMQ: email_queue ] ──deliver──→ Email workers ──→ SendGrid
           ←─confirm───        (stores messages)   ←───ACK─────  (your code)
```

A worker is just code, and you can run many copies of it on the same queue:

```javascript
channel.consume("email_queue", async (msg) => {
  try {
    const email = JSON.parse(msg.content.toString());
    await sendEmail(email);   // call the provider
    channel.ack(msg);         // tell RabbitMQ: done, delete it
  } catch (error) {
    // failed → retry (see below)
  }
});
```

## Why use one (what it solves)
| Problem | How the queue helps |
|---|---|
| Caller waits for slow work | The producer publishes and returns immediately (**async**) |
| Traffic spikes | The queue **buffers** the burst; workers drain it at a safe speed |
| A provider is down | Messages wait in the queue and get retried; nothing is lost |
| One slow service blocks others | Separate queues + workers → **isolation** |
| Need more throughput | Add more workers (**horizontal scaling**) |

**Async ≠ no database calls.** Async only means the **producer doesn't wait** for the consumer to finish. Workers can still read and write the DB.

## "Is the MQ smart?" (who decides what)
**No.** The broker transports and stores messages, and routes them **only by rules you configure**. It doesn't understand your business logic.

- **Your app** decides *what* to send, *to whom* and *on which channel* (it checks preferences, picks the template).
- **The broker** decides only *which queue* a message lands in, based on its routing setup.

Two ways to route:

1. **The app picks the queue:** it publishes the email job straight to `email_queue`, the SMS job to `sms_queue`.
2. **Broker routing rules (RabbitMQ):** the app publishes to an **exchange** with a **routing key**. **Bindings** (rules you set up) send it to the matching queues.
   ```javascript
   channel.publish("notifications", "email.welcome", Buffer.from(JSON.stringify(message)));
   // binding: "email.*" → email_queue
   ```
   - Kafka: **topics** and partitions. SQS: separate queues, with **SNS** for fan-out (one event → many queues). Pub/Sub: topics + subscriptions.

A message is just data, usually JSON, in whatever format your workers expect:

```json
{ "event_id": "evt_123", "user_id": "u_456", "channel": "EMAIL",
  "type": "WELCOME", "template": "welcome_email", "retry_count": 0 }
```

## One broker, many queues (not one broker per service)
- **Broker / cluster** = the whole messaging system. **Queue** = one line inside it.
- **Recommended:** **one RabbitMQ cluster** with **separate queues per channel** (email, SMS, push), each with its own workers, retry queue and DLQ.

```
                  RabbitMQ cluster
                         |
                 Notification exchange
       +-----------------+-----------------+
   Email queue       SMS queue        Push queue
   Email workers     SMS workers      Push workers
   Email retry       SMS retry        Push retry
   Email DLQ         SMS DLQ          Push DLQ
```

Why not **one shared queue** for everything? An email worker could pick up an SMS job, 10,000 emails would block 100 SMS, and you couldn't scale or monitor channels separately.

Why not **one broker per service**? It's extra infrastructure to run for no gain. Give a service its own cluster only when it needs stronger isolation, e.g. email grows so large that it slows the others.

> Warehouse analogy: one warehouse (broker), many departments (queues). Build a second warehouse only when one department outgrows the first.

## Batches: one message per recipient
Sending to 10,000 users:

| One message with 10,000 user IDs ❌ | 10,000 messages, one per user ✅ |
|---|---|
| Worker crashes at user 5,000 → which ones were sent? | Each job succeeds or fails alone |
| One huge retry; one worker busy for a long time | Retry only the failed ones |
| No per-user tracking | Status per user, scale with more workers, control the speed |

- One job = **one recipient + one channel**.
- For huge campaigns, **generate jobs in chunks** (e.g. 1,000 users at a time) so you don't load 10M users into memory at once.

## Preferences / opt-outs
The broker knows nothing about opt-outs; **your code** checks them:

1. **When creating jobs:** look up preferences and create jobs only for allowed channels (User B opted out of email → only an SMS job is created).
2. **Again in the worker, just before sending:** the user may have unsubscribed **after** the job was queued (queued at 10:01, unsubscribed at 10:02, sent at 10:03 ✗).

Preferences depend on **channel × category**:

| Category | Examples | Opt-out? |
|---|---|---|
| **Transactional** | OTP, password reset, order confirmation | Usually always sent |
| **Promotional** | Newsletter, discount offer | User can opt out (and the law often requires an unsubscribe option) |

```json
{ "user_id": "123",
  "preferences": { "EMAIL": { "TRANSACTIONAL": true, "PROMOTIONAL": false },
                   "SMS":   { "TRANSACTIONAL": true, "PROMOTIONAL": true } } }
```

## Acknowledgements (ACKs)
There are **two different ACKs**, and **neither goes to your app server**:

| ACK | From → to | Meaning |
|---|---|---|
| **Publisher confirm** | Broker → producer | "I've safely stored your message" |
| **Consumer ACK** | Worker → broker | "I finished this message; you can delete it" |

- The **worker** sends the ACK; the **broker** then deletes the message.
- **No ACK** (the worker crashed or rejected it) → the broker delivers the message again, so nothing is lost.
- Your server **doesn't get 10,000 ACKs**. It publishes and moves on.
- If you need to know progress (e.g. a campaign dashboard: 8,500 sent / 1,200 pending / 300 failed), **workers write the status to the DB**, and your server reads the DB. That's **business tracking**, separate from broker ACKs.

> One line: **your server creates work, the broker manages delivery, workers do the work, and the database records the results.**

## Retries
**Immediate requeue** (put it straight back) → if the provider is down, the same message fails over and over in a tight loop. ❌

**Delayed retry** ✅: on failure, the message goes to a **retry queue**, waits, and then returns to the main queue.

```
Email queue → worker → success → ACK (deleted)
                    → failure → retry queue (wait) → back to email queue
                    → failed too many times → DLQ
```

| Attempt | Wait before retry |
|---|---|
| 1 | 30 s |
| 2 | 2 min |
| 3 | 10 min |
| 4 | 30 min |
| Max reached | → **DLQ** |

- This is **exponential backoff**: each wait is much longer than the last. Add **jitter** (a small random amount) so thousands of messages don't all retry at the same second.
- **Counting attempts:** keep the same `event_id` and add a `retry_count` field to the message. (RabbitMQ also adds an `x-death` header that records how many times a message has been dead-lettered.) Don't confuse the event ID with the attempt number.
- RabbitMQ can do delays with **TTL + dead-letter routing** (a message that expires in the retry queue is sent back to the main queue) or with a **delayed-message plugin**. You don't need a separate queue per attempt.
- Only retry **temporary** errors (timeouts, 5xx, 429). "Invalid email address" goes straight to failed.

## Dead letter queue (DLQ)
- Where messages go **after all retries fail** (or on other configured conditions, e.g. rejected or expired).
- The DLQ **doesn't retry** and **doesn't send anything back** to your server. It **holds** failed messages.
- Engineers **inspect** them, fix the cause, and **replay** them in a controlled way, not automatically.
- **Monitor more than the DLQ**, so you catch problems before messages pile up there:
  - **queue depth** (how many messages are waiting)
  - **oldest message age** (is the queue stuck?)
  - retry volume, worker errors, provider error rates, DLQ depth

## Duplicates and idempotency
**Queues are usually at-least-once:** a message can be delivered **more than once** (e.g. the worker sent the email, then crashed **before** ACKing → the broker delivers it again).

**Idempotency** = processing the same message twice has the same effect as processing it once. How:

1. **Idempotency key** per job, e.g. `campaign_id + user_id + channel` or the `event_id`.
2. **Before sending:** check whether that key is already marked `SENT`. If yes → just ACK, don't send again.
3. **After sending:** mark it `SENT`.
4. Enforce it with a **unique constraint** in the DB:
   ```sql
   CREATE UNIQUE INDEX unique_notification
   ON notification_deliveries (campaign_id, user_id, channel);
   ```

Where to store the "already sent" records:

| Option | Pros | Cons |
|---|---|---|
| **Database** (+ unique constraint) | Permanent; you need the DB anyway | One indexed lookup per job (usually fine) |
| **Redis** (`notification:evt_123 = SENT`, with a TTL) | Very fast | Keys expire, and data can be lost |

**The gap neither one closes:** provider sends → worker crashes **before** writing `SENT` → retry → **sent twice**.
- If the provider supports **idempotency keys**, pass your `event_id` to it, and the provider rejects the repeat.
- Otherwise there is an unavoidable small uncertainty window. Handle it with **reconciliation**: ask the provider for the delivery status.
- **Exactly-once delivery is generally impossible** across systems that can crash. Aim for **at-least-once + idempotency**.

**Broker deduplication ≠ business idempotency.** Some brokers drop duplicate *publishes*:
- **SQS FIFO:** deduplication ID, within a 5-minute window
- **Azure Service Bus:** duplicate detection by message ID
- **Kafka:** idempotent producers and transactions

None of them can know whether the **email provider** already sent the email, so you still need your own idempotency.

## Worker DB checks: don't they cancel out the benefit of async?
No. The queue still gives decoupling, buffering, spike handling, independent scaling and retries. To keep the checks cheap:

- Check preferences **when creating jobs**, so jobs that shouldn't be sent never get created.
- **Combine** the final preference check and the idempotency check into **one indexed query**.
- **Batch** lookups (fetch preferences for many users in one query).
- **Cache** preferences in Redis when DB load justifies it (with a short TTL so opt-outs take effect quickly).

## Transactional outbox pattern
**Problem:** saving to the DB and publishing to the queue are **two separate operations**. If the app crashes between them, you get a DB row with no message, or a message with no DB row.

**Fix:** in the **same DB transaction**, write the business data **and** an "outbox" row containing the message. A separate **relay** process reads the outbox table and publishes to the broker, marking rows as published. Both writes succeed or fail together, and the relay retries publishing until it works. This is at-least-once again, so consumers must still be idempotent.

## Who does what
| Responsibility | Component |
|---|---|
| Business logic, which channels, preferences | App server / workers (your code) |
| Delivery, routing, buffering, ACKs, retries | Message broker (RabbitMQ / Kafka / SQS) |
| Permanently failed messages | DLQ |
| Preferences, campaigns, delivery status (permanent records) | Database (PostgreSQL / MySQL) |
| Fast temporary state: cache, shared rate limits, coordination | Redis (add when needed) |
| Preventing duplicate sends | DB unique constraint + idempotency key |

- **Queue** = delivery and transport. **Database** = permanent business state. **Worker** = processing and business decisions. **Redis** = fast temporary state.
- **Scale note:** SQL is fine for delivery history in a small or medium system. At our exercise's volume (50M notifications/day × 30 days ≈ 1.5B rows), use **Cassandra** or partitioned tables for the log.

## RabbitMQ vs Kafka (in one glance)
| | RabbitMQ | Kafka |
|---|---|---|
| Model | Queue: a message is **deleted after ACK** | Log: messages are **kept for a retention period**; each consumer tracks its position (**offset**) |
| Routing | Exchanges + routing keys + bindings | Topics + partitions |
| Retries / DLQ | Built-in tools (TTL, dead-letter exchanges) | Mostly built yourself (retry topics) |
| Good for | Task queues like notifications | Very high-volume event streams; many consumers re-reading the same events |

Start with **RabbitMQ** for learning task queues; its routing concepts are the easiest to see.

## Things to learn next
- RabbitMQ exchanges and routing keys
- Kafka topics and **consumer groups**
- **Fan-out:** one event triggers many independent services
- **Backpressure:** what to do when producers are faster than workers (e.g. 10M campaign messages)
