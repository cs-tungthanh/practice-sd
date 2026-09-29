# Mock Interview — Design Instagram

**I** = Interviewer · **C** = Candidate (you)

Spoken style. Short sentences. Read the **C** lines out loud.

---

## 0. Opening (0:00)

**I:** Hi, thanks for joining. Today I'd like you to design Instagram. Take it wherever you want.

**C:** Sure, thanks. Before I draw anything, I'd like to agree on the scope. Is that okay?

**I:** Go ahead.

---

## 1. Functional requirements (0:01 – 0:04)

**C:** So for the main features, I'm thinking three things.
First, users can create a post. A post has a photo or a video, plus a caption.
Second, users can follow other users.
Third, users can see a feed. The feed shows posts from the people they follow.

**I:** What kind of feed? Ranked, like real Instagram?

**C:** To keep it simple, I'd like to make it chronological. Newest posts first. Is that okay with you?

**I:** That's fine.

**C:** Great. And I'd like to put some things out of scope. Likes, comments, stories, search. They're important, but they make the design much bigger. If we have time at the end, I'm happy to come back to them.

**I:** Agreed. Leave them out.

---

## 2. Non-functional requirements (0:04 – 0:08)

**C:** Next, the non-functional side. First question — what scale are we talking about? Full Instagram scale?

**I:** Yes. Assume around 500 million daily active users.

**C:** Okay, that's big. So a few things.

**C:** First, CAP theorem. I'd choose **availability over consistency**. If someone in Germany posts, and I'm in the US, it's fine if I see it a few seconds later. Nobody gets hurt. So **eventual consistency** is okay here.

**I:** Any place where you'd want strong consistency?

**C:** Not for these three features, no. Maybe for things like payments or account settings. But those are out of scope.

**C:** Second, **low latency feed loading**. I'd say under 500 milliseconds. Under 200 feels instant. Around half a second is still fine. After one second, it feels slow.

**C:** Third, **low latency media delivery**. Photos and videos can be big. I don't want users to see an empty grey box while the image loads. So same target — about 500 milliseconds for media to start showing.

**C:** And last, the system must **scale** to 500 million DAU and still meet those latency numbers.

**I:** Good. Let's move on.

---

## 3. Core entities (0:08 – 0:10)

**C:** Let me list the core entities. These are the main nouns in the system. I won't do the full schema yet. I'll add columns later when I need them.

- **User**
- **Post** — the caption and metadata
- **Media** — the actual photo or video file
- **Follow** — a pair: user A follows user B

**I:** Why split Post and Media?

**C:** Because they're very different kinds of data. The post is small, structured data. The media is a big binary file. I already know I'll store them in different places. So I want to keep them separate from the start.

---

## 4. API (0:10 – 0:13)

**C:** Now the API. I'll make at least one endpoint for each requirement.

**C:** To create a post:

```
POST /posts
body: { media, caption }
→ returns postId
```

**C:** I know I can't really put a big video in one request body. I'll fix that later in the deep dive. For now I'll keep it simple.

**C:** To follow a user:

```
POST /follows
body: { followedId }
→ 200 OK
```

**I:** Where's the follower's ID?

**C:** I take it from the JWT, or the session, in the header. I don't trust a user ID from the body. If I did, someone could make other people follow them. You *can* send it in the body, but then the server must check it matches the JWT.

**C:** And to get the feed:

```
GET /feed?cursor={cursor}&limit={n}
→ returns Post[]
```

**C:** I'm using **cursor-based pagination**. The feed changes all the time, so a cursor is safer than page numbers.

**I:** Makes sense.

---

## 5. High-level design (0:13 – 0:25)

**C:** Okay, let me draw the high-level design. The goal here is just to make each requirement work. I'll make it fast later.

### 5.1 Create a post

**C:** I have a client. It goes through an **API gateway**. The gateway does routing, and also auth and SSL termination.
Behind it, I have a **Post Service**. It writes to a **database**.

**C:** The posts table looks like this:

| Column | Note |
|---|---|
| postId | |
| userId | the creator |
| mediaUrl | link to S3 |
| caption | |
| createdAt | |

**C:** And the media goes to **blob storage**, like **S3**.

**I:** Why not just put the photos in the database?

**C:** You *could*. But it'd be very expensive. Databases are built for structured data — indexes, fast queries. A photo is just a big blob of bytes. We don't need to query inside it. Blob storage is a cheap key-value store for files. So it's the right tool.

**C:** So the flow is: request comes in → Post Service uploads media to S3 → gets back the link → writes a new row with that link.

### 5.2 Follow a user

**C:** For follows, I add a `follows` table: `followerId`, `followedId`, `createdAt`.
For now, same database. I'll check later if that's still a good idea at scale.

**C:** But I'll make a separate **Follow Service**.

**I:** Why a separate service, if it's the same database?

**C:** Two reasons. Separate responsibility, and I can scale them separately. Also, if the follow service breaks, users can still create posts. Posting is the most important thing.

### 5.3 Get the feed — simple version

**C:** For the feed, I'll start with the simplest thing. It won't be fast enough, but it works.

