
## Cache-Aside

This is perhaps the most commonly used caching approach, at least in the projects that I worked on. The cache sits on the  _side_  and the application  **directly**  talks to both the cache and the database. There is no connection between the cache and the primary database. All operations to cache and the database are handled by the application. This is shown in the figure below.

![cache-aside](https://codeahoy.com/img/cache-aside.png)

Here’s what’s happening:

1.  The application first checks the cache.
2.  If the data is found in cache, we’ve  _cache hit_. The data is read and returned to the client.
3.  If the data is  **not found**  in cache, we’ve  _cache miss_. The application has to do some  **extra work**. It queries the database to read the data, returns it to the client and  **stores**  the data in cache so the subsequent reads for the same data results in a cache hit.

#### Pros
- update logic is on application level, easy to implement
- cache only contains what the application requests for

#### Cons
1. each cache miss results in 3 trips to read DB
2. data may be stale if DB is updated directly

## Read-Through Cache

Read-through cache sits in-line with the database. When there is a cache miss, it loads missing data from database, populates the cache and returns it to the application.

![read-through](https://codeahoy.com/img/read-through.png)

Both cache-aside and read-through strategies load data  **lazily**, that is, only when it is first read.

#### Use Cases, Pros and Cons

While read-through and cache-aside are very similar, there are at least two key differences:

- In cache-aside, the application is responsible for fetching data from the database and populating the cache. In read-through, this logic is usually supported by the library or stand-alone cache provider.

#### Pros
- Application logic is simple
- Easily scale the reads 

#### Cons:
- The disadvantage is that when the data is requested the first time, it always results in cache miss and incurs the extra penalty of loading data to the cache. Developers deal with this by ‘_warming_’ or ‘pre-heating’ the cache by issuing queries manually.
- Data access logic is in the cache, needs to write a plugin to access DB


## Write-Through Cache

In this write strategy, data is first written to the cache and then to the database. The cache sits in-line with the database and writes always go  _through_  the cache to the main database. This helps cache maintain consistency with the main database.

![write-through](https://codeahoy.com/img/write-through.png)

Here’s what happens when an application wants to write data or update a value:

1.  The application writes the data directly to the cache.
2.  The cache updates the data in the main database. When the write is complete, both the cache and the database have the same value and the cache always remains consistent.

#### Use Cases, Pros and Cons

On its own, write-through caches don’t seem to do much, in fact, they introduce extra write  **latency**  because data is written to the cache first and then to the main database (two write operations.) But when paired with read-through caches, we get all the benefits of read-through and we also get data  **consistency**  guarantee, freeing us from using cache invalidation (assuming ALL writes to the database go through the cache.)

#### Pros
- reads have lower latency
- the cache and DB are in sync

#### Cons
- Writes have higher latency because they need to wait for the DB writes to finish 
- Infrequent data is also stored in the cache

[DynamoDB Accelerator (DAX)](https://aws.amazon.com/dynamodb/dax/)  is a good example of read-through / write-through cache. It sits inline with DynamoDB and your application. Reads and writes to DynamoDB can be done through DAX. (Side note: If you are planning to use DAX, please make sure you familiarize yourself with  [its data consistency model](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.consistency.html)  and how it interplays with DynamoDB.)

## Write-Around

Here, data is written directly to the database and only the data that is read makes it way into the cache.

### Pros:
- Database is only source of truth
- 

#### Use Cases, Pros and Cons

Write-around can be combine with read-through and provides good performance in situations where data is written once and read less frequently or never. For example, real-time logs or chatroom messages. Likewise, this pattern can be combined with cache-aside as well.

## Write-Back or Write-Behind

Here, the application writes data to the cache which stores the data and acknowledges to the application  _immediately_. Then later, the cache writes the data  _back_  to the database.

This is very similar to to Write-Through but there’s one crucial difference: In Write-Through, the data written to the cache is  **synchronously**  updated in the main database. In Write-Back, the data written to the cache is  **asynchronously**  updated in the main database. From the application perspective, writes to Write-Back caches are  _faster_  because only the cache needed to be updated before returning a response.

![write-back](https://codeahoy.com/img/write-back.png)

This is sometimes called write-behind as well.

#### Use Cases, Pros and Cons

Write back caches improve the write performance and are good for  **write-heavy**  workloads. When combined with read-through, it works good for mixed workloads, where the most recently updated and accessed data is always available in cache.

It’s resilient to database failures and can tolerate some database downtime. If batching or coalescing is supported, it can reduce overall writes to the database, which decreases the load and  **reduces costs**, if the database provider charges by number of requests e.g. DynamoDB. Keep in mind that  **DAX is write-through**  so you won’t see any reductions in costs if your application is write heavy. (When I first heard of DAX, this was my first question - DynamoDB can be very expensive, but damn you Amazon.)

Some developers use Redis for both cache-aside and write-back to better absorb spikes during peak load. The main disadvantage is that if there’s a cache failure, the data may be permanently lost.

Most relational databases storage engines (i.e. InnoDB) have write-back cache enabled by default in their internals. Queries are first written to memory and eventually flushed to the disk.

#### Pros
- Low read and write latency
- Cache and database are eventually consistent

### Cons:
- Data loss in case of cache down
- Infrequent data is also stored in the cache
