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

## Use Cases

- **Push notifications for mobile apps** — iOS (APNS) and Android (FCM) notifications for engagement, alerts, and reminders
  - Service sends targeted push notifications via platform-specific gateways. Batched delivery respects rate limits. Handles device token refresh and unregistered devices.
  - **Avoid when:** the user is actively using the app — consider in-app notifications or WebSocket delivery instead of push.

- **Email notification delivery** — transactional emails (order confirmations, password resets) and marketing campaigns
  - Email sending via SMTP or third-party services (SendGrid, SES). Rate limiting prevents blacklisting. Template engine renders per-user content.
  - **Avoid when:** email is not time-sensitive — batch in a daily digest to reduce sending volume and cost.

- **SMS and WhatsApp messaging** — time-sensitive alerts (OTP codes, delivery status, appointment reminders)
  - Integration with Twilio or similar providers. Rate limits per number and per campaign. Delivery status callbacks for tracking.
  - **Avoid when:** the message can be delivered via push or email — SMS is expensive and should be reserved for high-priority or authentication messages.

- **In-app notification feed** — social media likes, comments, follows, or system announcements shown inside the app
  - Notification feed stored in database. Real-time delivery via WebSocket for active users. Pull-based loading for offline users when they reconnect.
  - **Avoid when:** notifications must be delivered to the user regardless of app state — combine with push notification fallback.

- **Notification batching and preference management** — daily digest emails, notification silencing, or per-channel opt-in/opt-out
  - Users configure which notification types they want and how often. Batched notifications reduce volume (e.g., "You have 5 new messages") while keeping users informed.
  - **Avoid when:** every notification is critical — all critical notifications should bypass batching and be delivered immediately.

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

**Q: Your notification system sends OTPs via SMS. During a holiday sale, SMS provider latency spikes to 10 seconds for OTP delivery. Users cannot log in. How do you fix this?**

- Add a secondary SMS provider and implement automatic failover when primary latency exceeds a threshold
- Offer alternative OTP delivery channels: email OTP, voice call, or in-app verification (push notification with OTP)
- Cache OTPs and allow a grace period: if the SMS is delayed, the user can request a resend without waiting
- **Interview follow-up:** How do you ensure OTP security when delivering through multiple channels?

**Q: Your marketing notification campaign sends 1 million emails. A significant portion bounces because the recipient email addresses are invalid. How do you handle email bounce processing?**

- Implement webhook integration with the email provider (SES, SendGrid) to receive bounce and complaint notifications
- Automatically mark bounced emails as invalid in the user preferences database and suppress future sends
- Separate hard bounces (invalid address, permanent) from soft bounces (mailbox full, temporary) — only hard bounces trigger suppression
- **Interview follow-up:** How do you handle the case where a legitimate email address temporarily hard-bounces due to a server error?

**Q: Your notification system sends a "welcome" email and push notification when a user signs up. Some users receive the push notification but never the email. How do you diagnose this?**

- Check the email delivery pipeline: was the email queued? Did the email provider accept it? Did it bounce?
- Verify the email template rendered correctly — if a template variable is missing, the email may be rejected as malformed
- Check the spam score — the email may have been flagged as spam and delivered to the spam folder instead of the inbox
- **Interview follow-up:** How do you implement an email delivery dashboard that shows the end-to-end status of every notification?

**Q: Your notification system needs to support in-app notifications (bell icon) alongside push and email. How do you design the in-app notification channel?**

- Store in-app notifications in a database table: user_id, notification_content, is_read, created_at
- The client polls or uses WebSocket to fetch unread notification count and content
- In-app notifications should be synchronized with push notifications — if the user is online, suppress push and deliver in-app only
- **Interview follow-up:** How do you implement "mark all as read" for in-app notifications when there are thousands of unread notifications?

**Q: A bug in your notification system causes the same notification to be sent 5 times to each user. How do you clean up the mess and prevent recurrence?**

