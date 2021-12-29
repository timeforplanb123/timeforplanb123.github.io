---
layout: post
title: Elastic stack. Part 1 
summary: Overview of the Elastic Stack.
featured-img:
categories: Linux Networking
tags: [ elastic, notes, linux ]
---
## Overview of the Elastic Stack

Elastic stack or ELK - stack from free software such as Elasticsearch, Logstash and Kibana. The full stack includes X-Pack and Beats components.

Elasticsearch performs the functions of data storage, cluster management, document indexing, balancing and routing of service and search queries.

Logstash provides data processing for transmission to Elasticsearch. The input data here is syslog, kafka, http and others, which can be filtered using various plugins (for example, csv, xml, json), and then, using output-plugins, given to Elasticsearch, Kafka, Email, any http. For example, a classic syslog can be parsed using Logstash into fields, converted to json and given to Elasticsearch or Kafka. And it will be a fairly simple task.

Kibana is a visualization platform, using the REST API, works with Elasticsearch and helps manage Elasticsearch and visualize data in the form of dashboards.

X-Pack is a set of additional features (some of which are not included in the basic, free license). For example, security (authentication, authorization via Kibana interface), user privilege control, ELK monitoring (CPU, memory usage, disk space, etc.), alerting (for any triggers), notifications (slack, zabbix, etc.), data export in csv and other formats, forecasting (load scaling), Graph (data interaction analysis), Elasticsearch SQL (Elastic query DSL for SQL).

Beats - data collector:
- Filebeat (log files, includes modules for mysql, nginx)
- Metricbeat (system and service metrics, memory and CPU, includes modules for nginx, SQL, etc.)
- Packetbeat (collects network data, http requests or database transactions)
- Winlogbeat (collects Windows Events Logs)
- Auditbeat (collects audit data from Linux)
- Heartbeat (monitor system uptime)

The interaction between the components looks like this:

**BEATS or LOGSTASH <-> ELASTICSEARCH <-> KIBANA X-PACK**

Elasticsearch can receive data directly, without Logstash or Beats. This function is performed by the Elasticsearch Ingest node. But, if there is a lot of data, data formats are different, then processing with Logstash or Beats is needed. With the any modules you can receive, filter and output different data formats.

## Sharding and Scalability

The index is where documents are stored in Elasticsearch.
The index consists of shards. Sharding is a way to split indices into small pieces. By default, in version 7 of Elasticsearch, 1 index = 1 shard. In versions below 7, one index was divided into 5 shards, which could trigger to an over-sharding problem. What are the limitations here? The number of documents in one shard cannot be more than 2 million, so there is no need to add all the data to one index, it is worth thinking in advance about the structure of data indices.

So, speaking of sharding:
- sharding is a way to divide the index into pieces. Each such piece is called a shard
- sharding is performed at the index level. One index can store 2 million documents
- the main purpose of shards is horizontal data scaling
- shard is an independent index, a piece of a large index
- shard is Apache Lucene Index
- shard does not have a certain size, but it grows with the growth of the number of documents
- shard stores about 2 million documents
- shard helps to parallelize queries, increasing the speed of working with the index
- REST API is used to view indices and shards, for example, `GET _cat/indexes?v` and `GET _cat/shards?v`
- indices are usually created automatically. See [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html#index-creation){:target="_blank"}

Good explanation of sharding is [here](https://stackoverflow.com/questions/15694724/shards-and-replicas-in-elasticsearch){:target="_blank"}

## Understanding replication

- replication is about copying of shards. Replication works at index-level
- the shard, that has been replicated, is called primary shard
- replica shard - a complete copy of the shard
- primary shard + replica shard = replication group
- replica shard can respond to search request, just like the primary shard
- the number of replicas is configured, when creating the index

Example of using a replica:
there is a cluster of 2 Elasticsearch nodes. There are 2 indices (2 shards). The first index is stored on the first node, the second index is stored on the second node. Then the replica of the first index will be stored on the second node, and the replica of the second index will be stored on the first node. In this case, you can save all data by losing one node.

Replication on single node:
- increases performance with CPU parallelization
- increases the simultaneous number of requests to shards in the replication group

Elasticsearch chooses the route to the primary and replica shards independently. If we store rarely changing data, then replicas on one node will increase performance, but if the data is updated frequently, then the replica will be as a backup and will have no effect on performance.

By default, each new index creates a replica (one index = one shard with + one replica shard). It's actually for Elasticsearch 7.x

[Good article about Elasticsearch Replication](https://codingexplained.com/coding/elasticsearch/understanding-replication-in-elasticsearch){:target="_blank"}

## Elasticsearch cluster. Overview of node roles

1. node.master

the master node is responsible for all management actions in the Elasticsearch cluster:
- creating and deleting indexes
- shards routing
- stores Elasticsearch cluster topology

There may be several master nodes in Elasticsearch cluster, then they form a quorum.

2. node.data

- stores data
- stored data includes executed search requests

3. node.ingest

- simplified logstash functionality
- performs part of the tasks of indexing documents

If there is a logstash, then ingest.node is usually not used.

4. coordination node

- it's placed separately from ther Elasticsearch node roles
- communicates with the master.node
- responsible for the distribution of service and ssearch requests
- works as load balancer

5. node.ml

- enables or disables the Machine Learning API for the node
- useful for ML jobs without affecting other tasks

6. node.voting_only

- waiting to become a new master.node
- relevant for large Elasticsearch clusters

By default, the node from a Elasticsearch cluster, has a data + ingest + master `dim` roles. Why do we change this roles? Because Elasticsearch cluster is usually required to provide fault toulerance and scalability. And various node roles can solve our task.

[Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html){:target="_blank"}

## Elasticsearch cluster example

Let's build such an Elasticsearch cluster and start it in the next part of this article

![]({{ site.url }}{{ site.baseurl }}/assets/img/posts/elastic_stack/elastic_cluster.png)
