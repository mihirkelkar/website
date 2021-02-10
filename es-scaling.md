---
layout: default
---

# Scaling Elastic Search

### Lessons from building a high throughput and fast elastic search cluster



Elastic search is a powerful *out-of-the-box* search product. It's built to scale horizontally, so adding more nodes to the cluster lets it handle more data without performance degradation. Despite the simplistic sounding approach, scaling a cluster under real production loads is a complicated problem that needs to be tailored to your specific use case.

In fast-growing businesses, elastic search clusters need significant fine-tuning to keep up with growth in data size, traffic volume, and changes in data access patterns. Simply adding nodes without understanding your data access patterns can blow up your cloud computing bills and customer experience.

Traffic growth typically impacts your cluster in two significant ways:

- **Speed Degradation**

Impacts your cluster's ability to respond to requests quickly

- **Throughput Degradation**

Impacts your cluster's ability to handle multiple requests at the same time

Before we dive into what we can do to scale well, let's develop some baseline understanding of how elastic search works. Elastic-search stores your data in an index; an index can be thought of as a collection of documents. Each document is a collection of fields, which are the key-value pairs that contain your data. A document can represent a customer or a product, and its fields can be the attributes associated with your customer/product.

![ScreenShot](/assets/one.png)

An index is divided into multiple shards. Each shard contains a subset of documents from the index. An index's shards are distributed across several nodes of the elastic search cluster. This way, each node only contains a part of an index. Each shard can also have replicas (as decided by the replication factor setting). A shard's replicas are distributed across the cluster's nodes. All of this helps protect your data against node failures and allows for concurrent data access.

<img src="https://imgur.com/MJJHgGn" width="600" height="600">

***What happens when you run an elastic search query?***

A query can be received by any node in the elastic search cluster. The node that receives that request becomes the coordinator node. The coordinator determines a target index and then performs the search on its shards and asks the other nodes to search on their respective shards for the target index. Each shard returns its matching documents to the coordinator node, where the list is merged, sorted, or ranked and returned as a query response.

<img src="https://imgur.com/IWnYZg9" width="600" height="600">

<img src="https://imgur.com/Zpb3Z3E" width="600" height="600">

**Improving Query Speed and throughput**

On a conceptual level, we can improve elastic search query performance by

1. Reducing the amount of time it takes to search one shard
2. Being able to run the search query simultaneously on multiple shards across multiple nodes
3. Reducing the time it takes to aggregate search results on the co-ordinating node

We can reduce the amount of time it takes to search a single shard by reducing the number of documents in it. Splitting up an index into multiple shards can help with that. Having an index's shards split across multiple nodes is also useful since each node can run a parallel query on its shards. Generally, we don't want the number of nodes to be more than the number of shards and replicas. For example, if you have a single index cluster with 3 shards and 1 replica each, you only need 6 nodes at the maximum. Anything above that would not be a prudent use of computing power since there are not enough shards to be allotted to a node. So if we intend to add more nodes to a cluster, we should consider creating more shard replicas to allocate to the nodes. This will help speed up performance linearly.

The number of searches a node can run in parallel is determined by the node's thread pool size, controlled through the node's settings. Adjusting the thread pool size can get complicated since performance gains can be somewhat dependent on the type of hardware your node is running on. In general, increasing the thread pool size does not always guarantee that the node can run more queries in parallel. In-fact, increasing your thread pool size over a specific limit might become counterproductive and lead to performance degradation. Thread pool size should be regularly monitored if you are making changes to it.

The aggregation phase for a query doesn't usually offer a lot of room for optimization unless you are aggregating a vast number of matching documents. The aggregation phase is generally swift.

From an application perspective, you can improve query performance by

***Grouping your data to match your data access patterns***

If a subset of your index's documents is being queried most of the time, it might make sense to split your index into several indices depending on the attributes it's being queried on.

For example, if your search index contains data on books and a majority of your users are searching based on the categories of these books, it might make sense to have an index per category. This will mean that queries can be directed to specific indexes, and a smaller set of documents will be searched.


<img src="https://imgur.com/viZISg0">

***Use filtering over scoring***

Elastic search can perform two kinds of searches

*a) score* *b) filter*

A score query tries to provide a relevant score for every document depending on how closely it matches the search criteria. Scoring every document is CPU intensive and can affect performance.

On the other hand, a filter query simply selects or discards a document depending on if it rigidly matches or doesn't match the search criteria. There is no scoring involved. This makes a filter query very fast and performant. If we can model our documents depending on the data access pattern, we should restrict all queries to be of the filter type.
