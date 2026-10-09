# SnapShare Scaling Plan

SnapShare is a photo-sharing app where users upload photos and scroll a feed of photos from people they follow.

## 1. Assumptions

| Assumption | Value |
|---|---|
| Registered users | 10,000,000 |
| Daily active users (DAU) | 10% of registered users |
| Uploads per active user per day | 1 photo |
| Feed page views per active user per day | 50 |
| Average original photo size | 2 MB |
| Thumbnail size | 50 KB (one per photo) |
| Seconds per day | 86,400 |
| Peak traffic | 5x the average |
| Units | Decimal (1 TB = 1,000,000 MB) |

Other simplifications: traffic is spread evenly over the day for the average; peak is 5x that. Replication and backups are not counted in the raw storage figure.

**Daily active users:** 10,000,000 x 10% = **1,000,000 DAU**

## 2. Estimates

### Uploads per second
- Uploads per day: 1,000,000 x 1 = 1,000,000
- Average: 1,000,000 / 86,400 = **about 12 uploads/s**
- Peak (5x): **about 58 uploads/s**

### Feed views per second
- Feed views per day: 1,000,000 x 50 = 50,000,000
- Average: 50,000,000 / 86,400 = **about 579 views/s**
- Peak (5x): **about 2,900 views/s**

### Storage per year
- Originals per day: 1,000,000 x 2 MB = 2 TB
- Thumbnails per day: 1,000,000 x 50 KB = 50 GB
- Total per day: 2.05 TB
- Originals per year: 2 TB x 365 = 730 TB
- Thumbnails per year: 50 GB x 365 = 18.25 TB
- **Total photo storage per year: about 748 TB (roughly 0.75 PB)**

Metadata (user id, photo id, caption, timestamp, URL) is tiny by comparison: about 500 bytes x 1,000,000 per day is roughly 180 GB per year.

## 3. Read-heavy or write-heavy?

**SnapShare is read-heavy.** Each active user views 50 feed pages for every 1 photo uploaded, so reads outnumber writes about **50 to 1** (579 reads/s vs 12 writes/s on average).

What this means for the design:
- Optimise the read path first: cache hot data, serve images from a CDN, and add read replicas to the database.
- Writes are few, so a single primary database can handle them for a long time.
- Slow work on the write path (thumbnails) can be moved off to a background queue so uploads stay fast.
- Slightly stale reads are acceptable (a new photo appearing a few seconds late is fine), which makes aggressive caching safe.

## 4. Why photos should not live in the database

- **Size:** About 730 TB of originals per year would make the database enormous, slow to back up, slow to restore and expensive to replicate. A database is built for small, structured rows, not multi-megabyte blobs.
- **Performance:** Large binary reads compete with queries for memory, disk and network, slowing down everything else.
- **Cost:** Database storage is far more expensive per GB than object storage.
- **Delivery:** A database cannot serve files to a CDN or browsers efficiently, while object storage serves files over HTTP directly.

**Where they go instead:** the photo files (original and thumbnail) go in **object storage** (such as Amazon S3 or Google Cloud Storage). The database stores only the **metadata and the file's URL/key**, and the CDN serves the files to users.

## 5. Architecture diagram

```
                         +-----------+
                         |  Clients  |
                         | (mobile/  |
                         |   web)    |
                         +-----+-----+
                               |
        +----------------------+-----------------------+
        | images (GET)                                 | API calls
        v                                              v
+----------------+                           +------------------+
|      CDN       |                           |  Load Balancer   |
| (caches photos |                           +--------+---------+
|  near users)   |                                    |
+-------+--------+                  +-----------------+-----------------+
        | cache miss                |                 |                 |
        v                           v                 v                 v
+----------------+            +-----------+     +-----------+     +-----------+
| Object Storage |            | App Server|     | App Server|     | App Server|
| (originals +   |            |     1     |     |     2     |     |     N     |
|  thumbnails)   |            +-----+-----+     +-----+-----+     +-----+-----+
+-------^--------+                  |                 |                 |
        |                           +--------+--------+-----------------+
        |                                    |
        |            +-----------------------+------------------------+
        |            |                       |                        |
        |            v                       v                        v
        |     +-------------+        +---------------+        +---------------+
        |     |    Cache    |        |   Database    |        |     Queue     |
        |     |  (Redis)    |        |   (primary)   |        | (thumbnail    |
        |     | feeds, meta |        +-------+-------+        |    jobs)      |
        |     +-------------+                |                +-------+-------+
        |                                    | replication            |
        |                                    v                        v
        |                            +---------------+        +---------------+
        |                            | Read Replica  |        |    Worker     |
        |                            | (feed reads)  |        | (makes 50 KB  |
        |                            +---------------+        |  thumbnails)  |
        |                                                     +-------+-------+
        |                                                             |
        +-------------------------------------------------------------+
                          worker reads original, writes thumbnail
```

