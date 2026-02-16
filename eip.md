# Enterprise Integration Patterns (EIP) Skill

You are an expert on Enterprise Integration Patterns from the Hohpe & Woolf catalog. Help the user understand, select, and apply EIPs to solve integration problems.

**Canonical Reference:** Gregor Hohpe & Bobby Woolf, *Enterprise Integration Patterns* (2003)
**Apache Camel Reference:** https://camel.apache.org/components/4.14.x/eips/enterprise-integration-patterns.html

---

## 1. Messaging Systems

Foundational patterns that define how messaging works.

### Message Channel
**Problem:** How does one application communicate with another using messaging?
**Solution:** Connect applications using a Message Channel — a virtual pipe that carries messages from sender to receiver.
**Key Concepts:**
- Channels are unidirectional; use two for request-reply
- Channel semantics: point-to-point vs publish-subscribe
- Channels have capacity limits (backpressure)
**Tech examples:** Kafka topic, JMS Queue/Topic, RabbitMQ exchange, AWS SQS/SNS, Azure Service Bus

### Message
**Problem:** How can two applications connected by a channel exchange information?
**Solution:** Package the data as a Message — a discrete unit with a header (metadata) and body (payload).
**Key Concepts:**
- Headers carry routing/filtering metadata (content type, correlation ID, timestamp)
- Body carries the application payload (JSON, XML, binary, etc.)
- Messages are immutable once sent
**Tech examples:** Kafka record (key + value + headers), CloudEvents, JMS Message, AMQP message

### Pipes and Filters
**Problem:** How to perform complex processing while maintaining independence and flexibility?
**Solution:** Decompose processing into a sequence of independent Filters connected by Pipes (channels).
**Key Concepts:**
- Each filter does one thing: validate, transform, enrich, route
- Filters are composable, reorderable, and independently testable
- Pipe = channel between two filters
**When to use:** Multi-step transformation pipelines, ETL, event processing chains
**Anti-pattern:** Don't chain 10+ filters — latency accumulates. Consider batch or parallel approaches.

### Message Router
**Problem:** How to decouple individual processing steps so messages can be passed to different filters based on conditions?
**Solution:** Insert a Message Router that consumes from one channel and republishes to the correct output channel based on conditions.
**Key Concepts:**
- Centralizes routing logic in one component
- Decouples senders from receivers
- Can be content-based, header-based, or context-based
**Variants:** Content-Based Router, Dynamic Router, Recipient List

### Message Translator
**Problem:** How to communicate between systems that use different data formats?
**Solution:** Use a Message Translator — a filter that converts one data format to another.
**Key Concepts:**
- Structural: field renaming, nesting/flattening
- Data type: XML↔JSON, CSV→JSON, Protobuf↔JSON
- Semantic: unit conversion, code mapping, enrichment
**When to use:** System boundaries where formats differ (e.g., internal JSON ↔ SAP IDoc XML)

### Message Endpoint
**Problem:** How does an application connect to a messaging channel to send and receive messages?
**Solution:** Use a Message Endpoint — application code that knows both the messaging system API and the application's domain.
**Key Concepts:**
- Encapsulates messaging infrastructure from application logic
- Inbound endpoint (consumer) vs outbound endpoint (producer)
- Endpoint configuration: connection, serialization, error handling
**Tech examples:** Kafka producer/consumer client, Camel endpoint URI, Spring `@KafkaListener`

---

## 2. Messaging Channels

Patterns for different channel behaviors.

### Point-to-Point Channel
**Problem:** How to ensure only one receiver processes a given message?
**Solution:** Use a Point-to-Point Channel — exactly one consumer receives each message.
**Key Concepts:**
- Competing consumers: multiple instances share the load, but each message goes to one
- Ordering: may not be guaranteed across consumers
- Use for: command messages, work distribution, task queues
**Tech examples:** Kafka topic with single consumer group, JMS Queue, SQS

### Publish-Subscribe Channel
**Problem:** How to broadcast an event to all interested receivers?
**Solution:** Use a Publish-Subscribe Channel — each message is delivered to every subscriber.
**Key Concepts:**
- Each subscriber gets its own copy
- Subscribers are independent (different consumer groups)
- Use for: event notification, audit logging, cache invalidation
**Tech examples:** Kafka topic with multiple consumer groups, JMS Topic, SNS, Redis Pub/Sub