1. Get everyone the user follows.
2. Get recent posts from each of those people.
3. Merge and sort them by time, in memory, in the Post Service.
4. Return the list.

**I:** Which database would you pick?

**C:** Postgres would work fine. Actually, Instagram uses Postgres. But at this scale I'd need to handle sharding myself, or use something like Citus. To keep the talk simple, I'll go with **DynamoDB**.

**C:** So the keys:

- **follows**: partition key = `followerId`, sort key = `followedId`. So I can quickly get everyone I follow. And the pair is unique.
- **posts**: partition key = `userId`, sort key = `postId`.

**I:** How do you sort posts by time with that?

**C:** I make `postId` **monotonically increasing**. Each new post gets a bigger ID. Something like a Snowflake ID. So sorting by ID is sorting by time. Or I could use `createdAt` plus ID as the sort key. Same result.

**C:** So now it's correct. But it's still slow. That's my first deep dive.

---

## 6. Deep dive 1 — Fast feed (0:25 – 0:37)

**I:** Okay. Why is it slow?

**C:** This pattern is called **fan-out on read**. When you open the feed, we go out to every person you follow. If you follow a thousand people, that's a thousand lookups, then a big sort. All inside 500 milliseconds.

**C:** Let me do quick math. 500 million users. Maybe 5 feed loads a day. That's 2.5 billion feed loads per day. A day is about 100,000 seconds. So around **25,000 feed loads per second**. And each one fans out to hundreds of queries.

**C:** So three problems:
- It's **expensive**. Lots of database reads. With DynamoDB we pay per read.
- It's **wasteful**. A post from Cristiano Ronaldo gets read hundreds of millions of times.
- It's **slow**, and it gets slower the more people you follow. So our most active users get the worst experience.

**I:** What would you try first?

**C:** A simple fix is a **cache in front of the posts table**. Key = userId, value = their recent posts. That helps the "wasteful" problem.
But it doesn't fix the real problem. We still fan out on every read. It treats the symptom, not the cause.

**I:** So what's the real fix?

**C:** **Pre-compute the feed.** Do the heavy work when someone *posts*, not when someone *reads*. This is called **fan-out on write**.

**C:** Here's how it works. When I create a post:
1. Find everyone who follows me.
2. Add my post to each of their feeds, in a **feed cache**. I'd use **Redis**.

**C:** Then when you open your feed, it's just one lookup by your user ID. Very fast.

**C:** One small thing — I need a new index. Now I'm asking "who follows *me*", which is the opposite direction. So in DynamoDB I add a **global secondary index**: partition key = `followedId`, sort key = `followerId`.

**I:** What exactly do you store in the cache? The full posts?

**C:** Good question. Let me compare.

**C:** Option one: store only **post IDs**. Then on read, I look up each post in the database. That's always fresh. If someone edits a caption, I see it. But I'm back to many database reads.

**C:** Option two: store the **full post data** in every feed. Fast to read. But if someone edits a post, I have to update it in millions of feeds. That's bad.

**C:** So I'd do a **mix of both**. Two caches:
- `userId → list of postIds` — the feed
- `postId → post data` — shared by everyone

**C:** On read, I get the list of IDs, then look up each post in the second cache. Both are in memory, so it's fast. And if someone edits a post, I update only two places: the database and one cache entry.

**I:** Won't the feed list grow forever?

**C:** Yes, so I'd use a Redis **sorted set**, sorted by time. And I trim it. For example, keep only the last week, or the last few hundred posts. If someone scrolls further back — which is rare — I fall back to the slow fan-out on read.
And for the post data cache, I use **LRU** or **LFU** eviction. On a miss, I read from DynamoDB.

**I:** Walk me through the write path end to end.

**C:** Sure.
1. User creates a post. It hits the Post Service.
2. Post Service writes to DynamoDB, and puts the post ID on a **message queue** — Kafka or SQS.
3. A group of **fan-out workers** read from the queue.
4. Each worker gets the author's followers from DynamoDB.
5. For each follower, it adds the post ID to their feed in Redis.

**C:** I use a queue because this work is heavy and it can be async. The user doesn't need to wait for it.

**I:** And the read path?

**C:** User asks for the feed → Post Service reads their list of post IDs from Redis → gets each post from the post cache → any misses go to DynamoDB → return to the client.

**I:** Okay. Any problem left?

**C:** Yes, one big one. **Celebrities.** Cristiano Ronaldo has hundreds of millions of followers. If he posts, one message means hundreds of millions of cache writes. That would block the queue. And many of those followers may not even open the app today. So it's a waste.

**I:** How do you handle that?

**C:** A **hybrid**. Fan-out on write for normal users. Fan-out on read for celebrities.
I set a limit — say, **100,000 followers**. That number is a guess. We'd tune it with real data.

**C:** Below the limit, we pre-compute like before. Above it, we don't. At read time, the Post Service does two things at the same time:
- Reads my pre-computed feed from Redis.
- Gets recent posts from the few celebrities I follow.

Then it merges them. I follow maybe three or four celebrities, not thousands. So that part is cheap. And celebrity posts are very hot, so they're almost always in the post cache.

**I:** Nice. Let's talk media.

