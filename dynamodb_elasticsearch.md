# Using DynamoDB + Elasticsearch for OLTP + OLAP

## Theory
DynamoDB is a scalable, performant, cost-effective, low maintenance database solution.  Too good to be true, right?  As always, there are tradeoffs.  NoSQL and especially DynamoDB have very limited query options and require good design up front to make it work effectively for your application.  DynamoDB is built specifically for transactional workloads (OLTP), but most applications also need some analytics/reports (OLAP) or search capabilities.  Pairing the two database technologies helps get the best of both worlds.

Why not just use relational/RDS/Aurora?  Relational is sort of the middle of the road approach that is pretty flexible and can handle both, but not as efficiently.  To do partial text search over several columns would require chained `LIKE` statements which isn't ideal.  In this case I am proposing a design for a low scale but complex app.  However, DynamoDB and Elasticsearch can both scale horizontally very efficiently and thus can be used in web scale applications.

This solution is also more cost effective.  When building a microservices based application, databases should not be shared between microservices to ensure they are not coupled from change management or availability perspectives.  An application with 5 data services, 3 node clusters, and 4 environments is a total of 60 database servers!  On the other hand, an equivalent app using this pattern would have just 11 servers - much more cost effective!  This is because the Elasticsearch is not the transactional system of record and can be regenerated at any time from the system of record.

## Replication Code

## Elasticsearch Sizing - optimize for small data volume since the cluster is scoped to a single application
- Nodes
  - 2 data nodes
    - must be even number for AZ balancing feature
    - AZ balancing ensures replica shards are in different AZs than their respective primary shards.
  - 3 master nodes
    - only needed for production environment
    - 3 nodes ensures quorum even if a node is lost to prevent split brain issues
- Indexes
  - around 10 indexes, i.e. 5 DynamoDB tables and 2 types per table
  - As of ES 6.x [only one type per index](https://www.elastic.co/guide/en/elasticsearch/reference/current/_basic_concepts.html#_type)
- Shards
  - 1-2 shards per index
  - Since we will have a higher number of relatively small indexes, keep shard overhead to a minimum ([docs](https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster))
  - >Small shards result in small segments, which increases overhead. Aim to keep the average shard size between a few GB and a few tens of GB.
  - >A good rule-of-thumb is to ensure you keep the number of shards per node below 20 to 25 per GB heap it has configured
- Replicas
  - 1 primary + 1 replica shard to distribute data to second node

Example template sets one shard per index and keeps the default of one replica per shard.
```json
{
  "index_patterns": ["*"],
  "settings": {
    "number_of_shards": 1
  }
}
```