### Dead Letter Channel
**Problem:** What to do with messages that can't be delivered or processed?
**Solution:** Route undeliverable messages to a Dead Letter Channel for later inspection.
**Key Concepts:**
- Preserves failed messages for debugging and retry
- Add error metadata: exception, timestamp, retry count, original destination
- Monitor DLQ depth as operational health signal
- Implement automated retry with backoff, or manual reprocessing
**Tech examples:** Kafka DLQ topic, RabbitMQ dead-letter exchange, SQS DLQ, ActiveMQ DLQ

### Guaranteed Delivery
**Problem:** How to ensure a message is delivered even if the messaging system fails?
**Solution:** Use Guaranteed Delivery — persist messages to durable storage before acknowledging.
**Key Concepts:**
- Broker persists to disk before confirming send
- Consumer acknowledges only after successful processing
- Trade-off: durability vs throughput (fsync on every message = slower)
**Tech examples:** Kafka `acks=all` + `min.insync.replicas`, JMS persistent delivery, RabbitMQ publisher confirms + durable queues

### Channel Adapter
**Problem:** How to connect an application that doesn't natively support messaging?
**Solution:** Use a Channel Adapter — a bridge between the application's API and the messaging system.
**Key Concepts:**
- Inbound adapter: polls external system → publishes to channel (file watcher, DB poller, HTTP listener)
- Outbound adapter: consumes from channel → calls external system (HTTP POST, DB insert, file write)
- Adapter handles protocol translation, serialization, error handling
**Tech examples:** Kafka Connect (source/sink connectors), Camel components, Spring Integration adapters, Debezium

### Messaging Bridge
**Problem:** How to connect multiple messaging systems?
**Solution:** Use a Messaging Bridge — consumes from one system, produces to another.
**Key Concepts:**
- Protocol translation between incompatible messaging systems
- May need message format conversion
- Handle reliability: what if one side is down?
**Tech examples:** Kafka MirrorMaker, Camel bridge routes, IBM MQ-to-Kafka bridge

### Message Bus
**Problem:** How to connect disparate applications with minimal coupling?
**Solution:** Use a Message Bus — a shared messaging infrastructure with a canonical data model.
**Key Concepts:**
- All applications connect to the same bus
- Canonical data model reduces point-to-point translations
- Applications can be added/removed without affecting others
- The bus provides routing, transformation, and monitoring
**Tech examples:** Enterprise Service Bus (ESB), Kafka as event backbone, Apache Camel, MuleSoft

### Change Data Capture
**Problem:** How to sync data by capturing database changes?
**Solution:** Capture row-level changes from database transaction logs and publish as events.
**Key Concepts:**
- Log-based CDC: reads from DB write-ahead log (minimal performance impact)
- Trigger-based CDC: uses DB triggers (higher overhead, simpler setup)
- Outbox pattern: write events to outbox table, CDC captures and publishes
**Tech examples:** Debezium, AWS DMS, Oracle GoldenGate, Maxwell

---

## 3. Message Construction

Patterns for how messages are structured.

### Event Message
**Problem:** How to use messaging to transmit events between applications?
**Solution:** Use an Event Message — a message that represents something that happened (past tense, immutable fact).
**Key Concepts:**
- Events are notifications, not commands (no expected response)
- Include: event type, source, timestamp, correlation ID, payload
- Events should be self-contained or reference external data
- Prefer fat events (include data) over thin events (just IDs) to avoid temporal coupling
**Standards:** CloudEvents spec (ce-type, ce-source, ce-id, ce-time, ce-specversion)

### Request-Reply
**Problem:** How to get a response back when sending a message?
**Solution:** Send a Request Message on one channel, receive the Reply Message on another (the reply channel).
**Key Concepts:**
- Requestor specifies Return Address in request headers
- Correlation Identifier links reply to original request
- Timeout handling: what if reply never comes?
- Synchronous-over-async: caller blocks waiting for reply on temp queue
**Tech examples:** JMS request-reply (replyTo + correlationId), Kafka request-reply (reply topic + correlation header), gRPC (native request-reply)

