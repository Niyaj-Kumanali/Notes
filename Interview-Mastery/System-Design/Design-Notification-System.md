# Design Notification System

- Requirements: push notifications, email, SMS, in-app notifications
- Notification service receives requests and routes them to the appropriate channel handlers
- Template engine renders message content with dynamic variables per channel
- Fan-out sends a single notification to multiple channels simultaneously
- Rate limiting per channel prevents carrier blocks and spam complaints
- Deduplication filters duplicate notification requests within a time window using a bloom filter or Redis
- Retry with exponential backoff retries failed deliveries at increasing intervals up to a maximum
- Priority queue separates critical notifications (OTP, security alerts) from bulk (marketing, newsletters)
- User preferences store opt-in and opt-out per channel, checked before delivery
- Delivery tracking records send, delivered, opened, and bounced events for each notification
- Analytics dashboards show delivery rates, open rates, and failure reasons
- Scaling requires worker pools consuming from message queues like Kafka or SQS
- Message queues decouple notification ingestion from delivery processing for elasticity
