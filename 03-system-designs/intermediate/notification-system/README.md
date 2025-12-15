# Design a Notification System

## Table of Contents
- [Problem Statement](#problem-statement)
- [Requirements](#requirements)
- [High-Level Design](#high-level-design)
- [Notification Types](#notification-types)
- [Core Components](#core-components)
- [Message Queue Architecture](#message-queue-architecture)
- [Template Management](#template-management)
- [Priority and Rate Limiting](#priority-and-rate-limiting)
- [Scalability](#scalability)
- [Implementation Examples](#implementation-examples)
- [Real-World Examples](#real-world-examples)
- [Interview Tips](#interview-tips)

---

## Problem Statement

Design a **notification system** that can send notifications through multiple channels including:
- Push notifications (mobile/web)
- Email
- SMS
- In-app notifications

**Similar to**: Facebook notifications, Slack notifications, Airbnb notifications

---

## Requirements

### Functional Requirements

1. **Multi-channel delivery**: Push, email, SMS, in-app
2. **User preferences**: Users can opt-in/opt-out per channel
3. **Template management**: Reusable notification templates
4. **Personalization**: Dynamic content based on user data
5. **Retry logic**: Handle delivery failures
6. **Notification history**: Track sent notifications
7. **Analytics**: Delivery rates, open rates, click rates

### Non-Functional Requirements

1. **Scalability**: Handle millions of notifications per day
2. **Reliability**: 99.9% delivery rate
3. **Low Latency**: Near real-time delivery (<1 second for high priority)
4. **Fault Tolerance**: Continue operating despite failures
5. **Extensibility**: Easy to add new channels
6. **Rate Limiting**: Avoid overwhelming users
7. **Cost Efficiency**: Optimize third-party API costs

---

## High-Level Design

```
┌────────────────────────────────────────────────────────────┐
│                    Notification Triggers                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ User     │  │ Order    │  │ Payment  │  │ Marketing│  │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
└───────┼─────────────┼─────────────┼─────────────┼─────────┘
        │             │             │             │
        └─────────────┼─────────────┼─────────────┘
                      ↓
┌────────────────────────────────────────────────────────────┐
│              Notification Service (API)                     │
│                                                             │
│  1. Validate request                                       │
│  2. Check user preferences                                 │
│  3. Apply rate limiting                                    │
│  4. Render templates                                       │
│  5. Enqueue to message queue                               │
└────────────────────┬───────────────────────────────────────┘
                     ↓
┌────────────────────────────────────────────────────────────┐
│                   Message Queue (Kafka/SQS)                │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Email   │  │   SMS    │  │   Push   │  │  In-App  │  │
│  │  Queue   │  │  Queue   │  │  Queue   │  │  Queue   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
└───────┼─────────────┼─────────────┼─────────────┼─────────┘
        ↓             ↓             ↓             ↓
┌────────────────────────────────────────────────────────────┐
│                    Channel Workers                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Email   │  │   SMS    │  │   Push   │  │  In-App  │  │
│  │  Worker  │  │  Worker  │  │  Worker  │  │  Worker  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
└───────┼─────────────┼─────────────┼─────────────┼─────────┘
        ↓             ↓             ↓             ↓
┌────────────────────────────────────────────────────────────┐
│              Third-Party Services                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ SendGrid │  │  Twilio  │  │   FCM    │                 │
│  │   SES    │  │   SNS    │  │   APNs   │                 │
│  └──────────┘  └──────────┘  └──────────┘                 │
└────────────────────────────────────────────────────────────┘
        │             │             │
        ↓             ↓             ↓
┌────────────────────────────────────────────────────────────┐
│                      End Users                              │
│  📧 Email       📱 SMS       🔔 Push      💬 In-App        │
└────────────────────────────────────────────────────────────┘
```

---

## Notification Types

### 1. Push Notifications

**Mobile:**
- iOS: Apple Push Notification Service (APNs)
- Android: Firebase Cloud Messaging (FCM)

**Web:**
- Web Push API (service workers)

**Example:**
```json
{
  "notification_id": "notif_123",
  "user_id": "user_456",
  "type": "push",
  "platform": "ios",
  "device_token": "abc123...",
  "title": "New message from Alice",
  "body": "Hey, are you free for coffee?",
  "data": {
    "chat_id": "chat_789",
    "message_id": "msg_101"
  },
  "badge": 5,
  "sound": "default"
}
```

### 2. Email Notifications

**Providers:**
- SendGrid
- Amazon SES
- Mailgun

**Example:**
```json
{
  "notification_id": "notif_124",
  "user_id": "user_456",
  "type": "email",
  "to": "user@example.com",
  "from": "noreply@app.com",
  "subject": "Your order #12345 has shipped",
  "template_id": "order_shipped",
  "variables": {
    "order_id": "12345",
    "tracking_number": "1Z999AA10123456784",
    "estimated_delivery": "Dec 18, 2024"
  }
}
```

### 3. SMS Notifications

**Providers:**
- Twilio
- Amazon SNS
- Nexmo

**Example:**
```json
{
  "notification_id": "notif_125",
  "user_id": "user_456",
  "type": "sms",
  "phone_number": "+1234567890",
  "message": "Your verification code is: 123456",
  "sender_id": "MyApp"
}
```

### 4. In-App Notifications

**Stored in database, displayed when user opens app**

```json
{
  "notification_id": "notif_126",
  "user_id": "user_456",
  "type": "in_app",
  "title": "Welcome to our app!",
  "message": "Complete your profile to get started",
  "icon": "profile_icon",
  "action_url": "/profile/edit",
  "read": false,
  "created_at": "2024-12-15T10:00:00Z"
}
```

---

## Core Components

### 1. Notification Service API

```python
from flask import Flask, request, jsonify
import uuid
from datetime import datetime

app = Flask(__name__)

class NotificationService:
    def __init__(self, queue, user_service, template_service):
        self.queue = queue
        self.user_service = user_service
        self.template_service = template_service

    def send_notification(self, notification_request):
        """
        Main entry point for sending notifications

        1. Validate request
        2. Check user preferences
        3. Render template
        4. Fan out to channels
        5. Enqueue messages
        """
        notification_id = str(uuid.uuid4())

        # 1. Get user preferences
        preferences = self.user_service.get_preferences(
            notification_request['user_id']
        )

        # 2. Determine channels
        channels = self._select_channels(
            notification_request,
            preferences
        )

        # 3. Render template
        rendered_messages = self._render_template(
            notification_request,
            channels
        )

        # 4. Enqueue to message queues
        for channel, message in rendered_messages.items():
            self.queue.publish(
                queue=f"notifications.{channel}",
                message={
                    'notification_id': notification_id,
                    'channel': channel,
                    'payload': message,
                    'user_id': notification_request['user_id'],
                    'priority': notification_request.get('priority', 'normal'),
                    'retry_count': 0,
                    'created_at': datetime.utcnow().isoformat()
                }
            )

        return {
            'notification_id': notification_id,
            'status': 'queued',
            'channels': list(channels)
        }

    def _select_channels(self, request, preferences):
        """Select delivery channels based on notification type and preferences"""
        requested_channels = request.get('channels', ['push', 'email'])
        enabled_channels = []

        for channel in requested_channels:
            # Check if user has enabled this channel
            if preferences.get(f'{channel}_enabled', True):
                # Check if user has necessary info (email, phone, device token)
                if self._has_channel_info(request['user_id'], channel):
                    enabled_channels.append(channel)

        return enabled_channels

    def _render_template(self, request, channels):
        """Render notification content for each channel"""
        template_id = request.get('template_id')
        variables = request.get('variables', {})

        rendered = {}
        for channel in channels:
            template = self.template_service.get_template(template_id, channel)
            rendered[channel] = template.render(variables)

        return rendered

@app.route('/api/v1/notifications', methods=['POST'])
def send_notification():
    """
    POST /api/v1/notifications
    {
      "user_id": "user_123",
      "template_id": "order_shipped",
      "channels": ["email", "push"],
      "variables": {
        "order_id": "12345",
        "tracking_number": "ABC123"
      },
      "priority": "high"
    }
    """
    request_data = request.json
    result = notification_service.send_notification(request_data)
    return jsonify(result), 202
```

### 2. User Preference Service

```python
class UserPreferenceService:
    def __init__(self, db):
        self.db = db

    def get_preferences(self, user_id):
        """Get user notification preferences"""
        return self.db.query("""
            SELECT
                email_enabled,
                sms_enabled,
                push_enabled,
                in_app_enabled,
                email_address,
                phone_number,
                device_tokens,
                quiet_hours_start,
                quiet_hours_end,
                frequency_limit
            FROM user_preferences
            WHERE user_id = %s
        """, user_id)

    def update_preferences(self, user_id, preferences):
        """Update user preferences"""
        self.db.update('user_preferences', user_id, preferences)

    def check_quiet_hours(self, user_id):
        """Check if current time is within user's quiet hours"""
        prefs = self.get_preferences(user_id)
        now = datetime.now().time()

        if prefs['quiet_hours_start'] and prefs['quiet_hours_end']:
            start = prefs['quiet_hours_start']
            end = prefs['quiet_hours_end']

            if start <= now <= end:
                return True  # In quiet hours

        return False
```

### 3. Channel Workers

#### Email Worker

```python
import boto3
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail

class EmailWorker:
    def __init__(self, provider='sendgrid'):
        if provider == 'sendgrid':
            self.client = SendGridAPIClient(api_key=SENDGRID_API_KEY)
        elif provider == 'ses':
            self.client = boto3.client('ses')

    def process_message(self, message):
        """Process email notification from queue"""
        try:
            payload = message['payload']

            # Send email
            self.send_email(
                to=payload['to'],
                subject=payload['subject'],
                body=payload['body'],
                html=payload.get('html')
            )

            # Update status
            self.update_status(message['notification_id'], 'sent')

        except Exception as e:
            # Retry logic
            if message['retry_count'] < 3:
                self.retry(message)
            else:
                self.update_status(message['notification_id'], 'failed')
                self.log_error(message['notification_id'], str(e))

    def send_email(self, to, subject, body, html=None):
        """Send email via SendGrid"""
        message = Mail(
            from_email='noreply@app.com',
            to_emails=to,
            subject=subject,
            plain_text_content=body,
            html_content=html
        )

        response = self.client.send(message)
        return response.status_code == 202
```

#### Push Notification Worker

```python
from pyfcm import FCMNotification
import requests

class PushNotificationWorker:
    def __init__(self):
        self.fcm = FCMNotification(api_key=FCM_SERVER_KEY)

    def process_message(self, message):
        """Process push notification from queue"""
        payload = message['payload']
        platform = payload['platform']

        if platform == 'ios':
            self.send_apns(payload)
        elif platform == 'android':
            self.send_fcm(payload)
        elif platform == 'web':
            self.send_web_push(payload)

    def send_fcm(self, payload):
        """Send via Firebase Cloud Messaging (Android)"""
        result = self.fcm.notify_single_device(
            registration_id=payload['device_token'],
            message_title=payload['title'],
            message_body=payload['body'],
            data_message=payload.get('data', {}),
            badge=payload.get('badge'),
            sound=payload.get('sound', 'default')
        )
        return result

    def send_apns(self, payload):
        """Send via Apple Push Notification Service (iOS)"""
        # Use APNs HTTP/2 API
        headers = {
            'authorization': f'bearer {APNS_JWT_TOKEN}',
            'apns-topic': BUNDLE_ID,
            'apns-priority': '10'
        }

        body = {
            'aps': {
                'alert': {
                    'title': payload['title'],
                    'body': payload['body']
                },
                'badge': payload.get('badge'),
                'sound': payload.get('sound', 'default')
            },
            **payload.get('data', {})
        }

        response = requests.post(
            f'https://api.push.apple.com/3/device/{payload["device_token"]}',
            headers=headers,
            json=body
        )

        return response.status_code == 200
```

#### SMS Worker

```python
from twilio.rest import Client

class SMSWorker:
    def __init__(self):
        self.client = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)

    def process_message(self, message):
        """Process SMS notification from queue"""
        payload = message['payload']

        try:
            self.send_sms(
                to=payload['phone_number'],
                body=payload['message']
            )
            self.update_status(message['notification_id'], 'sent')

        except Exception as e:
            self.handle_error(message, e)

    def send_sms(self, to, body):
        """Send SMS via Twilio"""
        message = self.client.messages.create(
            from_=TWILIO_PHONE_NUMBER,
            to=to,
            body=body
        )
        return message.sid
```

---

## Message Queue Architecture

### Fan-Out Pattern

```python
class NotificationFanOut:
    """
    Fan out notification to multiple channels
    Each channel has its own queue for independent processing
    """
    def __init__(self, kafka_producer):
        self.producer = kafka_producer

    def fan_out(self, notification, channels):
        """Send to multiple channel queues"""
        for channel in channels:
            topic = f'notifications.{channel}'

            self.producer.send(
                topic=topic,
                key=notification['user_id'],
                value=notification
            )

# Kafka topics:
# - notifications.email
# - notifications.sms
# - notifications.push
# - notifications.in_app
```

### Priority Queues

```python
class PriorityQueueManager:
    """
    Handle high-priority notifications first
    """
    def __init__(self):
        self.queues = {
            'high': [],
            'normal': [],
            'low': []
        }

    def enqueue(self, notification):
        """Add notification to appropriate priority queue"""
        priority = notification.get('priority', 'normal')
        self.queues[priority].append(notification)

    def dequeue(self):
        """Get next notification (highest priority first)"""
        for priority in ['high', 'normal', 'low']:
            if self.queues[priority]:
                return self.queues[priority].pop(0)
        return None
```

---

## Template Management

### Template Storage

```python
class TemplateService:
    def __init__(self, db):
        self.db = db
        self.cache = {}

    def get_template(self, template_id, channel):
        """Get template for specific channel"""
        cache_key = f"{template_id}:{channel}"

        if cache_key in self.cache:
            return self.cache[cache_key]

        template = self.db.query("""
            SELECT content, subject, variables
            FROM notification_templates
            WHERE template_id = %s AND channel = %s
        """, template_id, channel)

        self.cache[cache_key] = Template(template)
        return self.cache[cache_key]

class Template:
    def __init__(self, template_data):
        self.subject = template_data['subject']
        self.content = template_data['content']
        self.variables = template_data['variables']

    def render(self, variables):
        """Render template with variables"""
        rendered_content = self.content
        rendered_subject = self.subject

        for key, value in variables.items():
            placeholder = f"{{{{{key}}}}}"
            rendered_content = rendered_content.replace(placeholder, str(value))
            rendered_subject = rendered_subject.replace(placeholder, str(value))

        return {
            'subject': rendered_subject,
            'content': rendered_content
        }

# Database schema
"""
CREATE TABLE notification_templates (
    template_id VARCHAR(50),
    channel VARCHAR(20),  -- email, sms, push, in_app
    language VARCHAR(10),
    subject VARCHAR(255),
    content TEXT,
    variables JSONB,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    PRIMARY KEY (template_id, channel, language)
);
"""

# Example template
"""
{
  "template_id": "order_shipped",
  "channel": "email",
  "language": "en",
  "subject": "Your order {{order_id}} has shipped!",
  "content": "Hi {{user_name}},\n\nYour order {{order_id}} has been shipped!\nTracking: {{tracking_number}}\nEstimated delivery: {{delivery_date}}",
  "variables": ["user_name", "order_id", "tracking_number", "delivery_date"]
}
"""
```

---

## Priority and Rate Limiting

### Rate Limiting

```python
class NotificationRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    def check_rate_limit(self, user_id, channel):
        """
        Prevent notification spam
        Example: Max 10 emails per hour
        """
        key = f"rate_limit:notifications:{user_id}:{channel}"
        window = 3600  # 1 hour

        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, window)

        limit = self.get_limit(channel)

        if count > limit:
            return False, limit - count  # Exceeded

        return True, limit - count  # Allowed

    def get_limit(self, channel):
        """Channel-specific limits"""
        limits = {
            'email': 10,    # 10 per hour
            'sms': 5,       # 5 per hour (expensive)
            'push': 50,     # 50 per hour
            'in_app': 100   # 100 per hour
        }
        return limits.get(channel, 10)
```

### Deduplication

```python
class NotificationDeduplicator:
    def __init__(self, redis_client):
        self.redis = redis_client

    def is_duplicate(self, notification):
        """
        Prevent duplicate notifications
        Hash: user_id + template_id + variables
        """
        key = self._generate_hash(notification)
        cache_key = f"notification:sent:{key}"

        # Check if sent recently (last 24 hours)
        if self.redis.exists(cache_key):
            return True

        # Mark as sent
        self.redis.setex(cache_key, 86400, '1')  # 24 hour TTL
        return False

    def _generate_hash(self, notification):
        """Generate unique hash for notification"""
        import hashlib
        import json

        content = json.dumps({
            'user_id': notification['user_id'],
            'template_id': notification.get('template_id'),
            'key_variables': notification.get('variables', {})
        }, sort_keys=True)

        return hashlib.md5(content.encode()).hexdigest()
```

---

## Scalability

### Horizontal Scaling

```python
# Multiple workers per channel
# Use Kubernetes for auto-scaling

apiVersion: apps/v1
kind: Deployment
metadata:
  name: email-worker
spec:
  replicas: 10  # Scale based on queue depth
  selector:
    matchLabels:
      app: email-worker
  template:
    metadata:
      labels:
        app: email-worker
    spec:
      containers:
      - name: email-worker
        image: notification-worker:latest
        env:
        - name: CHANNEL
          value: "email"
```

### Database Partitioning

```sql
-- Partition notification history by month
CREATE TABLE notifications_2024_12 PARTITION OF notifications
FOR VALUES FROM ('2024-12-01') TO ('2024-12-31');
```

---

## Real-World Examples

### Facebook Notifications

- **Batching**: Group related notifications
- **Smart delivery**: Send when user likely to engage
- **Multi-device sync**: Mark as read across devices

### Slack Notifications

- **Threading**: Group notifications by conversation
- **@mentions**: High priority for mentions
- **Keyword alerts**: Custom notification triggers

---

## Interview Tips

### Key Points

1. **Multi-channel architecture**: Separate queues per channel
2. **User preferences**: Respect opt-outs, quiet hours
3. **Scalability**: Message queues, worker pools
4. **Reliability**: Retries, dead letter queues
5. **Rate limiting**: Prevent spam
6. **Analytics**: Track delivery, engagement

### Common Questions

**Q: How do you handle millions of notifications?**
- Message queue (Kafka/SQS) for async processing
- Multiple workers per channel
- Database sharding for notification history

**Q: What if email provider is down?**
- Retry with exponential backoff
- Switch to backup provider
- Dead letter queue for failed messages

**Q: How do you prevent notification spam?**
- Rate limiting per user per channel
- Batching related notifications
- Smart delivery timing

This notification system design demonstrates multi-channel communication, scalability, and reliability - essential for modern applications.
