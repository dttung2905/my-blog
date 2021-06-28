---
layout: post
title:  "Kafka Summit 2021-Europe: Advanced CDC Pattern with Debezium"
author: Tung 
categories: [ summary, Kafka, KafkaSummit ]
featured: true
image: assets/images/CDCAdvancedPattern.png
---

## 1. Outbox Pattern

### 1.1 Why do we need this pattern?

- How to reliably/ atomically update the database and publish messages/ events?
- The hardest part in microservices is data. Microservices don't exist in isolation and very often they need to propagate data and data changes within each other
- Example:
  - A microservice that manages **purchase order** i.e when a new order is placed, information about that order may have to be relayed to **shipment services** (so it can assemble shipments of one or more orders) and a **customer service** ( so it can update things like the customer's total credit balance based on the new order)
  - **First solution**: **purchase order** service will invoke some REST, grpc or other synchronous API provided by these services to **shipment services** and **customer service**
  - **Problem**: 
    - One service cannot function without the other services which it invokes. Eg: When a purchase order is placed, the other service might need to obtain the information how many times the purchase item is on stock from an inventory service
    - Lack of replayability i.e. the possibility for new consumers to arrive after events have been sent and still be able to consume the entire event stream from beginning
  - Second Solution: Use apache Kafka to notify other downstream services i.e shipment services and customer services
  - Problem:
    - Dual Write: purchase order service has to do 2 things: `INSERT` into its own database in the `PurchaseOrder` AND send to Kafka
    - what if the latter failed ? we will have a new purchase order persisted in the local database, but not having the corresponding message

### 1.2 How does this pattern solve our problem?

- Instead of making dual write to both Kafka and the database for the purchase order services. The purchase order service will only make a single write to `PurchaseOrder` table and another table called `outbox` table
- Debezium will capture the changes from the `outbox` table, populate it into Kafka. Downstream application will receive new events from the Kafka Connect Sink Connector

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/OutboxPattern.png"/></p>

- The schema of the `outbox` table can be found below

```bash
Column        |          Type          | Modifiers
--------------+------------------------+-----------
id            | uuid                   | not null
aggregatetype | character varying(255) | not null
aggregateid   | character varying(255) | not null
type          | character varying(255) | not null
payload       | jsonb                  | not null
```

- Its columns are these:

  - `id`: unique id of each message; can be used by consumers to detect any duplicate events, e.g. when restarting to read messages after a failure. Generated when creating a new event.

  - `aggregatetype`: the type of the *aggregate root* to which a given event is related; the idea being, leaning on the same concept of domain-driven design, that exported events should refer to an aggregate (["a cluster of domain objects that can be treated as a single unit"](https://martinfowler.com/bliki/DDD_Aggregate.html)), where the aggregate root provides the sole entry point for accessing any of the entities within the aggregate. This could for instance be "purchase order" or "customer".

    This value will be used to route events to corresponding topics in Kafka, so there’d be a topic for all events related to purchase orders, one topic for all customer-related events etc. Note that also events pertaining to a child entity contained within one such aggregate should use that same type. So e.g. an event representing the cancelation of an individual order line (which is part of the purchase order aggregate) should also use the type of its aggregate root, "order", ensuring that also this event will go into the "order" Kafka topic.

  - `aggregateid`: the id of the aggregate root that is affected by a given event; this could for instance be the id of a purchase order or a customer id; Similar to the aggregate type, events pertaining to a sub-entity contained within an aggregate should use the id of the containing aggregate root, e.g. the purchase order id for an order line cancelation event. This id will be used as the key for Kafka messages later on. That way, all events pertaining to one aggregate root or any of its contained sub-entities will go into the same partition of that Kafka topic, which ensures that consumers of that topic will consume all the events related to one and the same aggregate in the exact order as they were produced.

  - `type`: the type of event, e.g. "Order Created" or "Order Line Canceled". Allows consumers to trigger suitable event handlers.

  - `payload`: a JSON structure with the actual event contents, e.g. containing a purchase order, information about the purchaser, contained order lines, their price etc.

- At the moment of writing this document, Debezium has created a built-in feature for [Outbox Event Router SMT](https://debezium.io/documentation/reference/0.9/configuration/outbox-event-router.html). You just have to specify your `outbox` table name and Debezium will help to transform and route the message in the table to suitable kafka topic
- You can read more about this in a blog post [here](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/)

## 2. Strangler Fig Pattern

### 2.1 Why do we need this pattern? 

- This was mentioned in a 2004 article on his Martin Folwer's [website](https://martinfowler.com/bliki/StranglerFigApplication.html). It is usally used to handle big system migration, especially from monolithic to microservices
- It is based on an analogy to a vine that strangles a tree that it's wrapped around and gradually take over the host
- Benefits
  - Incremental migration
  - Can pause or stop migration at any time
  - Migration steps ideally reversible
- The key here is minimize risk

### 2.2 How does it solve our problem

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/StranglerFigPattern.png"/></p>

- There are several way to create this architectural diagram
- One way is to migrate the Customer Service, set up Debezium to capture changes from the legacy system to the new database. Depending on our routing layer's capabilities, we can use techniques such as dark launching, parallel runs, and canary releasing to reduce or remove the risk of rolling out the new services

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/ReadNewServices.png"/></p>

- As show in the picture above, what we can also do here is to only direct read requests to our service initially, while continue to send the writes to the legacy system.
- When we see that the read operations are going through without issues, we can then direct the write traffic to the new service. At this point, if we still need the legacy application to operate for whatever reason, we will need to stream changes from the new services toward the legacy application database
- Next , we want to stop any write or mutating activity in the legacy module and stop the data replication from it

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/ReadOldServices.png"/></p>

- You can read more about that in the RedHat blog [here](https://developers.redhat.com/articles/2021/06/14/application-modernization-patterns-apache-kafka-debezium-and-kubernetes#after_the_migration__modernization_challenges) or Shopify Migration blog [here](https://shopify.engineering/refactoring-legacy-code-strangler-fig-pattern)

## 3. Saga Pattern

### 3.1 Why do we need this pattern?

- Outbox pattern might solve the simpler inter-service communication problem. It is not sufficient alone for solving the more complex long-running distributed business transaction use case
- Two phase commit is really not an option

### 3.2 How does it solve our problem

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/SagaPattern.png"/></p>

- let’s consider the example of an e-commerce business with three services: order, customer, and payment. When a new purchase order is submitted to the order service, the following flow should be executed—including the other two services:
- First, we need to check with Customer Service whether order is under the customer's credit limit. If a customer has a credit limit of $500, and a new order with value of $300 comes in, it would be ok and the customer have a remaining limit of $200. A subsequent order with a value of $250 would definitely be rejected

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/SagaPatternFlow.jpg"/></p>

- With the outbox pattern in our toolbox, things become a bit clearer. The order service, acting as the Saga coordinator, triggers the entire flow after an incoming order placement call (typically via a REST API), by updating its local state—comprising the persisted order model and the Saga execution log—and emits messages to the other two participating services, one after another.
- Each service emits outgoing messages via the outbox table in its own database. From there, the messages are captured via Debezium and sent to Kafka, and finally consumed by the receiving service. Upon sending and receiving messages, the order service, acting as the orchestrator, also persists the Saga progress in a local state table. (More on that below.) Furthermore, all participants log the ids of the messages they’ve consumed in a journal table, to identify potential duplicates later on

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/SagaPatternFlow.jpg"/></p>

- Now, what happens if one step of the flow is failing? Let’s assume the payment step fails because the customer’s credit card has expired. In that case, the previously reserved credit amount in the customer service needs to be released again. To do so, the order service sends a compensation request to the customer service. Zooming out a bit (as the details around Debezium and Kafka are the same as before), this is what the message exchange would look like in this case:

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/FailedSagaFlow.jpg"/></p>

- This table fulfills the role of the Saga Log. Its columns are these:
  - `id`: Unique identifier of a given Saga instance, representing the creation of one particular purchase order
  - `currentStep`: The step at which the Saga currently is, e.g., “credit-approval” or “payment”
  - `payload`: An arbitrary data structure associated with a particular Saga instance, e.g., containing the id of the corresponding purchase order and other information useful during the Saga lifecycle; while the example implementation uses JSON as the payload format, one could also think of using other formats, for instance, [Apache Avro](https://avro.apache.org/), with payload schemas stored in a schema registry
  - `status`: The current status of the Saga; one of `STARTED, SUCCEEDED, ABORTING`, or `ABORTED`
  - `stepState`: A stringified JSON structure describing the status of the individual steps, e.g., `"{\"credit-approval\":\"SUCCEEDED\",\"payment\":\"STARTED\"}"`
  - `type`: A nominal type of a Saga, e.g., “order-placement”; useful to tell apart different kinds of Sagas supported by one system
  - `version`: An optimistic locking version, used to detect and reject concurrent updates to one Saga instance (in which case the message triggering the failing update needs to be retried, reloading the current state from the Saga log)

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/OutboxSagaTable.jpg"/></p>

- You can find a complete proof-of-concept implementation of this architecture in the Debezium [examples repository](https://github.com/debezium/debezium-examples/tree/master/saga) on GitHub. The key parts of the architecture are these:
  - The three services, [order](https://github.com/debezium/debezium-examples/tree/master/saga/order-service) (for managing purchase orders and acting as the Saga orchestrator), [customer](https://github.com/debezium/debezium-examples/tree/master/saga/customer-service) (for managing the customer’s credit limit), and [payment](https://github.com/debezium/debezium-examples/tree/master/saga/payment-service) (for handling credit card payments), each with their own local database (Postgres)
  - Apache Kafka as the messaging backbone
  - Debezium, running on top of Kafka Connect, subscribing to changes in the three different databases, and sending them to corresponding Kafka topics, using Debezium’s [outbox event routing](https://debezium.io/documentation/reference/configuration/outbox-event-router.html) component

- You can also read a detailed post by Redhat [here](https://developers.redhat.com/articles/2021/06/14/application-modernization-patterns-apache-kafka-debezium-and-kubernetes#after_the_migration__modernization_challenges) , the Saga Pattern in generic term [here](https://microservices.io/patterns/data/saga.html)

[Debezium Advanced Pattern](https://www.confluent.io/events/kafka-summit-europe-2021/advanced-change-data-streaming-patterns-in-distributed-systems/)
