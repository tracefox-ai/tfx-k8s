kubectl -n clickhouse port-forward \
    services/clickhouse-clickhouse \
    8123:8123

# Connect to chi-clickhouse-tracefox-0-0-0
kubectl exec -it -n clickhouse chi-clickhouse-tracefox-0-0-0 clickhouse-client

# Connect to other nodes
kubectl exec -it -n clickhouse chi-clickhouse-tracefox-0-0-0 clickhouse-client
kubectl exec -it -n clickhouse chi-clickhouse-tracefox-0-1-0 clickhouse-client
kubectl exec -it -n clickhouse chi-clickhouse-tracefox-1-0-0 clickhouse-client
kubectl exec -it -n clickhouse chi-clickhouse-tracefox-1-1-0 clickhouse-client

SELECT * FROM system.clusters WHERE cluster = 'tracefox';

Create a table that will be replicated across the cluster:

-- This creates the table on all nodes via distributed DDL
CREATE TABLE IF NOT EXISTS test_table ON CLUSTER tracefox
(
    id UInt64,
    name String,
    timestamp DateTime,
    value Float64
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/test_table', '{replica}')
PARTITION BY toYYYYMM(timestamp)
ORDER BY (id, timestamp);


Create a distributed table that routes queries across shards:
-- Create distributed table
CREATE TABLE IF NOT EXISTS test_table_distributed ON CLUSTER tracefox 
AS test_table
ENGINE = Distributed(tracefox, default, test_table, rand());


Insert Test Data

Insert some test data into the distributed table (this will route data to the shards):

-- Insert test data into the distributed table
INSERT INTO test_table_distributed (id, name, timestamp, value) VALUES
(1, 'test1', now(), 1.1),
(2, 'test2', now(), 2.2),
(3, 'test3', now(), 3.3),
(4, 'test4', now(), 4.4),
(5, 'test5', now(), 5.5),
(6, 'test6', now(), 6.6),
(7, 'test7', now(), 7.7),
(8, 'test8', now(), 8.8);

-- Wait a moment for replication to complete, then verify total count
SELECT count() AS total_count FROM test_table_distributed;


Test Replication

Verify that data is replicated within each shard by querying each node directly:

-- On chi-clickhouse-tracefox-0-0-0 (Shard 0, Replica 0)
SELECT 'chi-clickhouse-tracefox-0-0-0 (Shard 0, Replica 0)' AS node, count() AS count FROM test_table;

-- On chi-clickhouse-tracefox-0-1-0 (Shard 0, Replica 1) - should have same data as Shard 0
SELECT 'chi-clickhouse-tracefox-0-1-0 (Shard 0, Replica 1)' AS node, count() AS count FROM test_table;

-- On chi-clickhouse-tracefox-1-0-0 (Shard 1, Replica 0)
SELECT 'chi-clickhouse-tracefox-1-0-0 (Shard 1, Replica 0)' AS node, count() AS count FROM test_table;

-- On chi-clickhouse-tracefox-1-1-0 (Shard 1, Replica 1) - should have same data as Shard 1
SELECT 'chi-clickhouse-tracefox-1-1-0 (Shard 1, Replica 1)' AS node, count() AS count FROM test_table;

-- Note: Replicas within the same shard should have the same count, but different shards may have different counts
-- The sum of counts from all shards should equal the total count from test_table_distributed