- Implement idempotency keys on every notification request — the key is a hash of (user_id, notification_type, content, time window)
- Use Redis SET NX with a 5-minute TTL for deduplication at the ingestion layer
- For the current situation, send a bulk apology notification and implement a job to cancel duplicate deliveries that are still in the queue
- **Interview follow-up:** How do you identify and remove duplicate notifications that were already delivered to users?

**Q: Your notification system sends real-time notifications via WebSocket. When a user is connected on multiple devices, each device receives the same notification. How do you handle multi-device delivery?**

- Track all active device connections per user in a Redis set (key: user:{user_id}:devices)
- Fan-out the notification to all connected devices simultaneously via their respective WebSocket connections
- Track per-device read status so that marking a notification as read on one device syncs to all devices
- **Interview follow-up:** How do you handle the case where a notification is read on one device but the push notification on another device is still showing?

**Q: Your notification system needs to support scheduled notifications (e.g., "Your appointment is tomorrow at 10 AM"). How do you design the scheduling mechanism?**

- Store scheduled notifications in a database table with a scheduled_at timestamp and a status column (PENDING, SENT, FAILED)
- Run a scheduler service that polls for due notifications every minute, enqueues them to the notification pipeline, and marks them as SENT
- Use a job scheduler (Sidekiq, Celery, AWS Step Functions) with delayed execution for precise timing
- **Interview follow-up:** How do you handle timezone differences — should scheduled times be stored in UTC or user local time?

**Q: Your notification system logs all delivery attempts in a database. At 10 million notifications per day, the logs table grows by 10M rows daily. Queries on the logs become slow. How do you handle log storage?**

- Partition the logs table by date (e.g., notification_logs_2024_01, notification_logs_2024_02) for efficient query pruning
- Move logs older than 30 days to a cold storage (S3, Glacier) or a cheaper analytics database (ClickHouse)
- For real-time monitoring, keep only the last 7 days in the primary database and stream logs to an analytics pipeline in parallel
- **Interview follow-up:** How do you query across partitions for a user's notification history that spans multiple months?

## Interview Questions

- **How does a notification system handle fan-out to multiple channels?**
  - The notification service receives a request with a list of target channels. It forks the processing per channel in parallel, with each channel handler independently applying template rendering, rate limiting, and delivery. A single channel failure does not block other channels. Results are aggregated and returned asynchronously via webhook or callback.
- **Explain the role of template rendering in notification systems.**
  - Templates decouple notification content from delivery logic. Each channel has its own template (email HTML, SMS plain text, push title+body) with dynamic variables. The template engine renders content at delivery time, supporting localization, personalization, and consistent branding across channels.
- **How do you implement deduplication in a notification system?**
  - Require an idempotency key with each notification request, or generate one from a hash of (recipient, channel, content, timestamp_bucket). Use Redis SET NX with a TTL window (e.g., 5 minutes) — if the key exists, the request is a duplicate and is silently dropped. Log the duplicate for monitoring.
- **What is the difference between a priority queue and a standard queue for notifications?**
  - Priority queues separate critical notifications (OTP, security alerts, password resets) from bulk (marketing, newsletters). Critical notifications are processed with lower latency and higher reliability guarantees. Bulk notifications can be batched, delayed, and throttled. Priority queues can be implemented as separate physical queues or a single queue with priority sorting.
- **How do you implement notification aggregation for social media likes?**
  - Set a 30-second aggregation window. Collect all "like" events for a post within the window. Render a single notification: "Alice, Bob, and 12 others liked your post." Use a Redis sorted set with the post ID as key and user IDs as members with timestamps for ordering within the window.
- **Explain the role of a "template engine" in a notification system.**
  - Template engines separate notification content from code. Each channel (email, SMS, push) has its own template with placeholders. The engine renders templates with dynamic variables at delivery time. This enables localization, branding consistency, and campaign management without code changes.
- **How do you handle push notification token expiration?**
  - APNS and FCM return error codes for expired tokens. When a push delivery returns "Unregistered" (APNS) or "NotRegistered" (FCM), immediately remove the token from the user's device list. Periodically purge tokens that have not been refreshed in N days. Log token expiration rates for monitoring.
