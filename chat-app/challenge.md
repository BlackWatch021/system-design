# Exercise 4: Design a Chat App (like WhatsApp / Messenger)

## The problem

Users send text messages to each other in real time, one to one and in groups:

```
Alice types "Hey, are you free tonight?" → taps Send
        ↓  (well under a second)
Bob's phone shows the message instantly
Alice sees: ✓ sent → ✓✓ delivered → ✓✓ (blue) read
```

Everything you've built so far was **request → response**: the client asks, the server answers. Here the server must **push** a message to Bob **without Bob asking for it**. That changes a lot.

---

## Requirements

### Functional (what it must do)

1. **1:1 chat:** send and receive text messages.
2. **Group chat:** up to **100 members** per group.
3. **Delivery status:** sent ✓, delivered ✓✓, read (blue ✓✓).
4. **Offline users** get their messages when they come back online, and a **push notification** while they're away.
5. **Chat history:** users can scroll back through old messages.
6. **Presence:** show "online" / "last seen at 10:42".
7. _(Stretch)_ **Typing indicator** ("Bob is typing…").
8. _(Stretch)_ Send **images**.

### Non-functional (how well it must do it)

- **Real-time:** when both users are online, delivery feels instant (< ~200 ms).
- **No lost messages:** once Alice sees ✓, the message must eventually reach Bob, even if servers crash.
- **Ordering:** messages in a chat appear in the **same order** for everyone.
- **Highly available:** people rely on this to reach each other.
- **Scales** to tens of millions of users connected at the same time.

### Out of scope

- End-to-end encryption, voice/video calls, stories/status.
- Multiple devices per user (assume **one phone per user**).
- Spam and abuse detection.

---

## Numbers to work with (assumptions)

- **50 million** daily active users
- Each user sends **40 messages per day** on average
- **Peak** traffic is **3×** the average
- Average message ≈ **100 bytes** of text (+ metadata ≈ 200 bytes stored per message in total)
- Messages are stored for **5 years**
- At peak, **20 million** users are online (connected) **at the same time**
- One chat server can hold about **50,000** open connections at once

---

## Your task: work through these steps in order

### Step 1: Clarify requirements

Write down 2–3 questions you'd ask the interviewer, and the assumptions you'll make.

### Step 2: Back-of-the-envelope estimation

Calculate:

- Messages per day, average messages per second, and **peak** messages per second.
- Storage per day and over **5 years**.
- How many **chat servers** are needed just to hold the open connections at peak?

> Tip: 1 day ≈ 100,000 seconds. Round aggressively.

### Step 3: Real-time delivery (the core problem)

Bob's phone needs to receive Alice's message **as soon as it's sent**. Look up these approaches (concept lookup is allowed) and explain each **in your own words**:

- **Short polling** (Bob's app asks "anything new?" every few seconds)
- **Long polling**
- **WebSockets**
- _(Bonus)_ **Server-Sent Events (SSE)**

For each one: How does it work? How fast is it? What does it cost the server (think about 20 million users)? Then **pick one** and say why.

### Step 4: Protocol / API

- What kinds of messages travel over the connection between the app and the server? (Think: sending a chat message, the server confirming it, "delivered", "read", "typing"…) Write down the **fields** of a chat message.
- Which things are better as normal **REST** endpoints instead (e.g. loading old history, creating a group, logging in)?
- Alice's phone loses signal right after tapping Send, and it **resends** the message when it reconnects. How do you avoid Bob getting it **twice**? (Check your [glossary](../glossary.md)!)

### Step 5: Data model

- What tables or collections do you need? (Think: users, chats/conversations, group members, messages, delivery/read status.)
- The most common query is **"give me the latest 50 messages of chat X"** (and then the 50 before those, as the user scrolls up). How do you make that fast?
- **Ordering:** can you sort messages by the timestamp from the sender's **phone clock**? What could go wrong? What would you use instead?
- SQL or NoSQL? Why?

### Step 6: Routing between chat servers

Alice is connected to **chat server 7**. Bob is connected to **chat server 42**. Alice's message arrives at server 7.

- How does server 7 find out **which server Bob is on**?
- How does the message actually get from server 7 to server 42?
- What happens if Bob is **not connected anywhere** (offline)?

> Concepts you may want to look up: **pub/sub** (e.g. Redis Pub/Sub), "connection registry" / "session store".

### Step 7: High-level design (draw it!)

Draw the components and the flow for:

- **Both online:** Alice → … → Bob's phone, and the ✓ / ✓✓ going back to Alice.
- **Bob offline:** Alice → … → stored → push notification → Bob opens the app later → gets the message.
- **Group message:** Alice sends to a group of 100.

Consider: load balancer, chat servers (holding connections), a connection registry, a database for messages, a message queue, the notification service from Exercise 3. Label arrows with **what actually happens**.

### Step 8: Deep dive

Answer at least two of these:

- **Server crash:** a chat server holding **50,000** connections crashes. What happens to those users? What happens to messages that were "in flight"?
- **Ordering:** Alice sends "A", then "B" quickly. How do you guarantee Bob sees "A" before "B", and that a group of 100 sees the same order?
- **Group fan-out:** one message to a 100-member group = up to 100 deliveries. Where does that work happen, and what if 30 members are offline?
- **Presence at scale:** 20 million online users. If every "online/offline" change is sent to all of a user's contacts immediately, how much traffic is that? How would you reduce it? (Look up **heartbeat**.)
- **Load balancing long-lived connections:** a normal load balancer spreads *requests*. What changes when each connection stays open for hours? What happens when you add a new chat server?

---

## Rules

- Write your answers under each step (like before) or in a new `my-solution.md` in this folder.
- Add your diagram to this folder (`diagram.drawio` is fine).
- Don't look up full solutions online. It's fine to look up individual concepts (e.g. "what is a WebSocket").
- Stuck on a step? Write what you think, mark it with ❓, and move on. I'll help with those parts during review.
