# Exercise 5: Design a News Feed (like Twitter/X or Instagram home feed)

## The problem

Users post short updates and follow other users. When you open the app, you see your **home feed**: recent posts from everyone you follow, newest first.

```
Alice follows Bob, Carol and a celebrity (Star, 50 million followers)

Bob posts "Just finished my first marathon!"
        ↓  (a few seconds is fine)
Alice opens the app → her feed shows:
   Bob     · 2 min ago  · "Just finished my first marathon!"
   Star    · 10 min ago · "New album out Friday"
   Carol   · 1 hour ago · "Coffee recommendations?"
   ... scroll for more
```

In the chat app, one message went to **one person or up to 100 group members**. Here, one post can need to reach **millions** of people, and those people mostly **read** and rarely **write**. That changes where you do the work.

---

## Requirements

### Functional (what it must do)

1. **Create a post:** text (up to 280 characters), optionally with one image.
2. **Follow / unfollow** another user (one-way, like Twitter, not mutual friendship).
3. **Home feed:** posts from people you follow, **newest first**, loaded **20 at a time** as you scroll.
4. **User profile page:** all posts by one user, newest first.
5. **Like** a post, and show the like count.
6. _(Stretch)_ **Ranked feed:** instead of pure newest-first, show "most interesting" posts higher.

### Non-functional (how well it must do it)

- **Fast feed:** opening the feed should take **< 200 ms**.
- **Freshness:** a new post should appear in followers' feeds within **a few seconds** (not instant like chat, but not minutes either).
- **Highly available:** it's fine if a like count is slightly behind, or a post shows up a few seconds late. It's **not** fine if the feed doesn't load.
- **Scales** to hundreds of millions of users, including **celebrities with tens of millions of followers**.

### Out of scope

- Comments, replies, reposts/retweets, hashtags, search, trending topics.
- Direct messages (you already designed chat).
- Ads, spam detection, content moderation.
- Video.

---

## Numbers to work with (assumptions)

- **200 million** daily active users
- Each user **opens the feed 10 times per day** (each open loads 1 page of 20 posts)
- Each user creates **0.5 posts per day** on average (most people only read)
- A user follows **200** accounts on average
- A normal user has about **200 followers**; a **celebrity** can have **50 million**
- **Peak** traffic is **3×** the average
- A post's text + metadata ≈ **1 KB** stored
- **10%** of posts have an image, average image ≈ **500 KB**
- Posts are stored **forever** (estimate for **5 years**)

---

## Your task: work through these steps in order

### Step 1: Clarify requirements

Write down 2–3 questions you'd ask the interviewer, and the assumptions you'll make.

### Step 2: Back-of-the-envelope estimation

Calculate:

- **Writes:** posts per day, average posts per second, **peak** posts per second.
- **Reads:** feed loads per day, average per second, **peak** per second.
- The **read : write ratio**. What does that number tell you about where to optimise?
- **Storage** for post text over 5 years, and for images over 5 years. Where would each live?
- **Fan-out:** if every post were copied into each follower's feed, how many "feed inserts" is that per day? Per second?

> Tip: 1 day ≈ 100,000 seconds. Round aggressively.

### Step 3: API

Write the main endpoints (method, path, important parameters, what comes back):

- Create a post (with an image: how does the image get uploaded? You've seen this before in chat.)
- Get my home feed, page by page.
- Get a user's profile posts.
- Follow / unfollow.
- Like / unlike.

Then think about **pagination**: you load 20 posts, and while you're reading, 5 new posts arrive at the top. You scroll down and ask for "page 2". With **offset pagination** (`?page=2` / `?offset=20`), what goes wrong? What would you use instead? (Check your [glossary](../glossary.md)!)

### Step 4: Data model

- What tables or collections do you need? (Think: users, posts, follows, likes.)
- The **follows** table needs to answer two questions fast:
  - "Who does Alice **follow**?" (needed when building Alice's feed)
  - "Who **follows** Bob?" (needed when Bob posts)

  How do you make **both** fast?
- How do you store the **like count**? If you count rows in a likes table every time someone views a post, what happens for a post with 2 million likes?
- SQL or NoSQL for each piece? Why? (It's fine to use more than one database.)

### Step 5: Generating the feed (the core problem)

There are two basic approaches. Look them up (concept lookup is allowed) and explain each **in your own words**:

- **Fan-out on read** (also called **pull**): when Alice opens her feed, go and fetch the latest posts from each of the 200 people she follows, merge them, sort, return the top 20.
- **Fan-out on write** (also called **push**): when Bob posts, immediately insert that post into the stored feed of each of his followers. When Alice opens her feed, it's already built: just read it.

For each one, answer:

- How much work happens **when someone posts**? How much **when someone opens the feed**? Use your Step 2 numbers.
- How fast is opening the feed?
- What happens when **Star (50 million followers)** posts?
- What about users who **haven't opened the app in 6 months**?

Then **pick an approach** (or a mix of both) and say why.

> Concepts you may want to look up: **"fan-out on write vs fan-out on read"**, **"the celebrity problem" / "hot key"**, **hybrid feed**.

### Step 6: Caching

Reads outnumber writes by a lot, so caching matters most here.

- **What** would you cache? (Think: each user's feed, the posts themselves, user profiles, like counts.)
- Should a user's cached feed hold **full posts**, or only **post IDs**? What's the trade-off? (Hint: what happens when Bob edits his post, or its like count changes?)
- How many entries would you keep per user's feed? **All of them forever**, or only the latest N? What happens when someone scrolls past N?
- Estimate the memory: if you cache the latest **500 post IDs** (8 bytes each) for every daily active user, how much RAM is that?

### Step 7: High-level design (draw it!)

Draw the components and the flow for:

- **Posting:** Bob posts (with an image) → … → it ends up in his followers' feeds.
- **Reading:** Alice opens the app → … → sees 20 posts with names, avatars, images and like counts.
- **Celebrity:** Star posts → … → Alice still sees it in her feed.

Consider: load balancer, API gateway, post service, feed service, follow/graph service, a message queue + fan-out workers, feed cache, post cache, databases, object storage + CDN for images. Label arrows with **what actually happens**.

### Step 8: Deep dive

Answer at least two of these:

- **New follow:** Alice follows Dave right now. Do Dave's old posts appear in her feed? How? And when she **unfollows** Carol, what happens to Carol's posts already in her feed?
- **Deleted post:** Bob deletes a post that was already pushed into 200 feeds (or 50 million). How do you make it disappear?
- **Hot post:** Star's post gets **1 million likes in 10 minutes**. If every like does `UPDATE posts SET likes = likes + 1 WHERE id = …`, what happens to that row? How would you count likes instead?
- **Fan-out workers fall behind:** a big event happens and everyone posts at once. The fan-out queue backs up. What does the user see? What do you do? (Check [message-queues.md](../message-queues.md).)
- **Ranking (stretch):** instead of newest-first, you want "most interesting first". What signals would you use? Where in your design does the ranking happen: when the post is pushed, or when the feed is read?

---

## Rules

- Write your answers under each step (like before) or in a new `my-solution.md` in this folder.
- Add your diagram to this folder (`diagram.drawio` is fine).
- Don't look up full solutions online. It's fine to look up individual concepts (e.g. "what is fan-out").
- Stuck on a step? Write what you think, mark it with ❓, and move on. I'll help with those parts during review.
- **New: interview practice.** Before writing everything down in detail, try one **timed pass**: set a 45-minute timer and talk through Steps 1–7 **out loud** (as if to an interviewer), sketching on paper. Note where you got stuck. Then do the full written version at your own pace.
