---
layout: post
categories: Architecture
tag: ElasticSearch
title: Elastic Search Basics
---
This is the key notes when I was exploring Elastic Search.
<!--more-->
# Overview
## Concepts
2004 - top priority: scalability
**E** lastic search: search engine
**L** ogstash: pipeline
**K** ibana : dashboard
**B** eats : data shipper
Is distributed, RESTful search & real time analysis
Is NOT designed for data storage, not RDBMS or KV, used as single source of truth data (no transaction)

Node - a java process
Master, Data, Coordinating
Index - logical way of grouping data
Data schema / a mechanism of distributing data in cluster

## CRUD
It is strongly recommended in industry to manually create mapping, for two reasons:
* infinite metadata is disaster
* ES automatically generated field type may not be the one wanted

```JSON
PUT melv_comments/_doc/{_id}
POST melv_comments/_doc
{
        "_index": "melv_comments",
        "_type": "_doc",
        "_id": "1aaa",
        "_score": 1,
        "_source": {
          "username": "melv",
          "comment": "update rating",
          "rating": 1.5
        }
      },
      {
        "_index": "melv_comments",
        "_type": "_doc",
        "_id": "HtJSEG4BKrdniNM2Onry",
        "_score": 1,
        "_source": {
          "username": "melv",
          "comment": "No explicit id",
          "rating": 2
        }
      }
```
First way with specific id will be delt with as UPSERT, using for deduplication and routing key.
Second way with ES generated hash key, is recommended for auto sharding.

Index API overwrite
User _update to update certain fields
**There’s no in-place update in ES**
**DELETE should be cautious… no rollback supported**

## Shards: Distribution of an Index
Documents are routed by: Shard = hash(_routing) % number of primary shards
Default _routing is document id
Primary vs Replica
Primary: original shards of an index - write throughput, reallocation
Replicas: copies of the primary - HA, read throughput

## Segment
* Basic storage unit, a Lucene inverse index
* Shards are actually stored in segments
* Why ES does not like update / delete
  * No roll back
  * Performance impacted: segments become sparse & merge is IO intensive
  * Avoid frequent update/delete, and use *POST {index}/_forcemerge* when load is low

aggr - keyword
full text index - text

pagination query
- scroll: make a snapshot, return in a shorter time
- from & to & size: scan full index, can search newly added documents



POST /user/_search
{
    "query":{
        "bool":{
            "must":{"match" {"university":"Georgia"}},
            "should":{
                "bool":{
                    "must_not":{
                        "match":{"name":"jack"}
                    }
                }
            }
        }
    },
    "sort":{"_score":"asc"}
}
