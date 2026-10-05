# Capacity Gym — Get a feel for scale

> 50 sizing scenarios. Estimate nine numbers yourself, then open the reference answers (with the working) to check.

## How to estimate

| Estimate | Formula |
|---|---|
| Average QPS | daily users × requests per user ÷ 86,400 |
| Peak QPS | average × peak factor |
| Storage per day | daily requests × bytes per request ÷ 10⁹ (GB) |
| Retained storage | storage per day × retention days ÷ 1,000 (TB) |
| Peak bandwidth | peak QPS × bytes per request × 8 ÷ 10⁹ (Gbps) |
| Cache size | storage per day × cache % |
| Cache-miss origin QPS | peak QPS × (100 − cache %) |
| Partitions (lower bound) | ⌈peak QPS ÷ per-partition QPS⌉ |
| Servers (lower bound) | ⌈peak QPS ÷ per-server QPS⌉ |

Decimal units, no replication or compression. Within 25% is on target; within 2× is ballpark.

## Contents

- **Foundational** (5): [URL shortener workload](#url-shortener-workload) · [Distributed rate limiter workload](#distributed-rate-limiter-workload) · [Distributed cache workload](#distributed-cache-workload) · [Unique ID service workload](#unique-id-service-workload) · [Durable job scheduler workload](#durable-job-scheduler-workload)
- **Social** (5): [Social news feed workload](#social-news-feed-workload) · [Photo sharing workload](#photo-sharing-workload) · [Follow graph workload](#follow-graph-workload) · [Threaded comments workload](#threaded-comments-workload) · [Ephemeral stories workload](#ephemeral-stories-workload)
- **Messaging** (5): [Direct and group chat workload](#direct-and-group-chat-workload) · [Notification platform workload](#notification-platform-workload) · [Email service workload](#email-service-workload) · [Webhook delivery workload](#webhook-delivery-workload) · [Publish-subscribe event bus workload](#publish-subscribe-event-bus-workload)
- **Media** (5): [YouTube-style video platform workload](#youtube-style-video-platform-workload) · [Live streaming platform workload](#live-streaming-platform-workload) · [Music streaming workload](#music-streaming-workload) · [Video conferencing workload](#video-conferencing-workload) · [Podcast platform workload](#podcast-platform-workload)
- **Storage** (5): [Cloud file synchronization workload](#cloud-file-synchronization-workload) · [Distributed object storage workload](#distributed-object-storage-workload) · [Backup and point-in-time restore workload](#backup-and-point-in-time-restore-workload) · [Content-addressed blob store workload](#content-addressed-blob-store-workload) · [Distributed filesystem workload](#distributed-filesystem-workload)
- **Commerce** (5): [Commerce checkout workload](#commerce-checkout-workload) · [Inventory reservation workload](#inventory-reservation-workload) · [Payment ledger workload](#payment-ledger-workload) · [Ticket booking workload](#ticket-booking-workload) · [Marketplace orders workload](#marketplace-orders-workload)
- **Real Time** (5): [Ride dispatch workload](#ride-dispatch-workload) · [Collaborative document editor workload](#collaborative-document-editor-workload) · [Presence and typing indicators workload](#presence-and-typing-indicators-workload) · [Game leaderboard workload](#game-leaderboard-workload) · [Fleet tracking workload](#fleet-tracking-workload)
- **Search** (5): [Web search engine workload](#web-search-engine-workload) · [Search autocomplete workload](#search-autocomplete-workload) · [Product search workload](#product-search-workload) · [Log ingestion and search workload](#log-ingestion-and-search-workload) · [Vector retrieval service workload](#vector-retrieval-service-workload)
- **Infrastructure** (5): [Global load balancer workload](#global-load-balancer-workload) · [Authoritative DNS workload](#authoritative-dns-workload) · [Metrics and alerting workload](#metrics-and-alerting-workload) · [Feature flag platform workload](#feature-flag-platform-workload) · [Service discovery and configuration workload](#service-discovery-and-configuration-workload)
- **Advanced** (5): [Multi-region key-value store workload](#multi-region-key-value-store-workload) · [Durable workflow engine workload](#durable-workflow-engine-workload) · [Ad auction and budget pacing workload](#ad-auction-and-budget-pacing-workload) · [Recommendation serving workload](#recommendation-serving-workload) · [Streaming fraud detection workload](#streaming-fraud-detection-workload)

## Foundational

### URL shortener workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: One alias suddenly receives 200,000 redirects per second.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 20 |
| Bytes per request | 500 B |
| Peak factor | 8× |
| Retention | 365 days |
| Cache hit target | 95% |
| Per-partition budget | 3,000 QPS |
| Per-server budget | 5,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 2,315 req/s | 10,000,000 × 20 ÷ 86,400 |
| Peak QPS | 18,519 req/s | average × 8 |
| Storage per day | 100 GB | 200,000,000 requests × 500 B ÷ 10⁹ |
| Retained storage | 36.5 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 0.07 Gbps | peak × 500 B × 8 ÷ 10⁹ |
| Cache size | 95 GB | per day × 95% |
| Cache-miss origin QPS | 926 req/s | peak × 5% |
| Partitions | 7 | ⌈peak ÷ 3,000⌉ |
| Servers | 4 | ⌈peak ÷ 5,000⌉ |

</details>

---

### Distributed rate limiter workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A single tenant sends half of all requests to one token bucket.

| Given | Value |
|---|---|
| Daily active users | 5,000,000 |
| Requests per user per day | 20 |
| Bytes per request | 180 B |
| Peak factor | 10× |
| Retention | 7 days |
| Cache hit target | 60% |
| Per-partition budget | 20,000 QPS |
| Per-server budget | 12,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 1,157 req/s | 5,000,000 × 20 ÷ 86,400 |
| Peak QPS | 11,574 req/s | average × 10 |
| Storage per day | 18 GB | 100,000,000 requests × 180 B ÷ 10⁹ |
| Retained storage | 0.13 TB | per day × 7 days ÷ 1,000 |
| Peak bandwidth | 0.02 Gbps | peak × 180 B × 8 ÷ 10⁹ |
| Cache size | 10.8 GB | per day × 60% |
| Cache-miss origin QPS | 4,630 req/s | peak × 40% |
| Partitions | 1 | ⌈peak ÷ 20,000⌉ |
| Servers | 1 | ⌈peak ÷ 12,000⌉ |

</details>

---

### Distributed cache workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Adding four nodes remaps keys and doubles database reads.

| Given | Value |
|---|---|
| Daily active users | 20,000,000 |
| Requests per user per day | 100 |
| Bytes per request | 1,024 B |
| Peak factor | 4× |
| Retention | 1 days |
| Cache hit target | 90% |
| Per-partition budget | 50,000 QPS |
| Per-server budget | 18,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 23,148 req/s | 20,000,000 × 100 ÷ 86,400 |
| Peak QPS | 92,593 req/s | average × 4 |
| Storage per day | 2,048 GB | 2,000,000,000 requests × 1,024 B ÷ 10⁹ |
| Retained storage | 2.05 TB | per day × 1 days ÷ 1,000 |
| Peak bandwidth | 0.76 Gbps | peak × 1,024 B × 8 ÷ 10⁹ |
| Cache size | 1,843 GB | per day × 90% |
| Cache-miss origin QPS | 9,259 req/s | peak × 10% |
| Partitions | 2 | ⌈peak ÷ 50,000⌉ |
| Servers | 6 | ⌈peak ÷ 18,000⌉ |

</details>

---

### Unique ID service workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. Each request is one 16-byte ID result in this exercise; compare this with a packed 64-bit identifier and include protocol overhead separately. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A worker needs more IDs in one clock tick than its sequence bits allow.

| Given | Value |
|---|---|
| Daily active users | 5,000,000 |
| Requests per user per day | 100 |
| Bytes per request | 16 B |
| Peak factor | 6× |
| Retention | 365 days |
| Cache hit target | 0% |
| Per-partition budget | 50,000 QPS |
| Per-server budget | 100,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 5,787 req/s | 5,000,000 × 100 ÷ 86,400 |
| Peak QPS | 34,722 req/s | average × 6 |
| Storage per day | 8 GB | 500,000,000 requests × 16 B ÷ 10⁹ |
| Retained storage | 2.92 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 0 Gbps | peak × 16 B × 8 ÷ 10⁹ |
| Cache size | 0 GB | per day × 0% |
| Cache-miss origin QPS | 34,722 req/s | peak × 100% |
| Partitions | 1 | ⌈peak ÷ 50,000⌉ |
| Servers | 1 | ⌈peak ÷ 100,000⌉ |

</details>

---

### Durable job scheduler workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Three million reports become due at midnight for the same timezone.

| Given | Value |
|---|---|
| Daily active users | 5,000,000 |
| Requests per user per day | 10 |
| Bytes per request | 2,048 B |
| Peak factor | 20× |
| Retention | 30 days |
| Cache hit target | 0% |
| Per-partition budget | 1,500 QPS |
| Per-server budget | 2,500 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 579 req/s | 5,000,000 × 10 ÷ 86,400 |
| Peak QPS | 11,574 req/s | average × 20 |
| Storage per day | 102 GB | 50,000,000 requests × 2,048 B ÷ 10⁹ |
| Retained storage | 3.07 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 0.19 Gbps | peak × 2,048 B × 8 ÷ 10⁹ |
| Cache size | 0 GB | per day × 0% |
| Cache-miss origin QPS | 11,574 req/s | peak × 100% |
| Partitions | 8 | ⌈peak ÷ 1,500⌉ |
| Servers | 5 | ⌈peak ÷ 2,500⌉ |

</details>

---

## Social

### Social news feed workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: An account with 80M followers posts during peak traffic.

| Given | Value |
|---|---|
| Daily active users | 50,000,000 |
| Requests per user per day | 20 |
| Bytes per request | 12,000 B |
| Peak factor | 5× |
| Retention | 30 days |
| Cache hit target | 80% |
| Per-partition budget | 4,000 QPS |
| Per-server budget | 1,800 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 11,574 req/s | 50,000,000 × 20 ÷ 86,400 |
| Peak QPS | 57,870 req/s | average × 5 |
| Storage per day | 12,000 GB | 1,000,000,000 requests × 12,000 B ÷ 10⁹ |
| Retained storage | 360 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 5.56 Gbps | peak × 12,000 B × 8 ÷ 10⁹ |
| Cache size | 9,600 GB | per day × 80% |
| Cache-miss origin QPS | 11,574 req/s | peak × 20% |
| Partitions | 15 | ⌈peak ÷ 4,000⌉ |
| Servers | 33 | ⌈peak ÷ 1,800⌉ |

</details>

---

### Photo sharing workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A layout release asks for a previously uncached transform for every photo.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 8 |
| Bytes per request | 400,000 B |
| Peak factor | 8× |
| Retention | 365 days |
| Cache hit target | 96% |
| Per-partition budget | 2,500 QPS |
| Per-server budget | 3,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 926 req/s | 10,000,000 × 8 ÷ 86,400 |
| Peak QPS | 7,407 req/s | average × 8 |
| Storage per day | 32,000 GB | 80,000,000 requests × 400,000 B ÷ 10⁹ |
| Retained storage | 11,680 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 23.7 Gbps | peak × 400,000 B × 8 ÷ 10⁹ |
| Cache size | 30,720 GB | per day × 96% |
| Cache-miss origin QPS | 296 req/s | peak × 4% |
| Partitions | 3 | ⌈peak ÷ 2,500⌉ |
| Servers | 3 | ⌈peak ÷ 3,000⌉ |

</details>

---

### Follow graph workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: One account has 60M followers and its inbound row becomes unmanageable.

| Given | Value |
|---|---|
| Daily active users | 20,000,000 |
| Requests per user per day | 15 |
| Bytes per request | 1,400 B |
| Peak factor | 6× |
| Retention | 365 days |
| Cache hit target | 75% |
| Per-partition budget | 3,500 QPS |
| Per-server budget | 4,500 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 3,472 req/s | 20,000,000 × 15 ÷ 86,400 |
| Peak QPS | 20,833 req/s | average × 6 |
| Storage per day | 420 GB | 300,000,000 requests × 1,400 B ÷ 10⁹ |
| Retained storage | 153 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 0.23 Gbps | peak × 1,400 B × 8 ÷ 10⁹ |
| Cache size | 315 GB | per day × 75% |
| Cache-miss origin QPS | 5,208 req/s | peak × 25% |
| Partitions | 6 | ⌈peak ÷ 3,500⌉ |
| Servers | 5 | ⌈peak ÷ 4,500⌉ |

</details>

---

### Threaded comments workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A match-final thread receives 20,000 comments per second.

| Given | Value |
|---|---|
| Daily active users | 15,000,000 |
| Requests per user per day | 12 |
| Bytes per request | 6,000 B |
| Peak factor | 12× |
| Retention | 180 days |
| Cache hit target | 85% |
| Per-partition budget | 2,000 QPS |
| Per-server budget | 2,200 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 2,083 req/s | 15,000,000 × 12 ÷ 86,400 |
| Peak QPS | 25,000 req/s | average × 12 |
| Storage per day | 1,080 GB | 180,000,000 requests × 6,000 B ÷ 10⁹ |
| Retained storage | 194 TB | per day × 180 days ÷ 1,000 |
| Peak bandwidth | 1.2 Gbps | peak × 6,000 B × 8 ÷ 10⁹ |
| Cache size | 918 GB | per day × 85% |
| Cache-miss origin QPS | 3,750 req/s | peak × 15% |
| Partitions | 13 | ⌈peak ÷ 2,000⌉ |
| Servers | 12 | ⌈peak ÷ 2,200⌉ |

</details>

---

### Ephemeral stories workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A public story accumulates 30M unique viewers in an hour.

| Given | Value |
|---|---|
| Daily active users | 30,000,000 |
| Requests per user per day | 5 |
| Bytes per request | 250,000 B |
| Peak factor | 8× |
| Retention | 1 days |
| Cache hit target | 95% |
| Per-partition budget | 3,000 QPS |
| Per-server budget | 3,500 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 1,736 req/s | 30,000,000 × 5 ÷ 86,400 |
| Peak QPS | 13,889 req/s | average × 8 |
| Storage per day | 37,500 GB | 150,000,000 requests × 250,000 B ÷ 10⁹ |
| Retained storage | 37.5 TB | per day × 1 days ÷ 1,000 |
| Peak bandwidth | 27.78 Gbps | peak × 250,000 B × 8 ÷ 10⁹ |
| Cache size | 35,625 GB | per day × 95% |
| Cache-miss origin QPS | 694 req/s | peak × 5% |
| Partitions | 5 | ⌈peak ÷ 3,000⌉ |
| Servers | 4 | ⌈peak ÷ 3,500⌉ |

</details>

---

## Messaging

### Direct and group chat workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A 1,000-member trading room generates 10,000 messages per second.

| Given | Value |
|---|---|
| Daily active users | 50,000,000 |
| Requests per user per day | 40 |
| Bytes per request | 1,200 B |
| Peak factor | 5× |
| Retention | 365 days |
| Cache hit target | 10% |
| Per-partition budget | 6,000 QPS |
| Per-server budget | 7,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 23,148 req/s | 50,000,000 × 40 ÷ 86,400 |
| Peak QPS | 115,741 req/s | average × 5 |
| Storage per day | 2,400 GB | 2,000,000,000 requests × 1,200 B ÷ 10⁹ |
| Retained storage | 876 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 1.11 Gbps | peak × 1,200 B × 8 ÷ 10⁹ |
| Cache size | 240 GB | per day × 10% |
| Cache-miss origin QPS | 104,167 req/s | peak × 90% |
| Partitions | 20 | ⌈peak ÷ 6,000⌉ |
| Servers | 17 | ⌈peak ÷ 7,000⌉ |

</details>

---

### Notification platform workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A campaign backlog delays password reset messages by twenty minutes.

| Given | Value |
|---|---|
| Daily active users | 50,000,000 |
| Requests per user per day | 10 |
| Bytes per request | 1,800 B |
| Peak factor | 15× |
| Retention | 90 days |
| Cache hit target | 5% |
| Per-partition budget | 2,500 QPS |
| Per-server budget | 5,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 5,787 req/s | 50,000,000 × 10 ÷ 86,400 |
| Peak QPS | 86,806 req/s | average × 15 |
| Storage per day | 900 GB | 500,000,000 requests × 1,800 B ÷ 10⁹ |
| Retained storage | 81 TB | per day × 90 days ÷ 1,000 |
| Peak bandwidth | 1.25 Gbps | peak × 1,800 B × 8 ÷ 10⁹ |
| Cache size | 45 GB | per day × 5% |
| Cache-miss origin QPS | 82,465 req/s | peak × 95% |
| Partitions | 35 | ⌈peak ÷ 2,500⌉ |
| Servers | 18 | ⌈peak ÷ 5,000⌉ |

</details>

---

### Email service workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A few senders upload 100MB attachments and exhaust spool disk.

| Given | Value |
|---|---|
| Daily active users | 1,000,000 |
| Requests per user per day | 100 |
| Bytes per request | 80,000 B |
| Peak factor | 5× |
| Retention | 365 days |
| Cache hit target | 30% |
| Per-partition budget | 2,000 QPS |
| Per-server budget | 1,200 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 1,157 req/s | 1,000,000 × 100 ÷ 86,400 |
| Peak QPS | 5,787 req/s | average × 5 |
| Storage per day | 8,000 GB | 100,000,000 requests × 80,000 B ÷ 10⁹ |
| Retained storage | 2,920 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 3.7 Gbps | peak × 80,000 B × 8 ÷ 10⁹ |
| Cache size | 2,400 GB | per day × 30% |
| Cache-miss origin QPS | 4,051 req/s | peak × 70% |
| Partitions | 3 | ⌈peak ÷ 2,000⌉ |
| Servers | 5 | ⌈peak ÷ 1,200⌉ |

</details>

---

### Webhook delivery workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: An enterprise endpoint takes 25 seconds per request and blocks unrelated tenants.

| Given | Value |
|---|---|
| Daily active users | 1,000,000 |
| Requests per user per day | 200 |
| Bytes per request | 4,096 B |
| Peak factor | 12× |
| Retention | 30 days |
| Cache hit target | 0% |
| Per-partition budget | 2,500 QPS |
| Per-server budget | 4,500 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 2,315 req/s | 1,000,000 × 200 ÷ 86,400 |
| Peak QPS | 27,778 req/s | average × 12 |
| Storage per day | 819 GB | 200,000,000 requests × 4,096 B ÷ 10⁹ |
| Retained storage | 24.58 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 0.91 Gbps | peak × 4,096 B × 8 ÷ 10⁹ |
| Cache size | 0 GB | per day × 0% |
| Cache-miss origin QPS | 27,778 req/s | peak × 100% |
| Partitions | 12 | ⌈peak ÷ 2,500⌉ |
| Servers | 7 | ⌈peak ÷ 4,500⌉ |

</details>

---

### Publish-subscribe event bus workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Twenty consumer groups each read the full billion-event stream.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 100 |
| Bytes per request | 1,024 B |
| Peak factor | 7× |
| Retention | 14 days |
| Cache hit target | 0% |
| Per-partition budget | 20,000 QPS |
| Per-server budget | 15,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 11,574 req/s | 10,000,000 × 100 ÷ 86,400 |
| Peak QPS | 81,019 req/s | average × 7 |
| Storage per day | 1,024 GB | 1,000,000,000 requests × 1,024 B ÷ 10⁹ |
| Retained storage | 14.34 TB | per day × 14 days ÷ 1,000 |
| Peak bandwidth | 0.66 Gbps | peak × 1,024 B × 8 ÷ 10⁹ |
| Cache size | 0 GB | per day × 0% |
| Cache-miss origin QPS | 81,019 req/s | peak × 100% |
| Partitions | 5 | ⌈peak ÷ 20,000⌉ |
| Servers | 6 | ⌈peak ÷ 15,000⌉ |

</details>

---

## Media

### YouTube-style video platform workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. Each request is one 6-second media segment at an assumed 3Mb/s, or 2.25MB; 300 segments correspond to 30 minutes of viewing. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A newly published video draws 2M concurrent viewers while CDN segment caches are empty.

| Given | Value |
|---|---|
| Daily active users | 20,000,000 |
| Requests per user per day | 300 |
| Bytes per request | 2,250,000 B |
| Peak factor | 4× |
| Retention | 30 days |
| Cache hit target | 97% |
| Per-partition budget | 3,000 QPS |
| Per-server budget | 5,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 69,444 req/s | 20,000,000 × 300 ÷ 86,400 |
| Peak QPS | 277,778 req/s | average × 4 |
| Storage per day | 13,500,000 GB | 6,000,000,000 requests × 2,250,000 B ÷ 10⁹ |
| Retained storage | 405,000 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 5,000 Gbps | peak × 2,250,000 B × 8 ÷ 10⁹ |
| Cache size | 13,095,000 GB | per day × 97% |
| Cache-miss origin QPS | 8,333 req/s | peak × 3% |
| Partitions | 93 | ⌈peak ÷ 3,000⌉ |
| Servers | 56 | ⌈peak ÷ 5,000⌉ |

</details>

---

### Live streaming platform workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. Each request is an illustrative media segment averaging 0.75MB; 240 segments per viewer are modeled independently of manifest requests. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A match audience grows from 200K to 5M viewers in ninety seconds.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 240 |
| Bytes per request | 750,000 B |
| Peak factor | 15× |
| Retention | 7 days |
| Cache hit target | 98% |
| Per-partition budget | 3,000 QPS |
| Per-server budget | 5,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 27,778 req/s | 10,000,000 × 240 ÷ 86,400 |
| Peak QPS | 416,667 req/s | average × 15 |
| Storage per day | 1,800,000 GB | 2,400,000,000 requests × 750,000 B ÷ 10⁹ |
| Retained storage | 12,600 TB | per day × 7 days ÷ 1,000 |
| Peak bandwidth | 2,500 Gbps | peak × 750,000 B × 8 ÷ 10⁹ |
| Cache size | 1,764,000 GB | per day × 98% |
| Cache-miss origin QPS | 8,333 req/s | peak × 2% |
| Partitions | 139 | ⌈peak ÷ 3,000⌉ |
| Servers | 84 | ⌈peak ÷ 5,000⌉ |

</details>

---

### Music streaming workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Ten million listeners start the same album immediately at release.

| Given | Value |
|---|---|
| Daily active users | 20,000,000 |
| Requests per user per day | 40 |
| Bytes per request | 4,000,000 B |
| Peak factor | 5× |
| Retention | 365 days |
| Cache hit target | 97% |
| Per-partition budget | 3,000 QPS |
| Per-server budget | 4,500 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 9,259 req/s | 20,000,000 × 40 ÷ 86,400 |
| Peak QPS | 46,296 req/s | average × 5 |
| Storage per day | 3,200,000 GB | 800,000,000 requests × 4,000,000 B ÷ 10⁹ |
| Retained storage | 1,168,000 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 1,481 Gbps | peak × 4,000,000 B × 8 ÷ 10⁹ |
| Cache size | 3,104,000 GB | per day × 97% |
| Cache-miss origin QPS | 1,389 req/s | peak × 3% |
| Partitions | 16 | ⌈peak ÷ 3,000⌉ |
| Servers | 11 | ⌈peak ÷ 4,500⌉ |

</details>

---

### Video conferencing workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. This dataset models 15KB signaling or telemetry messages, not media bandwidth. Separately estimate media using participants, subscribed tracks, bitrate, and session duration. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A 2,000-person meeting sends every camera to every participant.

| Given | Value |
|---|---|
| Daily active users | 3,000,000 |
| Requests per user per day | 100 |
| Bytes per request | 15,000 B |
| Peak factor | 10× |
| Retention | 1 days |
| Cache hit target | 0% |
| Per-partition budget | 12,000 QPS |
| Per-server budget | 8,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 3,472 req/s | 3,000,000 × 100 ÷ 86,400 |
| Peak QPS | 34,722 req/s | average × 10 |
| Storage per day | 4,500 GB | 300,000,000 requests × 15,000 B ÷ 10⁹ |
| Retained storage | 4.5 TB | per day × 1 days ÷ 1,000 |
| Peak bandwidth | 4.17 Gbps | peak × 15,000 B × 8 ÷ 10⁹ |
| Cache size | 0 GB | per day × 0% |
| Cache-miss origin QPS | 34,722 req/s | peak × 100% |
| Partitions | 3 | ⌈peak ÷ 12,000⌉ |
| Servers | 5 | ⌈peak ÷ 8,000⌉ |

</details>

---

### Podcast platform workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Two million feeds are polled every minute although most publish weekly.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 4 |
| Bytes per request | 12,000,000 B |
| Peak factor | 6× |
| Retention | 365 days |
| Cache hit target | 95% |
| Per-partition budget | 2,500 QPS |
| Per-server budget | 4,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 463 req/s | 10,000,000 × 4 ÷ 86,400 |
| Peak QPS | 2,778 req/s | average × 6 |
| Storage per day | 480,000 GB | 40,000,000 requests × 12,000,000 B ÷ 10⁹ |
| Retained storage | 175,200 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 267 Gbps | peak × 12,000,000 B × 8 ÷ 10⁹ |
| Cache size | 456,000 GB | per day × 95% |
| Cache-miss origin QPS | 139 req/s | peak × 5% |
| Partitions | 2 | ⌈peak ÷ 2,500⌉ |
| Servers | 1 | ⌈peak ÷ 4,000⌉ |

</details>

---

## Storage

### Cloud file synchronization workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Initial synchronization performs one request per file and takes days.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 10 |
| Bytes per request | 2,000,000 B |
| Peak factor | 8× |
| Retention | 90 days |
| Cache hit target | 50% |
| Per-partition budget | 2,200 QPS |
| Per-server budget | 1,800 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 1,157 req/s | 10,000,000 × 10 ÷ 86,400 |
| Peak QPS | 9,259 req/s | average × 8 |
| Storage per day | 200,000 GB | 100,000,000 requests × 2,000,000 B ÷ 10⁹ |
| Retained storage | 18,000 TB | per day × 90 days ÷ 1,000 |
| Peak bandwidth | 148 Gbps | peak × 2,000,000 B × 8 ÷ 10⁹ |
| Cache size | 100,000 GB | per day × 50% |
| Cache-miss origin QPS | 4,630 req/s | peak × 50% |
| Partitions | 5 | ⌈peak ÷ 2,200⌉ |
| Servers | 6 | ⌈peak ÷ 1,800⌉ |

</details>

---

### Distributed object storage workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Losing a rack requires reconstructing petabytes while clients continue reading.

| Given | Value |
|---|---|
| Daily active users | 2,000,000 |
| Requests per user per day | 25 |
| Bytes per request | 5,000,000 B |
| Peak factor | 4× |
| Retention | 365 days |
| Cache hit target | 40% |
| Per-partition budget | 3,000 QPS |
| Per-server budget | 3,500 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 579 req/s | 2,000,000 × 25 ÷ 86,400 |
| Peak QPS | 2,315 req/s | average × 4 |
| Storage per day | 250,000 GB | 50,000,000 requests × 5,000,000 B ÷ 10⁹ |
| Retained storage | 91,250 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 92.59 Gbps | peak × 5,000,000 B × 8 ÷ 10⁹ |
| Cache size | 100,000 GB | per day × 40% |
| Cache-miss origin QPS | 1,389 req/s | peak × 60% |
| Partitions | 1 | ⌈peak ÷ 3,000⌉ |
| Servers | 1 | ⌈peak ÷ 3,500⌉ |

</details>

---

### Backup and point-in-time restore workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. Each request is a 1MB changed-data chunk sent to backup storage; restore throughput requires a separate scenario using snapshot size and replay volume. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A 500TB snapshot must be restored over a 10Gb/s link.

| Given | Value |
|---|---|
| Daily active users | 100,000 |
| Requests per user per day | 20 |
| Bytes per request | 1,000,000 B |
| Peak factor | 3× |
| Retention | 180 days |
| Cache hit target | 0% |
| Per-partition budget | 1,500 QPS |
| Per-server budget | 2,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 23.15 req/s | 100,000 × 20 ÷ 86,400 |
| Peak QPS | 69.44 req/s | average × 3 |
| Storage per day | 2,000 GB | 2,000,000 requests × 1,000,000 B ÷ 10⁹ |
| Retained storage | 360 TB | per day × 180 days ÷ 1,000 |
| Peak bandwidth | 0.56 Gbps | peak × 1,000,000 B × 8 ÷ 10⁹ |
| Cache size | 0 GB | per day × 0% |
| Cache-miss origin QPS | 69.44 req/s | peak × 100% |
| Partitions | 1 | ⌈peak ÷ 1,500⌉ |
| Servers | 1 | ⌈peak ÷ 2,000⌉ |

</details>

---

### Content-addressed blob store workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A common dependency receives millions of reference additions per minute.

| Given | Value |
|---|---|
| Daily active users | 5,000,000 |
| Requests per user per day | 20 |
| Bytes per request | 300,000 B |
| Peak factor | 7× |
| Retention | 365 days |
| Cache hit target | 70% |
| Per-partition budget | 3,000 QPS |
| Per-server budget | 3,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 1,157 req/s | 5,000,000 × 20 ÷ 86,400 |
| Peak QPS | 8,102 req/s | average × 7 |
| Storage per day | 30,000 GB | 100,000,000 requests × 300,000 B ÷ 10⁹ |
| Retained storage | 10,950 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 19.44 Gbps | peak × 300,000 B × 8 ÷ 10⁹ |
| Cache size | 21,000 GB | per day × 70% |
| Cache-miss origin QPS | 2,431 req/s | peak × 30% |
| Partitions | 3 | ⌈peak ÷ 3,000⌉ |
| Servers | 3 | ⌈peak ÷ 3,000⌉ |

</details>

---

### Distributed filesystem workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A billion tiny files fit on disks but exhaust metadata servers.

| Given | Value |
|---|---|
| Daily active users | 1,000,000 |
| Requests per user per day | 80 |
| Bytes per request | 64,000 B |
| Peak factor | 6× |
| Retention | 365 days |
| Cache hit target | 65% |
| Per-partition budget | 10,000 QPS |
| Per-server budget | 8,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 926 req/s | 1,000,000 × 80 ÷ 86,400 |
| Peak QPS | 5,556 req/s | average × 6 |
| Storage per day | 5,120 GB | 80,000,000 requests × 64,000 B ÷ 10⁹ |
| Retained storage | 1,869 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 2.84 Gbps | peak × 64,000 B × 8 ÷ 10⁹ |
| Cache size | 3,328 GB | per day × 65% |
| Cache-miss origin QPS | 1,944 req/s | peak × 35% |
| Partitions | 1 | ⌈peak ÷ 10,000⌉ |
| Servers | 1 | ⌈peak ÷ 8,000⌉ |

</details>

---

## Commerce

### Commerce checkout workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Most checkouts target one scarce SKU while the rest of the store is healthy.

| Given | Value |
|---|---|
| Daily active users | 5,000,000 |
| Requests per user per day | 5 |
| Bytes per request | 3,500 B |
| Peak factor | 15× |
| Retention | 365 days |
| Cache hit target | 10% |
| Per-partition budget | 1,200 QPS |
| Per-server budget | 2,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 289 req/s | 5,000,000 × 5 ÷ 86,400 |
| Peak QPS | 4,340 req/s | average × 15 |
| Storage per day | 87.5 GB | 25,000,000 requests × 3,500 B ÷ 10⁹ |
| Retained storage | 31.94 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 0.12 Gbps | peak × 3,500 B × 8 ÷ 10⁹ |
| Cache size | 8.75 GB | per day × 10% |
| Cache-miss origin QPS | 3,906 req/s | peak × 90% |
| Partitions | 4 | ⌈peak ÷ 1,200⌉ |
| Servers | 3 | ⌈peak ÷ 2,000⌉ |

</details>

---

### Inventory reservation workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Thousands of buyers contend for the last thousand units.

| Given | Value |
|---|---|
| Daily active users | 5,000,000 |
| Requests per user per day | 10 |
| Bytes per request | 700 B |
| Peak factor | 20× |
| Retention | 90 days |
| Cache hit target | 0% |
| Per-partition budget | 1,500 QPS |
| Per-server budget | 2,500 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 579 req/s | 5,000,000 × 10 ÷ 86,400 |
| Peak QPS | 11,574 req/s | average × 20 |
| Storage per day | 35 GB | 50,000,000 requests × 700 B ÷ 10⁹ |
| Retained storage | 3.15 TB | per day × 90 days ÷ 1,000 |
| Peak bandwidth | 0.06 Gbps | peak × 700 B × 8 ÷ 10⁹ |
| Cache size | 0 GB | per day × 0% |
| Cache-miss origin QPS | 11,574 req/s | peak × 100% |
| Partitions | 8 | ⌈peak ÷ 1,500⌉ |
| Servers | 5 | ⌈peak ÷ 2,500⌉ |

</details>

---

### Payment ledger workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Every merchant payment updates the same platform settlement balance.

| Given | Value |
|---|---|
| Daily active users | 2,000,000 |
| Requests per user per day | 5 |
| Bytes per request | 1,800 B |
| Peak factor | 8× |
| Retention | 2555 days |
| Cache hit target | 5% |
| Per-partition budget | 1,000 QPS |
| Per-server budget | 1,800 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 116 req/s | 2,000,000 × 5 ÷ 86,400 |
| Peak QPS | 926 req/s | average × 8 |
| Storage per day | 18 GB | 10,000,000 requests × 1,800 B ÷ 10⁹ |
| Retained storage | 45.99 TB | per day × 2555 days ÷ 1,000 |
| Peak bandwidth | 0.01 Gbps | peak × 1,800 B × 8 ÷ 10⁹ |
| Cache size | 0.9 GB | per day × 5% |
| Cache-miss origin QPS | 880 req/s | peak × 95% |
| Partitions | 1 | ⌈peak ÷ 1,000⌉ |
| Servers | 1 | ⌈peak ÷ 1,800⌉ |

</details>

---

### Ticket booking workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Three million waiting users poll the full seat map every second.

| Given | Value |
|---|---|
| Daily active users | 3,000,000 |
| Requests per user per day | 30 |
| Bytes per request | 6,000 B |
| Peak factor | 30× |
| Retention | 365 days |
| Cache hit target | 90% |
| Per-partition budget | 1,200 QPS |
| Per-server budget | 2,200 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 1,042 req/s | 3,000,000 × 30 ÷ 86,400 |
| Peak QPS | 31,250 req/s | average × 30 |
| Storage per day | 540 GB | 90,000,000 requests × 6,000 B ÷ 10⁹ |
| Retained storage | 197 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 1.5 Gbps | peak × 6,000 B × 8 ÷ 10⁹ |
| Cache size | 486 GB | per day × 90% |
| Cache-miss origin QPS | 3,125 req/s | peak × 10% |
| Partitions | 27 | ⌈peak ÷ 1,200⌉ |
| Servers | 15 | ⌈peak ÷ 2,200⌉ |

</details>

---

### Marketplace orders workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: One seller integration is unavailable while other items are ready to ship.

| Given | Value |
|---|---|
| Daily active users | 2,000,000 |
| Requests per user per day | 15 |
| Bytes per request | 5,000 B |
| Peak factor | 8× |
| Retention | 730 days |
| Cache hit target | 25% |
| Per-partition budget | 1,500 QPS |
| Per-server budget | 1,800 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 347 req/s | 2,000,000 × 15 ÷ 86,400 |
| Peak QPS | 2,778 req/s | average × 8 |
| Storage per day | 150 GB | 30,000,000 requests × 5,000 B ÷ 10⁹ |
| Retained storage | 110 TB | per day × 730 days ÷ 1,000 |
| Peak bandwidth | 0.11 Gbps | peak × 5,000 B × 8 ÷ 10⁹ |
| Cache size | 37.5 GB | per day × 25% |
| Cache-miss origin QPS | 2,083 req/s | peak × 75% |
| Partitions | 2 | ⌈peak ÷ 1,500⌉ |
| Servers | 2 | ⌈peak ÷ 1,800⌉ |

</details>

---

## Real Time

### Ride dispatch workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. The active unit is a driver device sending 17,280 updates over 24 hours, one update every 5 seconds; production should account for actual online duty cycles. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Thousands of riders and drivers concentrate in one geospatial bucket.

| Given | Value |
|---|---|
| Daily active users | 2,000,000 |
| Requests per user per day | 17,280 |
| Bytes per request | 120 B |
| Peak factor | 2× |
| Retention | 1 days |
| Cache hit target | 0% |
| Per-partition budget | 20,000 QPS |
| Per-server budget | 15,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 400,000 req/s | 2,000,000 × 17280 ÷ 86,400 |
| Peak QPS | 800,000 req/s | average × 2 |
| Storage per day | 4,147 GB | 34,560,000,000 requests × 120 B ÷ 10⁹ |
| Retained storage | 4.15 TB | per day × 1 days ÷ 1,000 |
| Peak bandwidth | 0.77 Gbps | peak × 120 B × 8 ÷ 10⁹ |
| Cache size | 0 GB | per day × 0% |
| Cache-miss origin QPS | 800,000 req/s | peak × 100% |
| Partitions | 40 | ⌈peak ÷ 20,000⌉ |
| Servers | 54 | ⌈peak ÷ 15,000⌉ |

</details>

---

### Collaborative document editor workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Opening a document replays five million edits and freezes the client.

| Given | Value |
|---|---|
| Daily active users | 1,000,000 |
| Requests per user per day | 18,000 |
| Bytes per request | 250 B |
| Peak factor | 3× |
| Retention | 30 days |
| Cache hit target | 20% |
| Per-partition budget | 10,000 QPS |
| Per-server budget | 12,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 208,333 req/s | 1,000,000 × 18000 ÷ 86,400 |
| Peak QPS | 625,000 req/s | average × 3 |
| Storage per day | 4,500 GB | 18,000,000,000 requests × 250 B ÷ 10⁹ |
| Retained storage | 135 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 1.25 Gbps | peak × 250 B × 8 ÷ 10⁹ |
| Cache size | 900 GB | per day × 20% |
| Cache-miss origin QPS | 500,000 req/s | peak × 80% |
| Partitions | 63 | ⌈peak ÷ 10,000⌉ |
| Servers | 53 | ⌈peak ÷ 12,000⌉ |

</details>

---

### Presence and typing indicators workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. The active unit is a connected device sending 2,880 heartbeats over 24 hours, one every 30 seconds; typing fanout is additional. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A regional network outage ends and 10M devices reconnect at once.

| Given | Value |
|---|---|
| Daily active users | 30,000,000 |
| Requests per user per day | 2,880 |
| Bytes per request | 80 B |
| Peak factor | 4× |
| Retention | 1 days |
| Cache hit target | 0% |
| Per-partition budget | 30,000 QPS |
| Per-server budget | 25,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 1,000,000 req/s | 30,000,000 × 2880 ÷ 86,400 |
| Peak QPS | 4,000,000 req/s | average × 4 |
| Storage per day | 6,912 GB | 86,400,000,000 requests × 80 B ÷ 10⁹ |
| Retained storage | 6.91 TB | per day × 1 days ÷ 1,000 |
| Peak bandwidth | 2.56 Gbps | peak × 80 B × 8 ÷ 10⁹ |
| Cache size | 0 GB | per day × 0% |
| Cache-miss origin QPS | 4,000,000 req/s | peak × 100% |
| Partitions | 134 | ⌈peak ÷ 30,000⌉ |
| Servers | 160 | ⌈peak ÷ 25,000⌉ |

</details>

---

### Game leaderboard workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Fifty million players and constant writes overload one ranking shard.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 20 |
| Bytes per request | 250 B |
| Peak factor | 12× |
| Retention | 90 days |
| Cache hit target | 70% |
| Per-partition budget | 12,000 QPS |
| Per-server budget | 15,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 2,315 req/s | 10,000,000 × 20 ÷ 86,400 |
| Peak QPS | 27,778 req/s | average × 12 |
| Storage per day | 50 GB | 200,000,000 requests × 250 B ÷ 10⁹ |
| Retained storage | 4.5 TB | per day × 90 days ÷ 1,000 |
| Peak bandwidth | 0.06 Gbps | peak × 250 B × 8 ÷ 10⁹ |
| Cache size | 35 GB | per day × 70% |
| Cache-miss origin QPS | 8,333 req/s | peak × 30% |
| Partitions | 3 | ⌈peak ÷ 12,000⌉ |
| Servers | 2 | ⌈peak ÷ 15,000⌉ |

</details>

---

### Fleet tracking workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. The active unit is a vehicle sending 4,320 updates over 12 moving hours, one update every 10 seconds. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A large customer opens a world map containing one million vehicles.

| Given | Value |
|---|---|
| Daily active users | 5,000,000 |
| Requests per user per day | 4,320 |
| Bytes per request | 160 B |
| Peak factor | 3× |
| Retention | 90 days |
| Cache hit target | 10% |
| Per-partition budget | 15,000 QPS |
| Per-server budget | 12,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 250,000 req/s | 5,000,000 × 4320 ÷ 86,400 |
| Peak QPS | 750,000 req/s | average × 3 |
| Storage per day | 3,456 GB | 21,600,000,000 requests × 160 B ÷ 10⁹ |
| Retained storage | 311 TB | per day × 90 days ÷ 1,000 |
| Peak bandwidth | 0.96 Gbps | peak × 160 B × 8 ÷ 10⁹ |
| Cache size | 346 GB | per day × 10% |
| Cache-miss origin QPS | 675,000 req/s | peak × 90% |
| Partitions | 50 | ⌈peak ÷ 15,000⌉ |
| Servers | 63 | ⌈peak ÷ 12,000⌉ |

</details>

---

## Search

### Web search engine workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A short query matches billions of postings and exhausts tail latency.

| Given | Value |
|---|---|
| Daily active users | 20,000,000 |
| Requests per user per day | 5 |
| Bytes per request | 18,000 B |
| Peak factor | 7× |
| Retention | 30 days |
| Cache hit target | 45% |
| Per-partition budget | 2,000 QPS |
| Per-server budget | 1,200 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 1,157 req/s | 20,000,000 × 5 ÷ 86,400 |
| Peak QPS | 8,102 req/s | average × 7 |
| Storage per day | 1,800 GB | 100,000,000 requests × 18,000 B ÷ 10⁹ |
| Retained storage | 54 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 1.17 Gbps | peak × 18,000 B × 8 ÷ 10⁹ |
| Cache size | 810 GB | per day × 45% |
| Cache-miss origin QPS | 4,456 req/s | peak × 55% |
| Partitions | 5 | ⌈peak ÷ 2,000⌉ |
| Servers | 7 | ⌈peak ÷ 1,200⌉ |

</details>

---

### Search autocomplete workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Fast typists create six requests for one intended search.

| Given | Value |
|---|---|
| Daily active users | 25,000,000 |
| Requests per user per day | 20 |
| Bytes per request | 1,800 B |
| Peak factor | 8× |
| Retention | 7 days |
| Cache hit target | 92% |
| Per-partition budget | 25,000 QPS |
| Per-server budget | 20,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 5,787 req/s | 25,000,000 × 20 ÷ 86,400 |
| Peak QPS | 46,296 req/s | average × 8 |
| Storage per day | 900 GB | 500,000,000 requests × 1,800 B ÷ 10⁹ |
| Retained storage | 6.3 TB | per day × 7 days ÷ 1,000 |
| Peak bandwidth | 0.67 Gbps | peak × 1,800 B × 8 ÷ 10⁹ |
| Cache size | 828 GB | per day × 92% |
| Cache-miss origin QPS | 3,704 req/s | peak × 8% |
| Partitions | 2 | ⌈peak ÷ 25,000⌉ |
| Servers | 3 | ⌈peak ÷ 20,000⌉ |

</details>

---

### Product search workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A request aggregates millions of distinct seller and attribute combinations.

| Given | Value |
|---|---|
| Daily active users | 20,000,000 |
| Requests per user per day | 4 |
| Bytes per request | 20,000 B |
| Peak factor | 10× |
| Retention | 30 days |
| Cache hit target | 60% |
| Per-partition budget | 1,800 QPS |
| Per-server budget | 1,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 926 req/s | 20,000,000 × 4 ÷ 86,400 |
| Peak QPS | 9,259 req/s | average × 10 |
| Storage per day | 1,600 GB | 80,000,000 requests × 20,000 B ÷ 10⁹ |
| Retained storage | 48 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 1.48 Gbps | peak × 20,000 B × 8 ÷ 10⁹ |
| Cache size | 960 GB | per day × 60% |
| Cache-miss origin QPS | 3,704 req/s | peak × 40% |
| Partitions | 6 | ⌈peak ÷ 1,800⌉ |
| Servers | 10 | ⌈peak ÷ 1,000⌉ |

</details>

---

### Log ingestion and search workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. The active unit represents an agent workload; each of its 100 daily requests is a 100KB log batch. This yields 100TB/day across 10M modeled agents. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A failing service emits the same stack trace millions of times per second.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 100 |
| Bytes per request | 100,000 B |
| Peak factor | 10× |
| Retention | 30 days |
| Cache hit target | 10% |
| Per-partition budget | 25,000 QPS |
| Per-server budget | 12,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 11,574 req/s | 10,000,000 × 100 ÷ 86,400 |
| Peak QPS | 115,741 req/s | average × 10 |
| Storage per day | 100,000 GB | 1,000,000,000 requests × 100,000 B ÷ 10⁹ |
| Retained storage | 3,000 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 92.59 Gbps | peak × 100,000 B × 8 ÷ 10⁹ |
| Cache size | 10,000 GB | per day × 10% |
| Cache-miss origin QPS | 104,167 req/s | peak × 90% |
| Partitions | 5 | ⌈peak ÷ 25,000⌉ |
| Servers | 10 | ⌈peak ÷ 12,000⌉ |

</details>

---

### Vector retrieval service workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Approximate search retrieves nearby vectors that are mostly outside the requested tenant or category.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 5 |
| Bytes per request | 9,000 B |
| Peak factor | 8× |
| Retention | 365 days |
| Cache hit target | 25% |
| Per-partition budget | 1,200 QPS |
| Per-server budget | 900 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 579 req/s | 10,000,000 × 5 ÷ 86,400 |
| Peak QPS | 4,630 req/s | average × 8 |
| Storage per day | 450 GB | 50,000,000 requests × 9,000 B ÷ 10⁹ |
| Retained storage | 164 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 0.33 Gbps | peak × 9,000 B × 8 ÷ 10⁹ |
| Cache size | 113 GB | per day × 25% |
| Cache-miss origin QPS | 3,472 req/s | peak × 75% |
| Partitions | 4 | ⌈peak ÷ 1,200⌉ |
| Servers | 6 | ⌈peak ÷ 900⌉ |

</details>

---

## Infrastructure

### Global load balancer workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A failed region’s entire traffic is redirected to a region with only 30% spare capacity.

| Given | Value |
|---|---|
| Daily active users | 100,000,000 |
| Requests per user per day | 100 |
| Bytes per request | 1,500 B |
| Peak factor | 12× |
| Retention | 7 days |
| Cache hit target | 0% |
| Per-partition budget | 50,000 QPS |
| Per-server budget | 100,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 115,741 req/s | 100,000,000 × 100 ÷ 86,400 |
| Peak QPS | 1,388,889 req/s | average × 12 |
| Storage per day | 15,000 GB | 10,000,000,000 requests × 1,500 B ÷ 10⁹ |
| Retained storage | 105 TB | per day × 7 days ÷ 1,000 |
| Peak bandwidth | 16.67 Gbps | peak × 1,500 B × 8 ÷ 10⁹ |
| Cache size | 0 GB | per day × 0% |
| Cache-miss origin QPS | 1,388,889 req/s | peak × 100% |
| Partitions | 28 | ⌈peak ÷ 50,000⌉ |
| Servers | 14 | ⌈peak ÷ 100,000⌉ |

</details>

---

### Authoritative DNS workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: One popular or attacked name attracts millions of queries per second.

| Given | Value |
|---|---|
| Daily active users | 100,000,000 |
| Requests per user per day | 40 |
| Bytes per request | 120 B |
| Peak factor | 15× |
| Retention | 30 days |
| Cache hit target | 95% |
| Per-partition budget | 50,000 QPS |
| Per-server budget | 150,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 46,296 req/s | 100,000,000 × 40 ÷ 86,400 |
| Peak QPS | 694,444 req/s | average × 15 |
| Storage per day | 480 GB | 4,000,000,000 requests × 120 B ÷ 10⁹ |
| Retained storage | 14.4 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 0.67 Gbps | peak × 120 B × 8 ÷ 10⁹ |
| Cache size | 456 GB | per day × 95% |
| Cache-miss origin QPS | 34,722 req/s | peak × 5% |
| Partitions | 14 | ⌈peak ÷ 50,000⌉ |
| Servers | 5 | ⌈peak ÷ 150,000⌉ |

</details>

---

### Metrics and alerting workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. The active unit is a metric series, not a person. Each series contributes 5,760 samples/day at a 15-second interval; 16 bytes is a raw sample assumption before indexes and compression. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: An instrumentation change adds request_id as a metric label.

| Given | Value |
|---|---|
| Daily active users | 50,000,000 |
| Requests per user per day | 5,760 |
| Bytes per request | 16 B |
| Peak factor | 2× |
| Retention | 30 days |
| Cache hit target | 20% |
| Per-partition budget | 100,000 QPS |
| Per-server budget | 60,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 3,333,333 req/s | 50,000,000 × 5760 ÷ 86,400 |
| Peak QPS | 6,666,667 req/s | average × 2 |
| Storage per day | 4,608 GB | 288,000,000,000 requests × 16 B ÷ 10⁹ |
| Retained storage | 138 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 0.85 Gbps | peak × 16 B × 8 ÷ 10⁹ |
| Cache size | 922 GB | per day × 20% |
| Cache-miss origin QPS | 5,333,333 req/s | peak × 80% |
| Partitions | 67 | ⌈peak ÷ 100,000⌉ |
| Servers | 112 | ⌈peak ÷ 60,000⌉ |

</details>

---

### Feature flag platform workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. A request is one flag evaluation. The high cache percentage models local SDK evaluation; remote configuration updates are a different workload. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Ten billion evaluations per day call the flag server synchronously.

| Given | Value |
|---|---|
| Daily active users | 100,000,000 |
| Requests per user per day | 100 |
| Bytes per request | 80 B |
| Peak factor | 5× |
| Retention | 30 days |
| Cache hit target | 99% |
| Per-partition budget | 5,000 QPS |
| Per-server budget | 30,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 115,741 req/s | 100,000,000 × 100 ÷ 86,400 |
| Peak QPS | 578,704 req/s | average × 5 |
| Storage per day | 800 GB | 10,000,000,000 requests × 80 B ÷ 10⁹ |
| Retained storage | 24 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 0.37 Gbps | peak × 80 B × 8 ÷ 10⁹ |
| Cache size | 792 GB | per day × 99% |
| Cache-miss origin QPS | 5,787 req/s | peak × 1% |
| Partitions | 116 | ⌈peak ÷ 5,000⌉ |
| Servers | 20 | ⌈peak ÷ 30,000⌉ |

</details>

---

### Service discovery and configuration workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. The active unit is one registered service instance sending 2,880 refreshes per day; watcher fanout must be modeled separately. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: Restarting 100K instances sends endpoint updates to every client.

| Given | Value |
|---|---|
| Daily active users | 1,000,000 |
| Requests per user per day | 2,880 |
| Bytes per request | 350 B |
| Peak factor | 8× |
| Retention | 7 days |
| Cache hit target | 90% |
| Per-partition budget | 15,000 QPS |
| Per-server budget | 20,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 33,333 req/s | 1,000,000 × 2880 ÷ 86,400 |
| Peak QPS | 266,667 req/s | average × 8 |
| Storage per day | 1,008 GB | 2,880,000,000 requests × 350 B ÷ 10⁹ |
| Retained storage | 7.06 TB | per day × 7 days ÷ 1,000 |
| Peak bandwidth | 0.75 Gbps | peak × 350 B × 8 ÷ 10⁹ |
| Cache size | 907 GB | per day × 90% |
| Cache-miss origin QPS | 26,667 req/s | peak × 10% |
| Partitions | 18 | ⌈peak ÷ 15,000⌉ |
| Servers | 14 | ⌈peak ÷ 20,000⌉ |

</details>

---

## Advanced

### Multi-region key-value store workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: One tenant key prefix generates 40% of global write traffic.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 40 |
| Bytes per request | 1,500 B |
| Peak factor | 8× |
| Retention | 90 days |
| Cache hit target | 40% |
| Per-partition budget | 4,000 QPS |
| Per-server budget | 6,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 4,630 req/s | 10,000,000 × 40 ÷ 86,400 |
| Peak QPS | 37,037 req/s | average × 8 |
| Storage per day | 600 GB | 400,000,000 requests × 1,500 B ÷ 10⁹ |
| Retained storage | 54 TB | per day × 90 days ÷ 1,000 |
| Peak bandwidth | 0.44 Gbps | peak × 1,500 B × 8 ÷ 10⁹ |
| Cache size | 240 GB | per day × 40% |
| Cache-miss origin QPS | 22,222 req/s | peak × 60% |
| Partitions | 10 | ⌈peak ÷ 4,000⌉ |
| Servers | 7 | ⌈peak ÷ 6,000⌉ |

</details>

---

### Durable workflow engine workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A workflow has accumulated a million events and replay takes minutes.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 10 |
| Bytes per request | 2,200 B |
| Peak factor | 10× |
| Retention | 365 days |
| Cache hit target | 5% |
| Per-partition budget | 1,800 QPS |
| Per-server budget | 2,500 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 1,157 req/s | 10,000,000 × 10 ÷ 86,400 |
| Peak QPS | 11,574 req/s | average × 10 |
| Storage per day | 220 GB | 100,000,000 requests × 2,200 B ÷ 10⁹ |
| Retained storage | 80.3 TB | per day × 365 days ÷ 1,000 |
| Peak bandwidth | 0.2 Gbps | peak × 2,200 B × 8 ÷ 10⁹ |
| Cache size | 11 GB | per day × 5% |
| Cache-miss origin QPS | 10,995 req/s | peak × 95% |
| Partitions | 7 | ⌈peak ÷ 1,800⌉ |
| Servers | 5 | ⌈peak ÷ 2,500⌉ |

</details>

---

### Ad auction and budget pacing workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: One campaign participates in millions of auctions across regions.

| Given | Value |
|---|---|
| Daily active users | 100,000,000 |
| Requests per user per day | 50 |
| Bytes per request | 900 B |
| Peak factor | 8× |
| Retention | 90 days |
| Cache hit target | 65% |
| Per-partition budget | 10,000 QPS |
| Per-server budget | 14,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 57,870 req/s | 100,000,000 × 50 ÷ 86,400 |
| Peak QPS | 462,963 req/s | average × 8 |
| Storage per day | 4,500 GB | 5,000,000,000 requests × 900 B ÷ 10⁹ |
| Retained storage | 405 TB | per day × 90 days ÷ 1,000 |
| Peak bandwidth | 3.33 Gbps | peak × 900 B × 8 ÷ 10⁹ |
| Cache size | 2,925 GB | per day × 65% |
| Cache-miss origin QPS | 162,037 req/s | peak × 35% |
| Partitions | 47 | ⌈peak ÷ 10,000⌉ |
| Servers | 34 | ⌈peak ÷ 14,000⌉ |

</details>

---

### Recommendation serving workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: The ranker waits 120ms for a rarely useful feature on every request.

| Given | Value |
|---|---|
| Daily active users | 100,000,000 |
| Requests per user per day | 30 |
| Bytes per request | 10,000 B |
| Peak factor | 8× |
| Retention | 30 days |
| Cache hit target | 55% |
| Per-partition budget | 5,000 QPS |
| Per-server budget | 2,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 34,722 req/s | 100,000,000 × 30 ÷ 86,400 |
| Peak QPS | 277,778 req/s | average × 8 |
| Storage per day | 30,000 GB | 3,000,000,000 requests × 10,000 B ÷ 10⁹ |
| Retained storage | 900 TB | per day × 30 days ÷ 1,000 |
| Peak bandwidth | 22.22 Gbps | peak × 10,000 B × 8 ÷ 10⁹ |
| Cache size | 16,500 GB | per day × 55% |
| Cache-miss origin QPS | 125,000 req/s | peak × 45% |
| Partitions | 56 | ⌈peak ÷ 5,000⌉ |
| Servers | 139 | ⌈peak ÷ 2,000⌉ |

</details>

---

### Streaming fraud detection workload

Use the supplied synthetic workload to estimate average and peak request rate, daily logical bytes, retained logical bytes, cache-miss origin QPS, and lower-bound partitions/servers. One active unit is a user and one request is a modeled application event or transfer; use the supplied mean byte size as an assumption, not a vendor benchmark. Assume all modeled requests contribute bytes to retained logical storage for this exercise; discuss why production read traffic, replication, compression, indexes, fanout, and retention tiers change that assumption. Then address: A mobile carrier NAT causes millions of users to share one IP feature bucket.

| Given | Value |
|---|---|
| Daily active users | 10,000,000 |
| Requests per user per day | 10 |
| Bytes per request | 1,800 B |
| Peak factor | 12× |
| Retention | 180 days |
| Cache hit target | 15% |
| Per-partition budget | 8,000 QPS |
| Per-server budget | 6,000 QPS |

<details><summary>Reference answers</summary>

| Estimate | Answer | Working |
|---|---|---|
| Average QPS | 1,157 req/s | 10,000,000 × 10 ÷ 86,400 |
| Peak QPS | 13,889 req/s | average × 12 |
| Storage per day | 180 GB | 100,000,000 requests × 1,800 B ÷ 10⁹ |
| Retained storage | 32.4 TB | per day × 180 days ÷ 1,000 |
| Peak bandwidth | 0.2 Gbps | peak × 1,800 B × 8 ÷ 10⁹ |
| Cache size | 27 GB | per day × 15% |
| Cache-miss origin QPS | 11,806 req/s | peak × 85% |
| Partitions | 2 | ⌈peak ÷ 8,000⌉ |
| Servers | 3 | ⌈peak ÷ 6,000⌉ |

</details>

---
