MongoDB Europe 2016
-------------------

Mongo 3.2 Features
=============

- Aggregation pipelines (a bit like joins)
- Document validation at database level
- Can use in memory data storage (at the same time as disk-based)

Mongo 3.4 Features
==========

- MongoDB Compass
-- GUI alternative to mongodb CLI
-- Lots of interactive graphs
-- Run queries using GUI, information on query plans, able to create indexes...
-- CPU, Network, etc info based on mongo-top etc.
-- Create, clone, update etc documents
-- Manage databse validation rules
- Read-Only views
- Recursive $lookup (part of aggregation pipelines)
-- graph-like queries!
- Zone sharding
-- tag documents as living in a particular datacentre to ensure queries
for that document don't leave the datacentre if they don't need to.
- Cloud Atlas
-- Mongo cloud provisioning from MongoDB themselves
-- Includes free-tier with limited data storage to try out
-- Currently only on AWS but plans to add Google and Microsoft Azure
-- Role based access users
-- IP whitelists and/or VPC peering for network security
-- Lots of performance metrics. Not sure about Slow query log?

Advanced MongoDB Aggregation Pipelines
============================

- Allows monadic composition of individual steps
-- things like sort, group, filter etc.
- Designed to conceptually match the kind of operation you'd write in code using
the above tools
- Except that its much more performant to do this at the database level
-- taking advantage of indexes, performance optimisations etc
- Can run pipelines over micro-sharded data for large datasets
-- microshard = data shards existing on the same machine.
- Pipelines are reordered automatically by MongoDB to make them as performant
as possible
-- Examples include leveraging indexes to filter or match rather than doing this
in memory

*New in 3.4*

- $graphLookup
-- using `startWith` and `connectToField` we can follow links between fields in
documents, following the 'graph'.
- Facets - return stats to organise query results by
-- e.g. $bucket, $sortByCount, etc.
- Lots of functional operators like zip, map, slice etc. are now available
as aggregation pipeline commands for running complex queries that benefit
from database efficiencies

WiredTiger
==========

- Mongo's default (they'd like to offer different kinds)
- Can be used independently of MongoDB
- Written by people working on Berkeley DB (open source)
- Storage engine (alternatives available are mmapV1 and RocksDB)
- This is where ACID transactional gaurantees happen!
- Does away with memory locking mechanisms as they are slow on mutli-core
systems
- Instead it uses 'hazard pointers'. These use the primitive types that locking
uses to create an alternative algorithm.
- Uses a compare and swap (cas) algorithm to avoid race conditions between
two attempts to write to memory. "If the memory value is the same as when I read
it, assign it to this new value"

Resilient MongoDB Applications
==========================

See blog post
https://emptysqua.re/blog/how-to-write-resilient-mongodb-applications/

- queries to Mongo could feel for a number of reasons
-- Like network issues, authorisation, etc
- Even if Mongo was transactional this wouldn't help very much, especially for network
errors
- In the face of these errors, Mongo drivers will generally set their internal state
of that server to unknown, then throw an exception
-- All drivers represent the state of the Mongo cluster (known as Service Discovery
and Monitoring)
-- things like auth errors will just throw an exception
- If the request is tried again by the client, the driver no longer knows which server is the primary
- It checks each server looking for the primary (using a retry loop)
-- when it finds it it can run the query

*Client options for dealing with errors*

- Don't retry
-- Good for proper network outage or command errors
-- not good for intermittent network issues as we don't know if we succeeded
- Retry x times
-- not good for proper outage or command errors, waste of time
-- not good for intermittent network issues - may write multiple times
-- good for primary failover as next request succeeds
- Retry once
-- good for proper outage or command errors not wasting much time
-- not good for intermittent network issues - may write multiple times
-- good for primary failover as next request succeeds
- Retry once with idempotency - WINNER
-- good for proper outage or command errors not wasting much time
-- good for intermittent network issues - can't write multiple times
-- good for primary failover as next request succeeds

*Idempotency in mongo*

- Queries are naturally idempotent
- inserts can be made idempotent if unique _ids are used.
-- If we receive a duplicate key error that means the first insert worked! So
success
- deletes are idempotent if delete unique documents
- Updates
-- idempotent if the update sets properties to specific values (e.g. using $set)
-- inherently non-idempotent operations like incrementing an int can be made
idepempotent by splitting them into multiple operations (described in blog post)
-- Caveat 1 more operations required
-- requires cleaning up unfinished operations using some script

*Testing*

- Use MockupDB to stub out MongoDB for e2e testing
http://mockupdb.readthedocs.io/
https://emptysqua.re/blog/smart-strategies-for-resilient-mongodb-applications/

ETL (Extract, Transform Load)
========================

- How best to batch-load large (billions of docs) amounts of data?
(in general and into Mongo specifically)
- This is difficult in Mongo as we don't have simple table rows to deal with
but potentially complex documents
- There are tools for extracting from relational DBs and inserting into Mongo
(or elsewhere) (such as Talend) but they may not be flexible enough as they are not
specifically geared towards Mongo
- A better alternative may be to write your own code
- Orders of magnitude are important
-- Typical CPU operation is in order of nanoseconds
-- Roundtrip to a database (even without network) is in order of milliseconds (due to
extra layers of network operations involved). Even if data is cached rather than
written to disk.
-- Network latency is often longer than disk writes

*Mistake 1 - Nested Queries*

- Involve many more round trips to the database and heavier processing

*Mistake 2 - Building documents in the database*
- Could split the queries to happen sequentially, saving intermediate
state to the database to allow this
- Now we have only several read queries but may have many more writes!
- This may even be worse than before!

*Mistake 3 - Load into memory*

- Could store intermediate state in memory to save on database write round trips
(remember in memory operations are in the order of nanoseconds)
- The problem is for large datasets memory use is an issue!

*Potential Solution*

- iterate through all datasources at once
- assemble full document in memory
- write once to Mongo

Mongo Scalability at Nuxeo
==============

- Scale out reads
-- Use read replicas
-- read traffic gets routing to one of the available replicas, therefore each
replica handles less traffic than a single-node deployment

- Scale out writes
-- use sharding
-- the data itself must be split amongst a cluster of shards
-- Incoming writes are must first decide which shard to go to, but as above
each shard will now deal with less write requests on average

Graph Operations with MongoDB
======================

- Vertices/Node = MongoDB document
- Native Graph DBs are good at graphs
-- Queries are performant
- But
-- Query languages can be complex
-- Poorly optimised for non-traversal queries (just retrieving the document)
-- Less often used as a persistence layer / System of Record
- Could use Relationship DBs instead but
-- Relationships are actually JOINs, rather than semantic connections
-- analysing complex relationships using joins can be challenging
- MongoDB graph queries may solve much of the use cases and solve some of these
cons

*$graphLookup*

- Part of aggregation pipeline
- Similar to $lookup
- Can lookup within a single collection or across multiple
- `startWith` (friends) -> `connectToField` (name) -> `connectFromField` (friends)
{
  'name': 'Lucy',
  'friends': ['Bob']
}

{
  'name': 'Bob',
  'friends': ['Kevin']
}

{
  'name': 'Kevin'
}
- `startWith` can inspect arrays
- `connectToField` and `connectFromField` can use compound fields (one.two)
- filters can be applied etc.
- $lookup and $graphLookup mean multi-stage queries that previously may have been
done in the application layer can now be pushed to the DB layer.
- Recursions can use indexes for efficiency
