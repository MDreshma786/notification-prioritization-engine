Notification Prioritization Engine
Round 1 – AI-Native Solution Crafting Test
Company: Cyepro Solutions
1. Problem Summary
Users receive too many notifications such as:
Repeated alerts
Low-value promotional messages
Notifications at wrong time
Important alerts getting delayed
The goal is to classify each incoming notification into:
NOW – Send immediately
LATER – Defer and schedule
NEVER – Suppress
The system must:
Prevent duplicate notifications
Reduce alert fatigue
Handle high volume traffic
Provide clear explanation for decisions
Fail safely if any service fails
2. High-Level Architecture
The system follows an event-driven architecture.
Main Components:
1. API Gateway
Receives incoming notification requests.
Handles authentication and rate limiting.
2. Ingestion Service
Validates the request format.
Generates a unique event ID.
Pushes event into message queue.
3. Message Queue (Kafka / RabbitMQ)
Handles high volume.
Ensures asynchronous and scalable processing.
4. Decision Engine (Core Component)
This is the brain of the system. It includes:
Rule Engine
ML Scoring Module
Deduplication Module
Alert Fatigue Manager
5. Scheduler Service
Sends NOW notifications.
Schedules LATER notifications.
Suppresses NEVER notifications.
6. Audit & Logging Service
Stores decision and explanation.
Supports audit and debugging.
7. Config Service
Allows admin to change rules without code deployment.
3. Decision Logic Design
The decision is based on:
Hard Rules + Scoring + Fatigue Check
Step 1: Hard Rules (Highest Priority)
If any of these conditions match:
Notification expired → NEVER
Exact duplicate → NEVER
User muted channel → NEVER
Critical system alert → NOW
These rules override everything.
Step 2: Scoring Mechanism
Final Score =
Priority Hint Weight
ML Importance Score
– Fatigue Penalty
– Recent Frequency Penalty
Score Decision:
Score above 70 → NOW
Score between 40 and 70 → LATER
Score below 40 → NEVER
Step 3: Conflict Handling
If notification is urgent but user already received many alerts:
If marked Critical → Send NOW
Otherwise → Defer for 10–15 minutes
4. Minimal Data Model
Table: notification_events
event_id
user_id
event_type
message
source
timestamp
channel
priority_hint
expires_at
Table: user_notification_history
user_id
last_sent_at
sent_count_last_1hr
sent_count_last_24hr
Table: suppression_records
event_id
reason
Table: audit_logs
event_id
decision
explanation
decision_timestamp
5. API Design
POST /notifications/evaluate
API Design
The API layer allows external services and administrators to interact with the Notification Prioritization Engine.
1. POST /notifications/evaluate
Purpose:
This is the main API of the system.
It receives a notification event and returns a decision (NOW / LATER / NEVER).
What it does:
Validates input
Checks hard rules
Runs duplicate detection
Applies scoring logic
Applies fatigue check
Logs explanation
Returns final decision
Example Input:
JSON
Copy code
{
  "user_id": "U123",
  "event_type": "order_update",
  "message": "Your order has been shipped",
  "source": "order_service",
  "priority_hint": 60,
  "timestamp": "2026-02-26T10:30:00",
  "channel": "push"
}
Example Output:
JSON
Copy code
{
  "decision": "NOW",
  "reason": "High priority and user has not exceeded hourly limit"
}
2. GET /notifications/history/{user_id}
Purpose:
Returns recent notification history for a user.
This is used for:
Alert fatigue management
Debugging
Monitoring
Example Request:
GET /notifications/history/U123
Example Response:
JSON
Copy code
{
  "last_sent_at": "2026-02-26T10:00:00",
  "sent_count_last_1hr": 2,
  "sent_count_last_24hr": 7
}
3. POST /config/rules
Purpose:
Allows administrators to update system rules without redeploying code.
Examples of configurable rules:
Maximum notifications per hour
Maximum notifications per day
Score thresholds
Cooldown duration
Example Input:
JSON
Copy code
{
  "max_per_hour": 4,
  "max_per_day": 12,
  "score_threshold_now": 75
}
This ensures the system is human-configurable as required.
4. GET /audit/{event_id}
Purpose:
Returns the explanation for a specific notification decision.
Used for:
Auditability
Transparency
User complaints investigation
Example Request:
GET /audit/EVENT_789
Example Response:
JSON
Copy code
{
  "decision": "LATER",
  "reason": "User exceeded hourly limit. Deferred by 15 minutes."
}
Why This API Design Works
Keeps system simple and modular
Covers evaluation, history, configuration, and auditing
Supports scalability
7. Duplicate Prevention Strategy
Exact Duplicate Handling
If dedupe_key matches within 5 minutes → Suppress.
Near Duplicate Handling
Compare message similarity using hashing or cosine similarity.
If similarity is above 90% → Suppress.
Example:
“Order shipped”
“Your order has been shipped”
8. Alert Fatigue Strategy
Cooldown Policy
Maximum 3 notifications per hour
Maximum 10 per day
Smart Deferral
If user received 3 notifications in last 10 minutes:
Defer new ones by 15 minutes.
Digest Mode
Promotional notifications are batched and sent once per day.
9. Fallback Strategy
If ML model fails:
Switch to rule-based engine.
If queue overload:
Drop promotional notifications first.
Never drop critical alerts.
If database fails:
Use in-memory cache and retry mechanism.
Fail-safe default:
Critical → NOW
Unknown → LATER
10. Monitoring & Metrics
Track:
NOW / LATER / NEVER ratio
Duplicate suppression rate
Average decision latency
Delivery success rate
User opt-out rate
Alert fatigue violations
Monitoring tools can include Prometheus and Grafana.
11. Why This Design Works
Scalable architecture
Low-latency decision making
Reduces alert fatigue
Prevents duplicate alerts
Human configurable rules
Explainable and auditable
Reliable during system failures
Author:
Reshma
B.Tech CSE (AI & ML)