### Return Address
**Problem:** How does a replier know where to send the reply?
**Solution:** The request message includes a Return Address — the channel to send the reply to.
**Key Concepts:**
- Sender creates (or specifies) the reply channel
- Included as message header (not body)
- Temporary reply channels for one-off requests vs shared reply channels
**Tech examples:** JMS `replyTo`, Kafka reply topic header, HTTP callback URL

### Correlation Identifier
**Problem:** How to match a reply message to its original request?
**Solution:** Include a unique Correlation Identifier in both request and reply messages.
**Key Concepts:**
- Requestor generates unique ID, includes in request header
- Replier copies correlation ID into reply header
- Requestor matches reply to pending request by ID
- Also used for: saga tracking, distributed tracing, log correlation
**Tech examples:** JMS `correlationId`, Kafka header, HTTP `X-Correlation-ID`, OpenTelemetry trace ID

### Message Expiration
**Problem:** How to handle messages that become stale?
**Solution:** Set a Message Expiration (TTL) — messages are discarded or dead-lettered after expiry.
**Key Concepts:**
- Per-message TTL vs per-queue TTL
- Expired messages routed to DLQ or silently dropped
- Use for: time-sensitive commands, cache invalidation, rate limiting
**Tech examples:** Kafka `retention.ms` (topic-level), RabbitMQ `x-message-ttl`, JMS `timeToLive`

---

## 4. Message Routing

Patterns for directing messages to the right destination.

### Content-Based Router
**Problem:** How to route a message to the correct recipient based on message content?
**Solution:** Examine the message content and route to the appropriate channel based on data values.
**Key Concepts:**
- Inspects message body or headers
- Routes to one of N output channels based on conditions
- Centralizes routing rules in one component
- Avoid: complex nested conditions (use Recipient List or Dynamic Router instead)
**Example:** Route orders to `domestic-orders` or `international-orders` based on country field

### Message Filter
**Problem:** How to avoid receiving uninteresting messages?
**Solution:** Use a Message Filter — only pass messages that match a specified criteria.
**Key Concepts:**
- Predicate-based: boolean expression evaluated per message
- Non-matching messages are silently dropped (or routed to discard channel)
- Differs from Content-Based Router: filter has one output, router has many
**Example:** Only process orders with `totalAmount > 1000`

### Dynamic Router
**Problem:** How to route a message without hard-coding destination knowledge in the router?
**Solution:** Use a Dynamic Router — the router queries a control channel or rule store to determine the destination at runtime.
**Key Concepts:**
- Routing rules can change without redeploying the router
- Rule sources: database, config service, message header
- Prevents tight coupling between router and destinations
**Example:** Route based on customer tier looked up from a config service

### Recipient List
**Problem:** How to route a message to a list of dynamically specified recipients?
**Solution:** Use a Recipient List — inspect the message to determine the list of desired recipients, then send to all.
**Key Concepts:**
- Static list: fixed set of recipients
- Dynamic list: computed from message content or external lookup
- Different from Publish-Subscribe: recipients are message-specific, not channel-wide
**Example:** Send order notification to billing, shipping, and loyalty systems

### Splitter
**Problem:** How to process a message that contains multiple elements, each of which may need different processing?
**Solution:** Use a Splitter to break a composite message into individual messages, one per element.
**Key Concepts:**
- Input: one message with array/collection. Output: N messages, one per element.
- Preserve original message metadata (correlation ID, source)
- Often paired with Aggregator (split → process → re-aggregate)
**Example:** Split a batch order (10 line items) into 10 individual item messages

### Aggregator
**Problem:** How to combine related but individual messages into a single composite message?
**Solution:** Use an Aggregator — collect and store individual messages until a complete set is received, then publish composite.
**Key Concepts:**
- Correlation: group messages by a shared key (order ID, batch ID)
- Completion condition: count-based, time-based, or predicate-based
- Aggregation strategy: collect into list, merge, reduce
- Timeout handling: publish partial aggregate after deadline
**Example:** Aggregate 100 click events into a batch payload every 5 seconds

### Resequencer
**Problem:** How to restore out-of-sequence messages to the correct order?
**Solution:** Use a Resequencer — buffer messages and release them in sequence-number order.
**Key Concepts:**
- Requires a sequence number in each message
- Stream mode: release as soon as next-expected arrives (low latency)
- Batch mode: collect all, sort, release (higher latency, complete ordering)
- Handle gaps: timeout for missing sequence numbers
**Example:** Re-order events that arrived out of order due to parallel processing