Reads: client -> CDN (images) / load balancer -> app server -> cache -> read replica.
Writes: client -> load balancer -> app server -> object storage + primary database -> queue -> worker.

## 6. Components, one sentence each

- **CDN:** Caches photos and thumbnails at locations close to users so images load fast and most image traffic never reaches our servers or storage.
- **Load balancer:** Spreads incoming requests across many app servers so no single server is overloaded and a failed server is skipped.
- **App servers:** Run the application logic (login, upload, feed building) and are stateless, so we can add more of them to handle more traffic.
- **Cache (Redis):** Keeps frequently read data such as feeds and photo metadata in memory so repeated reads avoid slow database queries.
- **Database (primary):** Stores structured metadata (users, follows, photo records) reliably and handles all writes.
- **Read replica:** Holds a copy of the primary and serves feed reads, taking the heavy read load off the primary database.
- **Object storage:** Stores the huge volume of photo files cheaply, durably and in a way that scales almost without limit.
- **Queue:** Holds thumbnail jobs so the upload request can finish immediately instead of waiting for image processing, and absorbs traffic spikes.
- **Worker:** Takes jobs from the queue, creates the 50 KB thumbnail from the original and saves it, so image processing is done in the background.

## 7. Upload flow, step by step

1. The user picks a photo in the app and taps upload; the client sends the request to SnapShare.
2. The request reaches the **load balancer**, which forwards it to a healthy **app server**.
3. The app server authenticates the user and validates the file (type and size).
4. The app server saves the original 2 MB photo to **object storage** under a unique key (for example `photos/{user_id}/{photo_id}.jpg`).
5. The app server writes a row to the **primary database** with the photo id, user id, caption, timestamp, object storage key and status `processing`.
6. The app server puts a "create thumbnail" job (containing the photo id and key) on the **queue**.
7. The app server immediately replies "upload successful" to the user, so the user does not wait for thumbnail creation.
8. A **worker** picks up the job from the queue, downloads the original from object storage, and resizes it to a 50 KB thumbnail.
9. The worker saves the thumbnail to object storage and updates the database row with the thumbnail key and status `ready`.
10. The relevant cache entries (for example followers' feeds) are invalidated or updated, so the new photo appears in followers' feeds.
11. When followers load their feed, the thumbnail and photo are served through the **CDN**, which fetches them from object storage on the first request and caches them for later ones.

If the worker fails, the job stays in the queue (or goes to a retry/dead-letter queue) and is tried again, so no thumbnail is lost.

## 8. Trade-offs

1. **Caching: speed vs freshness.** The cache makes feeds fast and protects the database, but cached feeds can be a few seconds or minutes out of date, and we must handle invalidation. We accept slightly stale feeds because the app is read-heavy and instant freshness is not critical.
2. **Read replica: read scaling vs replication lag.** A replica spreads read load, but it can lag behind the primary, so a user may not see their own just-uploaded photo immediately. We can mitigate this by reading a user's own recent uploads from the primary or cache. It also adds cost and operational complexity.
3. **Queue and worker: fast uploads vs eventual consistency.** Thumbnails are made asynchronously, so uploads feel instant, but for a short time the thumbnail does not exist yet (we show a placeholder). We also add more moving parts to run and monitor.
4. **Object storage and CDN: cost and scale vs complexity.** Storing photos outside the database is cheap and scalable, but the database and storage can become inconsistent (a row without a file, or a file without a row), and CDN caches may serve a deleted photo until it expires. At about 748 TB per year, we could also trade storage cost for quality by moving old photos to cheaper cold storage or compressing them, at the cost of slower access to old photos.
5. **Thumbnails: extra storage vs speed.** Storing a 50 KB thumbnail adds about 18 TB per year, but it saves large bandwidth and makes feeds load far faster than sending 2 MB originals.
