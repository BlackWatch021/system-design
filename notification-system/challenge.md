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

### Step 2: Back-of-the-envelope estimation

Calculate:

- **Average** notifications per second.
- **Peak** notifications per second during a campaign. How does it compare to the average?
- How long would the 8 million daily emails take if we could only send 1,000 per second?
- **Storage** for 30 days of notification logs.
- **SMS cost** per day. What would it cost per day if a bug sent every SMS **twice**?

> Tip: 1 day ≈ 100,000 seconds (really 86,400). Round aggressively.

### Step 3: API design

How does another service (e.g. the order service) ask us to send a notification?

- Method, path, request body, response.
- Should we respond **after** the notification is actually delivered, or **immediately**? (Think: what if the SMS provider takes 3 seconds to answer, or is down?)
- Which **status code** fits the answer you chose? (Look up `202`.)

### Step 4: Data model

- What do we need to store? Think about: user contact details (phone, email, device tokens), user **preferences** (opt-outs), **templates**, and the **notification log**.
- SQL or NoSQL for each? Why? (They don't all have to be the same.)

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

## Rules

- Write your answers under each step (like before) or in a new `my-solution.md` in this folder.
- Add your diagram to this folder (`diagram.drawio` is fine).
- Don't look up full solutions online. It's fine to look up individual concepts (e.g. "what is a message queue").
- Stuck on a step? Write what you think, mark it with ❓, and move on. I'll help with those parts during review.