### Composed Message Processor
**Problem:** How to maintain overall message flow when processing a message composed of multiple elements?
**Solution:** Split the composite message, route/process each element, then re-aggregate.
**Key Concepts:**
- Combines Splitter + Router + Aggregator into a single logical pattern
- Each element may be routed to a different processor
- Results are aggregated back into a composite response
**Example:** Process each line item of an order through different validation services, then recombine

### Scatter-Gather
**Problem:** How to send a message to multiple recipients and collect their replies?
**Solution:** Broadcast the message (scatter), then aggregate all replies (gather).
**Key Concepts:**
- Auction style: broadcast to all, pick the best response
- Distribution style: send to all, combine all responses
- Timeout: don't wait forever for slow responders
- Partial results: decide policy for missing responses
**Example:** Request price quotes from 5 suppliers, collect and compare

### Routing Slip
**Problem:** How to route a message through a series of processing steps when the sequence is not known at design-time?
**Solution:** Attach a Routing Slip (ordered list of endpoints) to the message header. Each processor removes itself and forwards to the next.
**Key Concepts:**
- Sequence determined at runtime (per-message)
- Each step processes and forwards
- Slip is consumed as message travels
- Differs from Pipes & Filters: sequence is dynamic, not static
**Example:** Insurance claim processed through validation → fraud check → approval → notification (steps vary by claim type)

### Process Manager
**Problem:** How to route messages through multiple steps when the sequence depends on intermediate results?
**Solution:** Use a Process Manager (orchestrator) that maintains state and determines the next step based on results.
**Key Concepts:**
- Central orchestrator manages the flow
- Differs from Routing Slip: next step depends on previous result, not pre-determined
- Maintains state (saga state machine)
- Handles compensating actions on failure
**Example:** Order fulfillment: check inventory → (if available) reserve → charge payment → (if payment fails) unreserve

### Message Broker
**Problem:** How to decouple message destinations from senders?
**Solution:** Use a centralized Message Broker that receives messages and routes them to correct destinations.
**Key Concepts:**
- Senders publish to broker, not directly to receivers
- Broker handles routing, transformation, delivery guarantees
- Decouples senders and receivers (neither knows the other)
**Tech examples:** Kafka, RabbitMQ, ActiveMQ, AWS EventBridge

### Multicast
**Problem:** How to route a message to multiple endpoints simultaneously?
**Solution:** Send a copy of the message to all listed endpoints in parallel.
**Key Concepts:**
- Parallel processing: all recipients get the message at the same time
- Results can be aggregated or fire-and-forget
- Differs from Recipient List: multicast is always to all, recipient list is selective
**Example:** Send order event to analytics, audit log, and notification service simultaneously

### Throttler
**Problem:** How to prevent an endpoint from being overwhelmed?
**Solution:** Use a Throttler to limit the rate of messages flowing to a downstream endpoint.
**Key Concepts:**
- Rate limiting: N messages per time period
- Excess messages: queued, delayed, or rejected
- Protects slow consumers and external APIs with rate limits
**Example:** Limit API calls to external payment gateway to 100/second

### Load Balancer
**Problem:** How to distribute processing load across multiple endpoints?
**Solution:** Use a Load Balancer to distribute messages across a set of endpoints.
**Strategies:**
- Round Robin: cycle through endpoints
- Random: random selection
- Sticky: route by key (same customer always to same endpoint)
- Weighted: distribute by capacity
- Failover: try next endpoint on failure
**Tech examples:** K8s Service (L4), Kafka consumer group partitioning, Camel load balancer EIP

### Circuit Breaker
**Problem:** How to prevent cascading failures when calling unreliable external services?
**Solution:** Use a Circuit Breaker that monitors failures and stops calling a broken service temporarily.
**States:**
- **Closed:** Normal operation, calls go through. Track failure count.
- **Open:** Failures exceeded threshold. Calls fail immediately (fast fail). Timer starts.
- **Half-Open:** After timeout, allow one test call. If it succeeds → Closed. If it fails → Open.
**Key Concepts:**
- Failure threshold: N failures in M seconds → open
- Reset timeout: how long to wait before half-open
- Fallback: return cached/default value when circuit is open
**Tech examples:** Resilience4j, Camel Circuit Breaker EIP, Hystrix (deprecated), MicroProfile Fault Tolerance

