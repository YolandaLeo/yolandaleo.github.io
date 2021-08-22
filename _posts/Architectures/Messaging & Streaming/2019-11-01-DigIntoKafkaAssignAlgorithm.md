---
layout: post
categories: Architecture
tag: Kafka
title: Dig into Kafka Assignment Algorithm
---
As I accidentally ran into the issues related to Kafka assignment algorithm when adding new consumers to a certain consumer group. I checked Kafka's documentation and here logging my understandings here, hope to help other guys understand the algorithm and choose the right one for your application (or write your own).
<!--more-->
## Range (org.apache.kafka.clients.consumer.RangeAssignor)
> Range partitioning works on a per-topic basis. For each topic, we lay out the available partitions in numeric order and the consumer threads in lexicographic order. We then divide the number of partitions by the total number of consumer streams (threads) to determine the number of partitions to assign to each consumer. If it does not evenly divide, then the first few consumers will have one extra partition.

Say, there's 3 consumers named C0, C1, C2, and 3 topics T0, T1, T2 each is devided into 4 partitions P0, P1, P2, P3.
As the Range works on **per-topic basis**, the distribution is shown here:
<img src="{{ site.baseurl }}/img/range.png" alt="range" class="inline"/>


## Round Robin (org.apache.kafka.clients.consumer.RoundRobinAssignor)
>The roundrobin assignor lays out all the available partitions and all the available consumers. It then proceeds to do a roundrobin assignment from partition to consumer. If the subscriptions of all consumer instances are identical, then the partitions will be uniformly distributed. (i.e., the partition ownership counts will be within a delta of exactly one across all consumers.) For example, suppose there are two consumers C0 and C1, two topics t0 and t1, and each topic has 3 partitions, resulting in partitions t0p0, t0p1, t0p2, t1p0, t1p1, and t1p2. The assignment will be: C0: [t0p0, t0p2, t1p1] C1: [t0p1, t1p0, t1p2]

With round-robin assigner, partitions among different topics will be more evenly distributed to consumers for the former use case. 
I've seen somewhere in kafka document:
>[…] we lay out the available partitions in numeric order and the consumer threads in lexicographic order […]

But to be more detailed, kafka sort the consumer threads hashed to help the ordering more even.