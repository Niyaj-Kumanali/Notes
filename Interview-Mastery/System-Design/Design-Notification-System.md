# Design Notification System

## Overview

- **Definition** — A system that delivers messages to users across multiple channels (push, email, SMS, in-app) with reliability, personalization, and delivery tracking
- **Why It Exists** — Applications need to reach users in real-time with transactional alerts (OTP, order confirmations) and engagement messages (marketing, reminders) across diverse delivery channels
- **Historical Context** — Early notification systems used SMTP for email and SMPP for SMS; Apple Push Notification Service (APNS, 2009) and Firebase Cloud Messaging (FCM, 2012) enabled mobile push at scale; modern systems integrate multiple channels with template engines and analytics
- **Key Concepts** — **Channel handlers** abstract delivery logic per channel; **Fan-out** sends a single notification to multiple channels; **Rate limiting per channel** prevents carrier blocks; **Deduplication** filters duplicate requests; **Exponential backoff** retries failed deliveries; **Priority queue** separates critical from bulk notifications

## Core Concepts

- Notification service receives requests and routes them to appropriate channel handlers
  - Ingress API accepts notification requests with recipient, channel list, content, and metadata
  - Validates request, checks user preferences, applies rate limits and deduplication
- Template engine renders message content with dynamic variables per channel
  - Templates per channel: email (HTML), SMS (plain text), push (title + body with optional image)
  - Variables (user name, order ID) are injected at render time
  - Supports localization: language-specific templates
- Fan-out sends a single notification to multiple channels simultaneously
  - One request can produce push, email, and SMS deliveries in parallel
  - Each channel handler operates independently — one channel failure does not affect others
- Rate limiting per channel prevents carrier blocks and spam complaints
  - Email: limit per sender domain per hour to avoid SMTP throttling
  - SMS: strict per-second rate to prevent carrier blacklisting
  - Push: APNS and FCM have per-device token rate limits
- Deduplication filters duplicate notification requests within a time window
  - Use idempotency key (provided by caller or generated from content hash)
  - Redis SET NX with TTL (e.g., 5 minutes) — if key exists, the request is a duplicate and is dropped
- Retry with exponential backoff retries failed deliveries at increasing intervals up to a maximum
  - Initial retry: 1 second, then 2s, 4s, 8s, max 60s
  - Maximum retry count: 3–5 depending on channel
  - After max retries, move to a dead-letter queue for manual inspection
- Priority queue separates critical notifications (OTP, security alerts) from bulk (marketing, newsletters)
  - High-priority queue: processed immediately, lower concurrency for reliability
  - Low-priority queue: batch-processed during off-peak, can be throttled more aggressively
- User preferences store opt-in and opt-out per channel, checked before delivery
  - NotificationPreferences table: user_id, channel, enabled (boolean), quiet_hours_start, quiet_hours_end
  - Checked before template rendering — if user opted out of SMS, skip that channel entirely
  - Global unsubscribe overrides all channels
- Delivery tracking records send, delivered, opened, and bounced events for each notification
  - Each notification has a unique ID; delivery events are captured at each stage
  - Webhook callbacks from push providers, email bounce handling via SNS/SES, SMS delivery receipts
  - Events stored in time-series database for analytics dashboards
- Scaling with worker pools and message queues
  - Notification ingestion writes to a message queue (Kafka, RabbitMQ, SQS)
  - Worker pools consume from queues — separate worker pools per channel for isolated scaling
  - Workers are stateless and can scale horizontally based on queue depth

## Common Mistakes

- **Notification storms overwhelming external providers**
  - Sending millions of identical notifications (e.g., "Someone liked your post") simultaneously, causing APNS/FCM to rate limit or block the app
  - **Why it looks correct:** Fans are excited about engagement, and sending each notification immediately seems best for user experience
  - Batch and aggregate notifications: "5 people liked your post" instead of 5 individual notifications. Apply rate limiting per recipient (max N notifications per hour per user). Use a cooldown period per notification type per user.
- **Ignoring user preferences and quiet hours**
  - Sending promotional emails at 2 AM to users who explicitly opted out or set quiet hours
  - **Why it looks correct:** The notification content was important and the preference check seemed like an optional optimization
  - Always check user preferences before queuing delivery. Store preferences in a fast-access cache (Redis) with the user's preference object. Enforce quiet hours server-side, not client-side (notifications can still arrive when the phone is silenced).
- **Not handling push token invalidation**
  - Continuously sending push notifications to expired or revoked device tokens, getting high error rates from APNS/FCM
  - **Why it looks correct:** The user was previously registered and the token was valid at that time
  - Track failed push deliveries by error code. APNS returns "Unregistered" for invalid tokens; remove or mark the token as inactive immediately. Periodically purge stale tokens from the database.

## Real-World Scenarios

### Transactional Notification Pipeline