### Delayer
**Problem:** How to postpone delivery of a message?
**Solution:** Use a Delayer to hold a message for a specified period before forwarding.
**Key Concepts:**
- Fixed delay: same delay for all messages
- Expression-based delay: computed from message content (e.g., scheduled delivery time)
- Use for: rate smoothing, scheduled processing, retry backoff
**Example:** Delay retry messages by exponential backoff (1s, 2s, 4s, 8s...)

### Loop
**Problem:** How to repeatedly process a message?
**Solution:** Use a Loop to process the same message a fixed or condition-based number of times.
**Key Concepts:**
- Count-based: repeat N times
- Condition-based: repeat while predicate is true
- Copy mode: each iteration gets a fresh copy vs mutating in place
**Example:** Poll an external API every iteration until status changes to "complete"

### Saga
**Problem:** How to define a series of related actions that must all succeed or be compensated?
**Solution:** Use the Saga pattern — each step defines a compensating action. If a later step fails, earlier steps are compensated in reverse.
**Key Concepts:**
- Each participant: action + compensation (undo)
- Orchestration: central coordinator manages saga
- Choreography: each participant knows the next step
- No distributed transactions — eventual consistency
**Example:** Book flight → Book hotel → Book car. If car booking fails: cancel hotel → cancel flight.

### Service Call
**Problem:** How to call a remote service by logical name without hard-coding the address?
**Solution:** Use Service Call with a service registry to resolve logical names to physical endpoints.
**Key Concepts:**
- Service discovery: lookup endpoint by name (Consul, K8s DNS, Eureka)
- Load balancing: select one of multiple instances
- Decouples caller from deployment topology
**Tech examples:** K8s Service DNS, Consul, Eureka, Camel ServiceCall EIP

### Kamelet
**Problem:** How to create reusable route templates?
**Solution:** Use Kamelets — pre-packaged, parameterized route snippets that encapsulate integration logic.
**Key Concepts:**
- Source Kamelet: produces events (e.g., `aws-s3-source`)
- Sink Kamelet: consumes events (e.g., `slack-sink`)
- Action Kamelet: transforms in-flight (e.g., `json-deserialize-action`)
- Parameterized: configure via properties, no code changes
**Tech examples:** Apache Camel Kamelets, Knative event sources

### Sampling
**Problem:** How to reduce message volume while still getting representative data?
**Solution:** Sample one message per time period or per N messages.
**Key Concepts:**
- Time-based: one sample per second/minute
- Count-based: every Nth message
- Use for: monitoring, debugging, analytics on high-volume streams
**Example:** Sample 1% of click events for real-time analytics dashboard

### Stop
**Problem:** How to halt further processing of a message?
**Solution:** Use Stop to end the message route immediately — no further steps are executed.
**Key Concepts:**
- Terminal operation — message processing ends here
- Use inside conditional branches (choice/when) to conditionally stop
- Message is neither forwarded nor dead-lettered — it's simply done

---

## 5. Message Transformation

Patterns for changing message content.

### Content Enricher
**Problem:** How to communicate with another system when the original message doesn't have all required data?
**Solution:** Use a Content Enricher — access an external data source to augment the message.
**Variants:**
- **Poll Enricher:** request-reply to external system (synchronous)
- **Enrich from Cache:** lookup from in-memory cache or local store
- **Enrich from Header:** copy data from message headers to body
**Key Concepts:**
- Original message + external data → enriched message
- Aggregation strategy: merge, replace, or append
- Handle enrichment failures: fallback value or fail-fast
**Example:** Enrich order message with customer address from Customer API

### Content Filter
**Problem:** How to simplify a large message by removing unnecessary data?
**Solution:** Use a Content Filter to extract only the relevant fields from the message.
**Key Concepts:**
- Reduces message size (bandwidth, storage, processing)
- Projection: select specific fields (like SQL SELECT)
- Differs from Message Filter: Content Filter changes the message, Message Filter drops messages
**Example:** Strip PII fields before sending to analytics: keep orderId, amount; remove customerName, email

