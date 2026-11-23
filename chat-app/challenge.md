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
Questions

1. Are updates/stories included in this challenge.
2. What files do I need to take in consideration if media upload is an option in message
3. Do I have to add the blocking other user feature
4. Admin, user, kick etc features are needed in group as of now??

Assumptions

1. User's are connected throughout the world

### Step 2: Back-of-the-envelope estimation

Calculate:

- Messages per day, average messages per second, and **peak** messages per second.
- Storage per day and over **5 years**.
- How many **chat servers** are needed just to hold the open connections at peak?

> Tip: 1 day ≈ 100,000 seconds. Round aggressively.

messages per day = 2,00,00,00,000
messages per second = 23,148
peak messages= 23,148*3=69444 messages per second
storage = 2,00,00,00,000 * 300 bytes(100+200)= 600 gb per day = 600*365*5 = 1095 TB
Number of users active per second = 579 users per second
Which a single server can easily manage
But in case of 20 million users at a time, at peak = minimum 400 servers are needed

### Step 3: Real-time delivery (the core problem)

Bob's phone needs to receive Alice's message **as soon as it's sent**. Look up these approaches (concept lookup is allowed) and explain each **in your own words**:

- **Short polling** (Bob's app asks "anything new?" every few seconds)
- **Long polling**
- **WebSockets**
- _(Bonus)_ **Server-Sent Events (SSE)**

For each one: How does it work? How fast is it? What does it cost the server (think about 20 million users)? Then **pick one** and say why.

Short Polling - In this technique, we send http request to user in short intervals asking if there is anything for the user, if yes then give it, if no then hit again after short interval.

Long pollint - Here the request is send to the server, and it waits for the messages or the time expiry, once either happens, the response is send to the user and again new request is send

Issue - in both we are constantly sending http request to check if there is any update, unncessary extra use of bandwidth and resources.

Websockets - Here we establish an open connection between the user and the server, once its established then user and server can directly talk to each other, no need of http request for each message, its event driven. It is the best option for chatting and live streaming as users and servers are connected in real time.

Server Sent events - here instead of client requesting, server is the one who constantly streams the data to user client, in first request a connection is established then server sends the data to user after certain interval.

### Step 4: Protocol / API

- What kinds of messages travel over the connection between the app and the server? (Think: sending a chat message, the server confirming it, "delivered", "read", "typing"…) Write down the **fields** of a chat message.
- Which things are better as normal **REST** endpoints instead (e.g. loading old history, creating a group, logging in)?
- Alice's phone loses signal right after tapping Send, and it **resends** the message when it reconnects. How do you avoid Bob getting it **twice**? (Check your [glossary](../glossary.md)!)

1. Kinds of messages, we are dealing with text messages as of now, at most we will include media files. In case you are asking about the messages in between the client and websocket server than there we are using event driven architecture where everything is event based.
   there are various way, like broadcast etc.
   Types can me
   {'messageSend',{messageType:'video',mediaId='s3 bucket name/id of media',messageId:"", sendTo:"userId",userId:"userId",messageStatus:"pending"}}
   {'messageReceived',{ messageType:'message', sendby:'userid, to:'userid',messageId:"", timestamp:time}}
   {'groupMessageReceived',{ messageType:'image',mediaId='s3 bucket name/id of media',messageId:"", sendby:'userid, toGroup:'groupId',timestamp:time}}
   |
   |\_ from groupId extract all users id and send the messageTeceived event
   {"GroupCreated",{groupName:'', members:[],settings:{admins:[],messageDisappear:"true",autoDeleteMessage:'false'}}}
   {"AddMemberToGroup",{groupId:"",userId:""}}
   {"RemoveUserFromGropu",{groupId:"",userId:""}}
   {"DeleteGroup",{groupId:""}}
   {"AddedToTheGroup",{userId:"",groupId:""}}

2. Apart from direct communication between users, individual or in group. Rest of the things can use the REST endpoints, media upload, authentication, authorization, creating room, managing room, adding/removing users, getting old messages etc.
3. For this issue, we can generate unique id for each message, like using a combination of usesrid+plus message number, something like this. So once the user comes back online, a request will try to resend the message, and it will check if the message already exists in the db or not, if yes, then do nothing and update the ui state, other wise deliver the message.

### Step 5: Data model

- What tables or collections do you need? (Think: users, chats/conversations, group members, messages, delivery/read status.)
- The most common query is **"give me the latest 50 messages of chat X"** (and then the 50 before those, as the user scrolls up). How do you make that fast?
- **Ordering:** can you sort messages by the timestamp from the sender's **phone clock**? What could go wrong? What would you use instead?
- SQL or NoSQL? Why?

1. users, group, groupMembers, userHomeRegion, messages, missedNotification (for offline users)
2. For this we can do pagination, we will load only the first 50 messages, and when user reaches the threshold, say 20th message from bottom. Then we will hit the pagination api to send next 50 messages. This way it will be fast yet seamless with better UX
3. We can't rely on user's phone clock as it can be mismatched and the time isn't same through the globe, instead we will give each message an id, and we will store it with that id as the primary key, then indexing on it for easy and fast read.
4. For this, I am going to use SQL as the majority of data is sorted.

Also created a data model diagram, check that out too.

### Step 6: Routing between chat servers

Alice is connected to **chat server 7**. Bob is connected to **chat server 42**. Alice's message arrives at server 7.

- How does server 7 find out **which server Bob is on**?
- How does the message actually get from server 7 to server 42?
- What happens if Bob is **not connected anywhere** (offline)?

1. For this first we will do region based directory, which will direct the request to where we might be keeping users id and the last connected redis cache, yes we will be using redis cache to keep the status of user. using this user sitting in us can communicate with user in japan. US user will connect to a server, that server will look at the other users region on the one massive directory, from there it will find its redis through which it will find to which server this user is connected to. Then a connection can be established between them.

2. So the user a is connected to server 47 and user b is connected to server 2. Now the user a will send the message and since the server 47 knows to which server the user b is connected to, it will directly send the event/message to that server and user b will receive the message.
3. So, during first connection, if the user can be found in any of the redis cache, then we can assume two cases
   -user is offline
   -user doesn't exists

In case of offline, we will store the message in the pending db, so once the user connects back, first it will check the db if there is any updates for them. In case of user doesn't exists, then it doesn't exists, can't do anything about it, just the show the user that your target user doesn't exist.

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
- **Load balancing long-lived connections:** a normal load balancer spreads _requests_. What changes when each connection stays open for hours? What happens when you add a new chat server?

1. if the chat server goes down, so does the users with it, but they will be alloted another server. Talking about the messages, they will be lost and since we already saved them in the db, they will be fetched once the user comes back online.
2. For each message we will be generating individual ids, using these ids we can ensure that the order is preserve. Plus we are following the pub/sub feature so the the message is published to everyone in the same instance, which preserves the order, irrespective of how fast one send it.
3. For this we already have a redis pub/sub server which is responsible for the delivery of these messages, plus for the offline users, we will be adding these messages in the missed db table, so once they come online, they will fetch them from there.
4. For this, we won't be sending the details of every contacts only the most recent ones, other wise when user opens the other user's chat, then we will fetch, we will look if user is online or offline. This will readuce the traffic quite a lot
5. For this we have a dedicated cluster of websockt servers, which will be handeling the live connections and they are pretty reliable and can hold large number of connections at a time, when we add a new chat server, then it will reduce the load on the other server although each server can easily handle 50,000 connections at a time, so its not a big deal for it.

---

## Rules

- Write your answers under each step (like before) or in a new `my-solution.md` in this folder.
- Add your diagram to this folder (`diagram.drawio` is fine).
- Don't look up full solutions online. It's fine to look up individual concepts (e.g. "what is a WebSocket").
- Stuck on a step? Write what you think, mark it with ❓, and move on. I'll help with those parts during review.


---

# 📌 Review notes: improved solution (reference)

> Compare these with your answers above.
>
> The improved diagrams are on a new page called **"Improved (reference)"** in both [diagram.drawio](diagram.drawio) and [datamodel.drawio](datamodel.drawio). Use the page tabs at the bottom of draw.io to switch to them. Your own drawings are untouched on **Page-1**.

## The one idea that fixes most things

**The server gives every message a sequence number per chat (`seq` = 1, 2, 3…), and messages are stored by `(chat_id, seq)`.**

That single decision solves four problems:
- **ordering:** everyone sorts by `seq`
- **pagination:** `seq < X LIMIT 50`
- **syncing after being offline:** `seq > lastSeq`
- **read receipts:** "read up to seq 1041"

Sharding by `chat_id` then handles storage.

---

## Step 1: Clarify requirements

**Questions:**
1. Text only, or images and videos too? What's the size limit?
2. Group admin features (add, remove, admin-only messages)? Blocking users?
3. How long is history kept? Must it stay on the server, or can it live on the phone?
4. Is one device per user enough, or do we need phone + web in sync?

**Assumptions:**
- Text + images.
- One device per user.
- Groups of up to 100.
- History kept on the server for 5 years.
- Users all over the world.
- No end-to-end encryption.

---

## Step 2: Estimation

| What | Calculation | Result |
|---|---|---|
| Messages per day | 50M users × 40 | **2B per day** |
| Average per second | 2B ÷ 100k s | **≈ 20k/s** |
| Peak per second | × 3 | **≈ 60–70k/s** |
| Storage per day | 2B × **200 B** (already includes the text) | **≈ 400 GB/day** |
| Storage over 5 years | 400 GB × 365 × 5 | **≈ 730 TB** (≈ **2 PB** with 3 copies) |
| Chat servers | 20M concurrent ÷ 50k each | **400**, so plan for **~500** for failures and peaks |

**Takeaways:**
- For chat, the important number is **concurrent connections**, not requests per second.
- About 2 PB means **the messages database must be sharded**.

---

## Step 3: Real-time delivery

| Approach | How it works | Speed | Cost at 20M users |
|---|---|---|---|
| Short polling | The app asks "anything new?" every few seconds | Up to the polling interval late | 20M ÷ 5 s = **4M requests/s**, almost all returning nothing |
| Long polling | The server holds the request open until there's news or it times out, then the app asks again | Fast | A new HTTP request after every message or timeout; lots of reconnecting |
| **WebSocket** | One **TCP** connection stays open, and either side can send at any time | **Instant** | One open connection per online user (~50k per server); no wasted requests |
| SSE | The server streams events over HTTP whenever they happen. **One-way only** (server → client) | Instant | The client still needs separate HTTP requests to *send* |

**Pick: WebSocket.** Chat sends in both directions (messages, receipts, typing), and you need instant delivery without millions of empty requests.

---

## Step 4: Protocol / API

**Events over the WebSocket.** Each has a `type` plus data:

| Direction | Event | Purpose |
|---|---|---|
| app → server | `message.send` | Send a message |
| server → sender | `message.ack {clientMsgId, seq}` | "Saved" → **✓** |
| server → receiver | `message.new` | A new message |
| receiver → server → sender | `receipt.delivered {chatId, upToSeq}` | **✓✓** |
| receiver → server → sender | `receipt.read {chatId, upToSeq}` | **blue ✓✓** |
| both ways | `typing {chatId}` | Temporary: not stored, not retried |
| server → app | `presence {userId, online / lastSeen}` | Only for chats the user has open |
| server → app | `group.memberAdded`, `group.updated` | Notices *after* a REST change |

**Fields of a chat message:**
```json
{
  "type": "message.send",
  "clientMsgId": "8f1c…",          // generated by the phone, used to detect duplicates
  "chatId": "c_123",               // the same field works for 1:1 chats and groups
  "kind": "text",                  // text | image | video
  "text": "Hey, are you free tonight?",
  "mediaId": null                  // set for images, after the upload to S3
}
// The server adds: seq, senderId, createdAt (server time). The client never sets status or seq.
```

**REST instead:**
- `POST /login`
- `GET /chats` (the chat list)
- `GET /chats/{id}/messages?before=1041&limit=50` (history)
- `POST /groups`, `POST /groups/{id}/members`, `DELETE /groups/{id}/members/{userId}`
- `POST /media/upload-url` (signed URL)

The rule: **real-time pushes go over the WebSocket. Request → response actions go through REST.**

**Avoiding duplicates on resend (idempotency):**
- The phone keeps the **same `clientMsgId`** when it resends.
- The DB has a **unique constraint / idempotency key on `(sender_id, client_msg_id)`**, so the second insert fails.
- The server then just **returns the original ack** (with the same `seq`).
- Don't "check whether it exists, then insert". Two copies can arrive at the same moment and both pass the check (a check-then-act race).

---

## Step 5: Data model

The full diagram is on the "Improved (reference)" page of `datamodel.drawio`.

```
users              (id PK, name, phone UNIQUE, email, created_at)
user_home_region   (user_id PK, region)                    ← tiny, copied to every region
chats              (id PK, type 'direct'|'group', last_seq, created_at)
groups             (chat_id PK/FK, name, description, created_by, only_admins_can_send, disappearing_after)
direct_chats       (user_low, user_high) PK → chat_id      ← finds "the chat between Alice and Bob"
chat_members       (chat_id, user_id) PK, role, joined_at, muted_until,
                   last_delivered_seq, last_read_seq        ← INDEX (user_id) for "my chats"
messages (NoSQL)   partition key chat_id, sort key seq DESC,
                   sender_id, client_msg_id, kind, text, media_url, created_at
```

**Key changes from your version:**
- **`chats` replaces `msgfrom` / `msgto` / `_individual`.** A 1:1 chat is just a chat with 2 members, so 1:1 chats and groups use the same code and the same query.
- **`pendingmessages` is removed.** Offline users sync with `seq > lastSeq` from `messages`. A second copy means two writes and a risk of the copies disagreeing.
- **`msgStatus` on each message is replaced** by `last_delivered_seq` / `last_read_seq` on each member row. This works for groups, where 100 people each have their own status, and it's one number to update instead of a row per message.
- **Added the missing `text` / `media_url`** columns.
- **Composite primary key `(chat_id, user_id)`** on members, so the same user can't be added twice.

**Latest 50 messages, then scrolling up (cursor pagination):**
```sql
SELECT * FROM messages WHERE chat_id = 'c_123' ORDER BY seq DESC LIMIT 50;              -- newest page
SELECT * FROM messages WHERE chat_id = 'c_123' AND seq < 1041 ORDER BY seq DESC LIMIT 50; -- scroll up
```
- Both read **one partition that's already sorted**, so they're fast whether the chat has 100 messages or 100,000.
- Avoid `OFFSET 5000`. The database still has to walk past all 5,000 rows.

**Ordering:**
- Phone clocks can be wrong or deliberately changed, and they differ between phones.
- Use a **server-assigned `seq` per chat**: an atomic counter (`INCR chat:X:seq` in Redis, or `last_seq + 1` on the `chats` row).
- The order messages **reach the server** is the official order. Everyone sees the same order.
- A random message ID does **not** give you order. You need the counter.

**SQL or NoSQL:**
- **Messages:** wide-column NoSQL (Cassandra / ScyllaDB / HBase). The reasons:
  - The writes are huge (2B per day, about 70k/s at peak).
  - The data is about 2 PB.
  - It's always read the same way: one chat, sorted by seq.
  - Partitioning by `chat_id` spreads the data across machines automatically.

  Messenger (HBase) and Discord (Cassandra, later ScyllaDB) use this setup.
- **Users, chats, groups, members:** SQL. The data is small and relational, and you need transactions and unique constraints.
- A **SQL database sharded by `chat_id`** would also work for messages. Either way, the real answer is: **shard by chat**.

---

## Step 6: Routing between chat servers

1. **Finding Bob:**
   - Bob's **home region** comes from the `user_home_region` table. It's copied to every region, so this is a local lookup.
   - If his home is another region, server 7 **forwards the message to that region**.
   - In that region, the **connection registry** (Redis: `bob → server-42`, with a TTL refreshed by heartbeats) gives his server. It's an O(1) lookup in the same data center, about 1 ms.
2. **Moving the message:**
   - Server 7 runs **`PUBLISH "server-42"`** on **Redis Pub/Sub**.
   - Server 42 subscribed to its own channel when it started, so it receives the message. It finds Bob's socket in its local `userId → socket` map and sends the message.
   - Servers don't connect to each other directly.
3. **Bob is offline (not in the registry):**
   - The message is **already saved** (it's always saved first).
   - Put a push job on the **notification queue** (Exercise 3).
   - When Bob opens the app, it sends `sync {chat: lastSeq}` and gets everything newer from `messages`.
   - Whether a user *exists* is checked in the `users` table. The registry only knows who is **online**.

---

## Step 7: High-level design

The full diagram is on the "Improved (reference)" page of `diagram.drawio`.

**Both online (the numbers match the diagram):**
```
1. Alice → LB → server 4:        message.send {clientMsgId, chatId, text}
2. server 4 → Messages DB:       seq = next for chat X; INSERT (idempotent)
3. server 4 → Alice:             message.ack {seq}                      → ✓
4. server 4 → registry:          bob → server-8   (often already cached)
5. server 4 → Pub/Sub:           PUBLISH "server-8"
6. Pub/Sub → server 8            (subscribed to its own channel)
7. server 8 → Bob:               message.new
8. Bob → server 8:               receipt.delivered {upToSeq}
9. server 8:                     update last_delivered_seq; PUBLISH "server-4"
10. server 4 → Alice:                                                   → ✓✓
```

**Bob offline:**
```
Steps 1–4 → registry has no entry for Bob → push job → notification queue → workers → APNs/FCM → "New message from Alice"
Bob opens the app → connects → sync {chat X: seq > lastSeq} → messages from the DB → receipt.delivered → Alice ✓✓
```

**Group of 100:**
```
Save the message ONCE (chat_id, seq) → ack Alice ✓
→ read chat_members (cached) → registry MGET for all 100 members → group them by server
→ PUBLISH once per SERVER (100 members on 40 servers = 40 publishes, not 100)
→ 30 offline members → push jobs; they sync later
```

**Fixes compared with your diagram:**
- WebSocket runs over **TCP**, not UDP.
- The phone connects **through the load balancer straight to a chat server**, and the upgrade happens on that connection. HTTP servers don't hand it off.
- **Every** message is saved before it's published, not only when the user is offline.
- Draw **two** chat servers so the relay (server A → Pub/Sub → server B) and the ✓✓ path back are visible.
- Add the **offline → notification queue → push** path, and the **group fan-out**.
- Messages DB: **shards** (by `chat_id`). Replicas only make copies; they don't add storage.
- Media: your signed-URL upload is ✅. Downloads should come **through the CDN**, with S3 as the origin.

---

## Step 8: Deep dive (all five)

**1. Server crash (50k connections):**
- Those phones see the connection drop. They **reconnect with exponential backoff + jitter**, so 50k phones don't all hit the load balancer in the same second (a *reconnect storm*). Each lands on a healthy server.
- The dead server's registry entries **expire** (TTL / missed heartbeats). New connections write fresh entries.
- **In-flight messages:**
  - Anything Alice got **✓** for is **in the DB**, so nothing is lost. Bob gets it through `sync` after he reconnects.
  - Anything Alice sent but got **no ✓** for is **resent by her phone with the same `clientMsgId`**, so there's no duplicate.

**2. Ordering:**
- **Within one sender:** one WebSocket (TCP) keeps "A" before "B", and the server handles each connection's messages one at a time, so "A" gets the lower `seq`.
- **Across senders and the whole group:** the **server-assigned `seq` per chat** is the single official order.
- Every client **places messages by `seq`**, not by arrival time. If seq 41 is missing when 42 arrives, the client fetches the gap.
- Pub/Sub alone doesn't guarantee order between different publishers.

**3. Group fan-out:**
- The work happens on the **sender's chat server**, or on **fan-out workers behind a queue** for very large groups.
- The message is **stored once**, then delivered **once per server** (grouped using the registry).
- **30 offline members:** nothing extra is stored. Each gets a push job, and they sync from `seq` when they reconnect.

**4. Presence at scale:**
- **Rough traffic:** 20M online users × ~10 online/offline changes per day × ~100 contacts each ≈ **230k presence events/s**. That's **more than the chat messages** (≈70k/s).
- **Ways to reduce it:**
  - Send presence **only to people who have that chat open right now**. Everyone else fetches it when they open the chat. (This was your idea ✅.)
  - **Heartbeat:** the phone pings every ~30 s, and two missed pings mean offline. Store `last_seen`.
  - **Debounce:** wait a few seconds before announcing "offline", so a flaky connection doesn't send online → offline → online.
  - For very large groups, don't show presence at all.

**5. Load balancing long-lived connections:**
- A normal load balancer balances **requests**. Here it only places a connection **once**, and that connection stays for hours.
- So a **newly added server gets almost no load** at first. Only new connections go to it.
- **Fixes:**
  - Use a **least-connections** rule, so new connections prefer the emptiest server.
  - Use a Layer 4 (TCP) load balancer that supports WebSockets.
  - To remove or redeploy a server, **drain** it: stop sending it new connections, then tell its clients (in batches, with jitter) to reconnect elsewhere.
  - Same idea for rebalancing: gradually ask some clients on busy servers to reconnect.

---

## Key takeaways

1. **Server-assigned `seq` per chat + the `(chat_id, seq)` key** gives you ordering, pagination, sync and receipts.
2. **Save before you send.** ✓ means "saved"; delivery comes after. That's why crashes don't lose messages.
3. **Registry = lookup only. Pub/Sub = relay only. DB = the truth.** Pub/Sub can drop messages safely because the DB has them.
4. **Store once, deliver per server.** Never copy a message per recipient.
5. **Shard by chat** for about 2 PB. **Replicas are for failover**, not for capacity.
6. Long-lived connections change load balancing: **least-connections, draining, reconnect with jitter**.
