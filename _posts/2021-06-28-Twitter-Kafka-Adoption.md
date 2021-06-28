---
layout: post
title:  "Kafka Summit 2021-Europe: Kafka Adoption at Twitter"
author: Tung 
categories: [ summary, Kafka, KafkaSummit ]
featured: true
image: assets/images/TwitterKafkaAdoption.png
---
## 1. Migration Challenges

- Huge scale of Migration with thousands of topics and ten of thousands of consumers
- Require cooperation from hundreds of team
- Mission Critical as it impacts 

## 2. Overall Architecture

### 2.1 Before migration

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/BeforeMigrationTwitter.png"/></p>

### 2.2 After Migration

<p align="center">
  <img src="{{ site.baseurl }}/assets/images/AfterMigrationTwitter.png"/></p>

- Scale of Kafka cluster in Twitter as of March 2021

  - Cluster

    - On-premise, bare-metal powered by Aurora Mesos
    - Plan to move to K8s on Cloud and on-premise

  - Broker Config

    - 4TB/ 8TB SSD will also support 24 TB SSD soon
    - JVM heap 5GB to 15GB, pagecache around 45GB currently

  - Scale

    - 50 Cluster, up to 300 brokers per cluster
    - More than 2000 topics , > 40k subscriber
    - 120M EPS , 120G BPS Ingress, 900G BPS Egress
    - Some topics has >3G ingress

  - Version

    - Broker 2.5
    - Client/ Stream 2.4.1


## 3. Existing Problem and Challenges

  - High latency
    - To keep batch efficiency and high throughput , Twitter uses [KIP-480](https://cwiki.apache.org/confluence/display/KAFKA/KIP-480%3A+Sticky+Partitioner) : Sticky Partitioner
  - Huge lag from Rebalancing
    - Twitter uses [KIP-429](https://cwiki.apache.org/confluence/display/KAFKA/KIP-429%3A+Kafka+Consumer+Incremental+Rebalance+Protocol): Incremental Rebalancing
  - Disk Failure and unclean leader election
    - in Kafka 2.5 it happens with lesser frequency ( suspected due to replica.lag.time.max.ms = 30)
    - [KIP-501](https://cwiki.apache.org/confluence/display/KAFKA/KIP-501+Avoid+out-of-sync+or+offline+partitions+when+follower+fetch+requests+are+not+processed+in+time)
  - Adoption of Kafka Stream
    - Open Source Finatra: Native Kafka Stream Integration
      - Allow fast creation of fully fuctional KafkaStream Service
      - Support custom DSL and async processor and transformer
      - Rich unit testing functionality
      - RocksDB integration supporting efficient range scan
    - Limitation of Finatra:
      - Scalability: The StreamAssignor group message size limit the maximum size of group member (fixed in 2.5)
      - Stateful: Ensure the streamThread on the same machine is assigned the same partition as before (KIP345 help partially)

[Kafka Adoption at Twitter](https://www.confluent.io/events/kafka-summit-europe-2021/twitters-apache-kafka-adoption-journey/)