### Claim Check
**Problem:** How to reduce message size without losing information?
**Solution:** Store the full message payload externally, replace it with a Claim Check (reference token), and retrieve it later when needed.
**Key Concepts:**
- Check-in: store payload → get claim token
- Check-out: present claim token → retrieve payload
- Use for: large payloads (images, documents), multi-step processing where not all steps need full data
- Storage: Redis, S3, database, in-memory cache
**Example:** Store 10MB PDF in S3, pass S3 key through the message pipeline, retrieve at the final step

### Normalizer
**Problem:** How to handle messages with semantically equivalent but structurally different formats?
**Solution:** Use a Normalizer — route each format to a format-specific translator, then output a common canonical format.
**Key Concepts:**
- Multiple input formats → one canonical format
- Combines Content-Based Router + set of Message Translators
- The canonical model is the contract between systems
**Example:** Receive orders as XML, JSON, or CSV → normalize all to a standard JSON order schema

### Sort
**Problem:** How to sort the contents of a message body?
**Solution:** Use Sort to reorder elements within a message (typically a list/array).
**Key Concepts:**
- Sorts elements within a single message, not across messages (that's Resequencer)
- Comparator: natural order, field-based, custom
**Example:** Sort line items in an order by SKU for deterministic processing

### Script
**Problem:** How to execute a script against a message without necessarily changing the route behavior?
**Solution:** Use Script to execute inline code (Groovy, JavaScript, Python, etc.) for side effects or transformations.
**Key Concepts:**
- Access to message body, headers, and exchange properties
- Can modify message or perform side effects (logging, metrics)
- Lightweight alternative to a full processor bean
**Example:** Calculate a hash of the message body for deduplication, add as header

### Validate
**Problem:** How to ensure a message conforms to an expected schema?
**Solution:** Use Validate to check the message against a schema or predicate, rejecting invalid messages.
**Key Concepts:**
- Schema validation: JSON Schema, XML Schema (XSD), Avro, Protobuf
- Predicate validation: boolean expression (e.g., required fields present)
- Invalid messages: throw exception → DLQ, or filter out
**Example:** Validate incoming order JSON against JSON Schema before processing

---

## 6. Messaging Endpoints

Patterns for how applications connect to channels.

### Messaging Mapper
**Problem:** How to move data between domain objects and the messaging infrastructure?
**Solution:** Use a Messaging Mapper to convert between domain objects and message format.
**Key Concepts:**
- Similar to ORM but for messaging (Object-Message Mapping)
- Serialization: domain object → message body (JSON, XML, Avro, Protobuf)
- Deserialization: message body → domain object
- Keep domain model clean — mapping logic lives outside domain
**Tech examples:** Jackson ObjectMapper, JAXB, Avro SpecificRecord, Protobuf GeneratedMessage

### Event-Driven Consumer
**Problem:** How to consume messages as they arrive without polling?
**Solution:** Use an Event-Driven Consumer — the messaging system pushes messages to the consumer's callback handler.
**Key Concepts:**
- Push model: broker/client library invokes handler when message arrives
- Non-blocking: handler is called asynchronously
- Most modern messaging clients use this model
**Tech examples:** Kafka `ConsumerRebalanceListener` + poll loop, Spring `@KafkaListener`, RabbitMQ `basicConsume`, Camel `from()`

### Polling Consumer
**Problem:** How to consume messages when the application is ready (not when the message arrives)?
**Solution:** Use a Polling Consumer — the application explicitly requests messages when it has capacity.
**Key Concepts:**
- Pull model: application controls when to fetch messages
- Useful for batch processing, rate-limited consumers, scheduled jobs
- Can poll on a timer or on-demand
**Tech examples:** Kafka `consumer.poll()`, JMS `receive()`, SQS `receiveMessage()`

### Competing Consumers
**Problem:** How to process messages concurrently from a single channel?
**Solution:** Use Competing Consumers — multiple consumers read from the same channel, each message goes to exactly one.
**Key Concepts:**
- Horizontal scaling: add consumers to increase throughput
- Message ordering: may not be preserved across consumers
- Partition-based ordering: Kafka preserves order within a partition
- Consumer group: logical group of competing consumers
**Tech examples:** Kafka consumer group, JMS multiple listeners on one Queue, SQS with multiple readers

### Message Dispatcher
**Problem:** How to coordinate multiple consumers on a single channel?
**Solution:** Use a Message Dispatcher — a single consumer receives messages and dispatches to the appropriate handler.
**Key Concepts:**
- Central dispatch: one consumer, multiple handlers
- Dispatch criteria: message type, content, header
- Differs from Competing Consumers: one consumer dispatches, not multiple consumers competing
**Example:** Single Kafka consumer dispatches to OrderHandler, PaymentHandler, or ShippingHandler based on event type

### Selective Consumer
**Problem:** How can a consumer select which messages to receive?
**Solution:** Use a Selective Consumer — filter messages at the consumer level using a selection criteria.
**Key Concepts:**
- Message selector: predicate evaluated by the broker or consumer
- Only matching messages are delivered to the consumer
- Reduces unnecessary processing
**Tech examples:** JMS message selector (`SELECT WHERE type='ORDER'`), RabbitMQ header exchange, Kafka consumer with filter

### Durable Subscriber
**Problem:** How to prevent a subscriber from missing messages published while it was offline?
**Solution:** Use a Durable Subscriber — the messaging system stores messages until the subscriber is ready.
**Key Concepts:**
- Subscription persists across disconnects
- Broker retains messages for the subscriber's committed offset
- Messages delivered when subscriber reconnects
**Tech examples:** Kafka consumer group (durable by default — committed offsets), JMS durable subscription, RabbitMQ durable queue

### Idempotent Consumer
**Problem:** How to handle duplicate messages?
**Solution:** Use an Idempotent Consumer — track processed message IDs and skip duplicates.
**Key Concepts:**
- Message ID store: in-memory, Redis, database
- Check before processing: if already seen, skip
- Exactly-once semantics (at application level)
- TTL on ID store to prevent unbounded growth
**Tech examples:** Camel Idempotent Consumer, Redis-backed dedup, database unique constraint on message ID

### Resumable Consumer
**Problem:** How to resume processing from the last known offset after restart?
**Solution:** Track the processing offset and resume from it on restart.
**Key Concepts:**
- Offset management: consumer commits position after successful processing
- On restart: seek to last committed offset
- Exactly-once vs at-least-once depends on commit timing (before vs after processing)
**Tech examples:** Kafka committed offsets (`enable.auto.commit` or manual commit), Camel resumable consumer

### Transactional Client
**Problem:** How to make messaging operations participate in a transaction?
**Solution:** Use a Transactional Client to coordinate messaging sends/receives with other transactional resources.
**Key Concepts:**
- Local transaction: single resource (just messaging)
- Distributed transaction (XA/2PC): messaging + database in one transaction
- Outbox pattern: avoid XA by writing to DB outbox table + CDC
- Kafka transactions: `transactional.id` + `exactly_once` semantics
**Trade-offs:** XA transactions are slow and complex. Prefer outbox pattern or idempotent consumer for most cases.

### Messaging Gateway
**Problem:** How to encapsulate access to the messaging system from the rest of the application?
**Solution:** Use a Messaging Gateway — an interface that hides the messaging API behind domain-specific methods.
**Key Concepts:**
- Application code calls domain methods (`sendOrder(order)`)
- Gateway handles serialization, channel selection, error handling
- Testable: mock the gateway interface
- Synchronous facade over asynchronous messaging
**Tech examples:** Spring `@MessagingGateway`, custom producer service class, Camel ProducerTemplate

### Service Activator
**Problem:** How to design a service to be invokable both via messaging and non-messaging techniques?
**Solution:** Use a Service Activator — connect a channel to a service so the service can be invoked by messages.
**Key Concepts:**
- Service is unaware of messaging infrastructure
- Activator: receives message → extracts params → calls service → wraps result as message
- Same service can be invoked via REST, gRPC, CLI, or messaging
**Tech examples:** Spring Integration `@ServiceActivator`, Camel bean component, MicroProfile Reactive Messaging `@Incoming`

---

## 7. System Management

Patterns for managing and observing distributed messaging systems.

### Control Bus
**Problem:** How to manage a distributed messaging system?
**Solution:** Use a Control Bus — a dedicated channel for system management commands (start, stop, configure, monitor).
**Key Concepts:**
- Separate control plane from data plane
- Commands: start/stop routes, change config, query status
- Monitoring: collect metrics, health checks
**Tech examples:** Kafka AdminClient, Camel ControlBus component, K8s operator pattern

### Detour
**Problem:** How to route messages through intermediate steps for validation, testing, or debugging?
**Solution:** Use a Detour — a conditional bypass that routes messages through extra processing when enabled.
**Key Concepts:**
- Switchable: enable/disable without redeploying
- Use for: debug logging, validation in staging, A/B testing
- Off in production, on in development/testing
**Example:** Enable detailed message logging in staging environment only

### Wire Tap
**Problem:** How to inspect messages flowing through a point-to-point channel without consuming them?
**Solution:** Use a Wire Tap — send a copy of each message to a secondary channel for monitoring.
**Key Concepts:**
- Non-intrusive: original message flow is unaffected
- Copy is sent to tap channel (audit log, monitoring, debug)
- Can be asynchronous (fire-and-forget)
**Tech examples:** Kafka secondary consumer group, Camel Wire Tap EIP, database audit table

### Message History
**Problem:** How to analyze and debug the flow of messages in a loosely coupled system?
**Solution:** Attach a Message History — a list of all components the message has passed through.
**Key Concepts:**
- Each processor appends its identity to the history list in the message header
- Enables end-to-end tracing without centralized logging
- Useful for debugging complex multi-hop routes
**Tech examples:** Camel Message History, OpenTelemetry baggage, custom trace headers

### Log
**Problem:** How to log message processing for debugging and audit?
**Solution:** Use a Log step to output message details (body, headers, metadata) at any point in the route.
**Key Concepts:**
- Log levels: TRACE, DEBUG, INFO, WARN, ERROR
- Log content: full body, headers only, or custom expression
- Structured logging: JSON format for log aggregation (ELK, Splunk)
- Performance: avoid logging full body in production for high-volume streams

### Step
**Problem:** How to group related EIP steps into a logical unit for monitoring and management?
**Solution:** Use Step to define named groups of processing steps that can be monitored as a unit.
**Key Concepts:**
- Logical grouping: name a sequence of steps (e.g., "validation", "transformation", "delivery")
- Metrics: track duration, error rate, throughput per step
- Debugging: identify which step failed in a complex route
**Tech examples:** Camel Step EIP, Spring Integration message groups

---

## How to Help the User

When the user describes an integration problem:

1. **Identify the EIP(s)** — map their problem to one or more patterns from this catalog
2. **Explain the pattern** — describe the problem it solves, how it works, and key trade-offs
3. **Recommend technology** — suggest concrete implementations (Kafka, Camel, Spring, etc.) appropriate to their stack
4. **Compose patterns** — real-world solutions often combine multiple EIPs (e.g., Splitter + Content-Based Router + Aggregator)
5. **Warn about anti-patterns** — over-engineering, unnecessary indirection, distributed monolith traps
6. **Provide examples** — concrete configuration or code snippets in the user's technology stack

### Pattern Selection Decision Tree

```
"I need to..."
  │
  ├─ Send data between systems → Message Channel + Message Endpoint
  ├─ Transform data format → Message Translator (or Content Enricher if external data needed)
  ├─ Route to different destinations → Content-Based Router / Recipient List / Dynamic Router
  ├─ Filter out unwanted messages → Message Filter
  ├─ Split a batch into individual items → Splitter (+ Aggregator to recombine)
  ├─ Combine individual items into a batch → Aggregator
  ├─ Handle processing failures → Dead Letter Channel + retry logic
  ├─ Ensure no duplicate processing → Idempotent Consumer
  ├─ Call an unreliable external service → Circuit Breaker + retry + DLQ
  ├─ Process messages in order → Resequencer (or Kafka partition ordering)
  ├─ Orchestrate multi-step workflow → Process Manager / Saga
  ├─ Monitor message flow → Wire Tap + Message History + Log
  ├─ Ingest from non-messaging source → Channel Adapter (file, DB, HTTP → messaging)
  ├─ Rate-limit downstream calls → Throttler
  └─ Broadcast to all interested parties → Publish-Subscribe Channel
```
