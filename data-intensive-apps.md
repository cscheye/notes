# [Designing Data-Intensive Applications](https://dataintensive.net/)

## Ch 1. Reliable, Scalable, Maintainable Applications

- these days there are so many specialized but overlapping data stores; modern apps are generally composed by many data stores/systems: database, cache, search index, stream processing, batch processing
- how can we make a system reliable, scalable, and maintainable?
    - reliable -- consider hardware, software, and user faults
    - scalable -- understand load parameters, how performance will be affected when they increase; do this with monitoring percentiles. performance 𝛂 resources / load 
    - maintainable -- make ops tasks easy or automated, KISS
- monitoring resources -- *avoid averages; they skew results*. Instead, look at median and percentiles to understand what worst performance actually is
- vertical scaling (better hardware on a single machine) vs horizontal scaling (distribute data/workload across multiple machines, commodity hardware)
- elasticity = ability to auto-scale per load; good for unpredictable load, but more complex

## Ch 2. Data Models & Query Languages

- before sql... CODASYL hierarchical data model, everything represented as a tree, access data by writing imperative code that traversed a pathway
- SQL: models relational data, access data via declarative queries, explicit schema that's checked on write
    - not good for highly relational data or when schema isn't stable
- Document: models self-contained (not relational) data, mostly imperative query languages (some dbs have created declarative wrappers), flexible schema
    - MapReduce querying (NoSQL and big data dbs) -- map over documents and return a calculated value for matching docs, then reduce the map outcome to a final set of aggregated values
- Graph: models highly relational data, declarative queries, flexible schema
    - vertices and edges represent many different types of data and connections
    - property graphs and triple stores are two different kinds of graph dbs, both have idea of subject, predicate, object
- *declarative query language* (vs imperative)
    - state a desired data set, not how to get it; derived from algebra
    - simpler, more concise than specifying how to get a data set
    - allows db engine to make performance optimizations under the hood without changing query
- semantic web: make the internet a world wide database, represent data for computers as well as people
    - never caught on... why? I think moving toward it in a roundabout way: 1) SPAs consuming JSON APIs, 2) Full text searching and AI advancing so that we don't need structured data...

## Ch 3. Storage and Retrieval

### Indexes

- simple in-memory hash, where id => byte offset of data on disk
- LSM-Tree indexing strategy
    - uses SSTables (sorted string table)
    - writing: write to memtable (in-memory tree); when memtable is full, write that tree to disk in a file sorted by key (SSTable); periodically merge the oldest segmented SSTable files together
    - reading: first try to find key in memtable, then in progressively older SSTable files
    - optimizations: also write append-only logs in case system crashes before memtable is flushed to disk; keep a sparse hash (e.g. since strings are sorted you know that the key "banana" is between "apple" and "cantaloupe"); bloom filters (data structure that optimizes looking up keys that DNE)
    - pros: higher write throughput, more compressed, *sometimes* lower write amplification (one write to the db results in multiple writes to disk) because of sequential writes vs a b-tree that rewrites the entire page for one entry
    - cons: compaction process can have a significant impact on performance if reads have to wait for an expensive write operation (*this actually happened at Etsy! Nikki explained that they saw a huge performance hit when logs were being merged*)
- B-Trees
    - layers of "pages"; boundary values are always sorted, with references to the next page between (e.g. [100 => [100 => [100, 101, 102, ...], 110, 120, 130, ...], 200, ...] ). Final "leaf" page with individual keys (e.g. ^ [100,101,102]) either has values or refs to another page with values
    - reading: read references to lower pages until the leaf is reached
    - writing: every write means re-writing a whole page; the tree is kept balanced by splitting pages when an insert would exceed space limitations
        - WAL (write-ahead logs) for crash recovery: write log of operation before doing it to an append-only file
    - pros: battle-tested; a key's data is only in 1 place, therefore better for dbs that needs transactional safety guarantees
    - cons: sometimes higher write amplification than LSM-trees (have to rewrite entire page for a single entry; also have to split two pages when a page exceeds space limit); not as compact (always need extra room for adding more keys)

#### Getting data from the index

- clustered index = storing row data in the index
- nonclustered index = storing only a reference to the row within the index
    - reference points to a location in a heap file; the heap file only stores row once, when updating value you either overwrite in place or write a forwarding reference to the new location (in case it's too big for existing location)
- compromise -- index has some data and a reference for auxiliary data. A covering index contains enough info for some queries so that they don't need to actually read the whole row
- *mysql only uses one index per query*

### OLAP vs OLTP
Analytics processing access patterns vary significantly from transaction processing. Analytics needs to query and aggregate large volume of data, and latency isn't as important. Transaction processing (i.e. user-facing transactions) require low-latency and generally involve only a few records accessed by id (e.g. a user's purchase data). For small applications, the same storage engines can be used for both, but as data set becomes larger, it's best to use separate data stores for OLTP and OLAP.

- Data Warehouses = data stores optimized for analytics, tend to be column-oriented data stores
- ETL = "extract, transform, load";  the process of porting over data from OLTP systems into your OLAP data warehouse

#### Column-oriented databases
- columns (rather than rows) are stored together on disk, and row-data is associated via the order that it's stored in (i.e. the 3rd value in each column corresponds to the same row/instance)
- columnar data can be compressed using bitmap encoding -- clever! (see p. 98)
- instead of indexing, you can sort rows in a specific order that optimizes compression and/or range queries. You can have different sort orders on various replicas, so you still have data replication but each replica is optimized for a different type of query.
- materialized views = similar to sql view, but query results are actually stored in a table on disk instead of running the view sql on the fly
- Vertica has "projections", similar to materialized views. Be careful about creating these, just as indexes, because more projections means more work on writes/updates!
- write pattern: similar to LSM-trees -- write to an in-memory data structure and append-only logs, then periodically compact and merge into the columns stored on disk
- read pattern: again, similar to LSM-trees -- query optimizer needs to read from in-memory data store first and then disk. However, queries are declarative, so this is hidden from the developer
