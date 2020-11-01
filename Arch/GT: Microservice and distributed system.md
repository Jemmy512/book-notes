# Microservice
## Service Description
RESTfull API, XML, IDL

## Service Registry

1. Registry API
    1. Registor API
    2. Anti-Registor API
    3. Hearbeat Report API
    4. Service Subscript API
    5. Service Update API
    6. Service Get API
2. Cluster Deploy
3. Services Health Status Dectation
4. Service Status Change Notification
5. Whitelist

## Service Monitor
1. Monitor Object
    1. Client Monitor
    2. Interface Monitor
    3. Resource Monitor
    4. Infrastructure Monitor

2. Monitor Indicator
    1. QPS (Query Per Seconds)
    2. Response Time
    3. Failure Rate

3. Data Collection
    1. Servcie Activaly Report
    2. Agent Collection
    3. Smapling Rate
4. Data Transmition
    1. UDP
    2. Kafka
    3. Binary Foramt: PB
    4. Text Foramt: JSON, XML
5. Data Process
    1. Aggragate by interface/machine
    2. Index DB, time series DB
6. Data Demonstration

7. Practice
    1. Beats, LogStash: Collect & Transform
    2. ElasticSearch: Search & Analyze
    3. Kibana: Visualize & Manager

## Service Track
[Dapper, a Large-Scale Distributed Systems Tracing Infrastructure](https://research.google/pubs/pub36356/)

## Service Governacne
1. Nodes Management
    1. Remove by registry
    2. Remove by service consumer
2. Load banlance
    1. Round-robin
    2. Weighted round-robin
    3. Least Active
    4. Consistent Hash
    5. Random
3. Service Routing

4. Service Fault Tolerance
    1. FailOver: Automatic switching on failure
    2. FailBack: Notification on failure
    3. FailCache: Cache on failure
    4. FailFast

5. Service Health Management
    1. Heartbeat switch protection mechanism
        * Protect registry center against vast requests from client when network is unstable
    2. Service node removal protection mechanism
        * Protect registry center against removing all nodes when the network is unstable

6. Configuration Center
    1. Application:
        1. Resource Servicing
        2. Business dynamic degradation
        3. Packet traffic switching

# High-Performance Design
1. Read and write separation
    1. Duplication Delay
        1. After writing, the read operation routed to the master server
        2. If read slave server failed, read master.
        3. Only non-key business use read and write separation
    2. Allocation Mechanism
        1. Code encapsulation
        2. Middle-ware encapsulation
2. Database and table separation
    1. Business database separation
        * have join, transaction, cost issues
    2. Table separation
        1. vertical separation
            1. Split out not frequently used, occupy a lot of space columns
            2. The complexity comes from increasing the number of operations
        2. horizontal separation
            1. Suitable for very big tables, e.g., more than 50,000,000 rows
            2. Complexity: data routing(Range, Hash, Configuration), join, count, order by
3. Cache
    1. Cache Penetration: the data to be searched doesn't exist at DB and the returned empty result set is not cached as well and hence every search for the key will hit the DB eventually.
        1. cache empty/null result
        2. Bloom filter
    2. Cache avalanche: lots of cached data expire at the same time or the cache service is down and all of a sudden all searches of these data will hit DB and cause high load to the DB layer and impact the performance.
        1. Using clusters to ensure that some cache server instance is in service at any point of time.
        2. Some other approaches like hystrix circuit breaker and rate limit can be configured so that the underlying system can still serve traffic and avoid high load
        3. Can adjust the expiration time for different keys so that they will not expire at the same time.
    3. Cache breakdown: the cached data expires and at the same time there are lots of search on the expired data which suddenly cause the searches to hit DB directly and increase the load to the DB layer dramatically.
        1. Use lock
        2. Asynchronous update
    3. Cache hotspot
4. Load Balance
    1. DNS Load Balance: geographic-level
    2. Hardware Load Balance: cluster-level
    3. Software Load Balance: machine-level
# High Availability Design

# Extensible Design