- User places an order — the order service sends a notification request to the notification service
- Fan-out sends order confirmation via email, SMS, and push simultaneously
- Email rendered from a template with order details; push shows a short summary; SMS is a brief confirmation
- OTP notifications go to the high-priority queue for immediate processing

### Marketing Campaign Delivery

- Marketing team uploads a campaign with a list of target users
- System batches users into groups of 1000, checks preferences, renders templates, and enqueues to low-priority queue
- Rate limited to 10,000 emails per hour to avoid SMTP blacklisting
- Delivery and open rates tracked per campaign for analytics

### Social Media Notification Batching

- When a post gets 100 likes, the system batches them: "John, Maria, and 48 others liked your photo"
- Aggregation window: 30 seconds — notifications within the window are grouped
- Reduces push notification volume by 90% while keeping users informed

## Scenario-Based Questions

**Q: A celebrity joins your platform and 1 million users follow them. When the celebrity posts, your notification system tries to send 1 million push notifications simultaneously. APNS and FCM start returning 429 errors. What do you do?**

- Implement batched fan-out: group recipients into chunks of 1000 and process each chunk with a controlled delay
- Apply rate limiting per provider: track APNS and FCM throughput and throttle accordingly
- Use a priority queue with delivery budgets: high-value followers (verified, recently active) get priority
- Pre-aggregate: if the platform shows follower feeds, send a single notification "New post from X" and let the app handle in-feed display
- **Interview follow-up:** How would you handle the case where the celebrity posts 5 times in 30 minutes? Should you send 5 separate notifications or aggregate?

**Q: Your SMS provider starts returning errors after 10,000 messages — you discover they have a 5,000/hour rate limit you did not account for. How do you prevent this in the future?**

- Implement a rate limiter per provider: token bucket with refill rate matching provider limits
- Add a circuit breaker for each provider: if errors exceed a threshold (20% failure rate in 1 minute), stop sending to that provider and failover to a secondary provider
- Store provider rate limits in configuration, monitored and updated as agreements change
- **Interview follow-up:** How would you handle graceful degradation if all SMS providers are rate-limited or down?

## Interview Questions

- **How does a notification system handle fan-out to multiple channels?**
  - The notification service receives a request with a list of target channels. It forks the processing per channel in parallel, with each channel handler independently applying template rendering, rate limiting, and delivery. A single channel failure does not block other channels. Results are aggregated and returned asynchronously via webhook or callback.
- **Explain the role of template rendering in notification systems.**
  - Templates decouple notification content from delivery logic. Each channel has its own template (email HTML, SMS plain text, push title+body) with dynamic variables. The template engine renders content at delivery time, supporting localization, personalization, and consistent branding across channels.
- **How do you implement deduplication in a notification system?**
  - Require an idempotency key with each notification request, or generate one from a hash of (recipient, channel, content, timestamp_bucket). Use Redis SET NX with a TTL window (e.g., 5 minutes) — if the key exists, the request is a duplicate and is silently dropped. Log the duplicate for monitoring.
- **What is the difference between a priority queue and a standard queue for notifications?**
  - Priority queues separate critical notifications (OTP, security alerts, password resets) from bulk (marketing, newsletters). Critical notifications are processed with lower latency and higher reliability guarantees. Bulk notifications can be batched, delayed, and throttled. Priority queues can be implemented as separate physical queues or a single queue with priority sorting.

## Developer Recommendations

- **Always rate limit per channel and per provider**
  - Each delivery channel (email, SMS, push) has different rate limits from providers
  - Implement a token bucket per provider; monitor provider responses for 429s and scale down
  - **Production story:** An unthrottled campaign sent 100K SMS in 10 minutes, getting the SMS provider to temporarily block the account — rate limiting to 5,000/hour fixed the issue
- **Batch and aggregate notifications to prevent storms**
  - Combine multiple events into a single notification — "5 new messages" instead of 5 notifications
  - Use a 30-second aggregation window with a flush timer
  - **Production story:** A social app was sending 50M push notifications/day for likes — aggregation reduced it to 5M/day with no user satisfaction impact
- **Implement user preference checks early in the pipeline**
  - Check user preferences before template rendering or queue enqueue
  - Cache preferences in Redis with a 5-minute TTL to avoid repeated DB lookups
  - Include global unsubscribe, per-channel opt-in/opt-out, and quiet hours
- **Use dead-letter queues for failed deliveries**
  - After exhausting retries, move the notification to a dead-letter queue (DLQ)
  - Monitor DLQ depth and alert on anomalies — a sudden spike may indicate a provider outage or configuration error
  - Periodically reprocess DLQ messages after the root cause is fixed
- **Design for provider failover**
  - Each channel should have a primary and secondary provider configured
  - If the primary returns errors, automatically failover to the secondary after a circuit breaker threshold
  - Test failover regularly with chaos experiments