- **What is the difference between "transactional" and "marketing" notifications in terms of architecture?**
  - Transactional: triggered by user actions (order confirmation, password reset), low latency required, high priority, cannot be batched. Marketing: scheduled campaigns, can be batched and delayed, lower priority, subject to rate limits and quiet hours. They should use separate queues and worker pools.
- **How do you implement "quiet hours" server-side?**
  - Store quiet hours per user (e.g., 10 PM — 8 AM). Before delivering a notification, check if the current time falls within quiet hours. If so, delay delivery until quiet hours end or suppress non-critical notifications. Critical notifications (security alerts) can bypass quiet hours with a flag.
- **How do you scale the notification ingestion API?**
  - The ingestion API is stateless and can scale horizontally behind a load balancer. The bottleneck is the message queue and the worker pools. Use auto-scaling for workers based on queue depth. Partition the queue (Kafka topics, SQS queues) by notification priority or channel type for isolated scaling.
- **Explain the architecture for multi-channel fallback.**
  - When a notification request specifies multiple channels, send via the primary channel first. If delivery fails (after retries), fallback to the next channel. Example: if push notification fails because the user is offline, send SMS instead. The fallback order is configurable per notification type.
- **How do you handle notification unsubscribe/opt-out?**
  - Store opt-out preferences per user per channel in a fast-access cache. Check preferences before any delivery attempt. Provide a one-click unsubscribe link in email footers. For push, provide in-app settings. The unsubscribe action should be reflected within seconds, not hours.
- **What is the role of "batch processing" in notification systems?**
  - Batch processing groups multiple notification requests into a single operation to improve throughput. Examples: batching emails by recipient domain for better SMTP reuse, batching push notifications to APNS/FCM in chunks of 100, batching SMS for carrier delivery. Batching reduces per-message overhead.
- **How do you implement delivery receipts and read receipts?**
  - Each notification has a unique ID. Email: track opens via a tracking pixel or link click. Push: APNS/FCM provide delivery and open callbacks. SMS: delivery receipts from the carrier. Store receipts as events with timestamp. Provide a webhook API for clients to receive delivery status callbacks.
- **How do you handle internationalization (i18n) in notification templates?**
  - Store template variants per locale (en_US, fr_FR, ja_JP). The template engine selects the template based on the user's language preference. Variables are locale-independent. Use ICU message format for plurals and gender-specific content. Fallback to English if the requested locale template is missing.
- **What is the difference between "push notification" and "in-app notification"?**
  - Push: sent by the server to the device via APNS/FCM, delivered even when the app is closed, shown in the notification tray. In-app: delivered via WebSocket or polling when the app is open, shown in the app's UI. Push is for offline reach; in-app is for the real-time experience.
- **How do you design a notification preferences UI for users?**
  - Group notification types into categories (transactions, marketing, reminders). Allow per-channel opt-in/opt-out per category. Provide a global mute toggle. Show a preview of each notification type. Store preferences in a JSON blob per user for flexibility, with a cache layer for fast reads.
- **How do you monitor notification system health?**
  - Track per-channel metrics: delivery rate, error rate, latency (p50/p99), bounce rate, spam complaint rate. Monitor queue depths and worker pool utilization. Alert on sudden drops in delivery rate (possible provider outage) or spikes in error rates. Use synthetic monitoring: send test notifications periodically and verify delivery.
- **How do you handle notification retries with idempotency?**
  - When a delivery fails, retry with exponential backoff (1s, 2s, 4s, max 3 retries). The notification ID serves as the idempotency key: the delivery worker checks if the notification was already delivered before retrying. For critical notifications, use a "retry until acknowledged" strategy with a dead-letter queue after max retries.
- **How do you design a notification system for regulatory compliance (GDPR, CAN-SPAM)?**
  - Store consent records per user per channel with timestamps. Include unsubscribe links in every email (CAN-SPAM requirement). Support data deletion requests — purge notification history for deleted users. Keep audit logs of consent changes. For GDPR, ensure notification data is stored within the user's region or provide explicit cross-border transfer consent.

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
