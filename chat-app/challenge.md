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