---

## 7. Deep dive 2 — Media upload and delivery (0:37 – 0:45)

**I:** How big can files be?

**C:** Let me ask you — what limits should we support?

**I:** Photos up to 8 MB. Videos up to 4 GB.

**C:** Okay. So two problems: **upload** and **download**.

### 7.1 Upload

**C:** For upload, I can't send 4 GB in one POST request. Every layer has a size limit. For example, AWS API Gateway has a 10 MB limit. So photos might pass, but videos won't.

**C:** So the answer is: **chunk it**, and upload **directly to S3**, not through my services.

**C:** Here's the new flow:
1. Client calls `POST /posts` with only the caption and media details — size and type.
2. Post Service creates a row with `status = uploading`. No media link yet.
3. Post Service asks S3 for **pre-signed URLs**.
4. Client uploads the chunks straight to S3, using **multipart upload**. S3 joins the chunks back together.

**I:** How do you know when the upload is done?

**C:** Two options.

**C:** Option one: **trust the client**. S3 tells the client "done", and the client calls PATCH on the post to set it to `uploaded`. It's simple. But I don't love it. A bad client could lie, and then our database has bad state. That's painful to debug.

**C:** Option two, which I prefer: **S3 event notifications**. When the upload finishes, S3 sends an event. A small worker, or a Lambda, listens for it. It updates the post: `status = uploaded`, and sets the media link.

**I:** How does the worker know which post it belongs to?

**C:** By the **object key**. When I upload to S3, I choose the key. I use the media ID or post ID as the key. So the event already tells me which row to update.

### 7.2 Download

**C:** For download, the simple way is: client gets the S3 link and downloads from S3.
The problem is distance. If the user is in Singapore and my bucket is in US West, the data travels across the world. That's slow, especially for big files.

**C:** So I add a **CDN**. It's a cache for files, with servers all around the world. The Singapore user gets the file from a server close to them. If the CDN doesn't have it, it fetches from S3 once, then caches it.

**I:** And the 4 GB video on a slow 3G phone?

**C:** Right, the CDN isn't enough then. So I add **media processing**. After upload, S3 sends another event. An async worker makes several versions — low, medium, and high resolution. For video, also split into small segments. I could use a service like AWS MediaConvert or Cloudinary.

**C:** I save all versions back to S3. And in the post row, instead of one link, I store a small **manifest** — a map from resolution to URL.

**C:** Then the client is smart. It knows its own network speed. It picks the right resolution from the manifest and gets it from the CDN. On a bad connection, it gets the small version. So it still loads fast.

---

## 8. Scale and wrap-up (0:45 – 0:50)

**I:** We have a few minutes. Anything else on scale?

**C:** Yes, let me go through it quickly. A lot of the scaling is already done by the design.

- **Feed**: the hybrid fan-out is the main piece.
- **Media**: CDN plus chunked upload. And S3 scales almost without limit.
- **Services**: Post and Follow services are **stateless**, so I scale them **horizontally**.
- **Fan-out workers**: also stateless. I add more workers when the queue gets long.
- **Queue**: partition by **post ID**. So `postId mod N`.
- **Database**: DynamoDB handles sharding for me. If we used Postgres, I'd need to decide the shard key myself.
- **Redis**: shard by **user ID**. Maybe split into two clusters — one for feeds, one for post data.

**C:** Let me do some quick storage math.
Say 100 million new posts per day. Average media is 2 MB. That's about **200 TB of media per day**. That's a lot, but it's S3, so it's okay. To save money, I'd move old media that nobody views to cheaper storage, like **S3 Glacier**.
Post metadata is maybe 1 KB per post. So about **100 GB per day**. DynamoDB handles that fine.

**C:** General rule: hot data stays close and fast — CDN, memory. Cold data moves to cheaper and slower storage.

**I:** Good. Let's say we had more time. What would you go deeper on?

**C:** Probably two things. First, cache consistency — how we keep Redis and DynamoDB in sync when posts are edited or deleted. Second, how we handle a normal user who suddenly goes viral and crosses the celebrity limit.

**I:** Great. That's all the time we have. Thanks, that was really solid.

**C:** Thank you. I enjoyed it.

---

## Phrases to reuse

| Situation | Say this |
|---|---|
| Starting | "Before I draw anything, I'd like to agree on the scope." |
| Cutting scope | "They're important, but they make the design much bigger." |
| Starting simple | "I'll start with the simplest thing. It won't be fast enough, but it works." |
| Admitting a gap | "I know this won't work for big files. I'll fix it in the deep dive." |
| Comparing options | "Let me compare. Option one… Option two… So I'd do a mix of both." |
| Picking a number | "That number is a guess. We'd tune it with real data." |
| Asking for info | "Let me ask you — what limits should we support?" |
| Wrap-up | "If we had more time, I'd go deeper on…" |

## Notes on the source video

- The video says 250,000 feed loads per second. The correct math is 2.5 billion ÷ 100,000 ≈ **25,000 per second**. I fixed it above.
- The video mixes up the storage math at the end. With 100M posts/day × 2 MB, it's **~200 TB/day**. I used that.
