## Citus Multi-tenancy clustering

### What is Citus

The Citus database is an open source extension to Postgres that gives you all the greatness of Postgres, at any scale—from a single node to a large distributed database cluster. Because Citus is an extension (not a fork) to Postgres, when you use Citus, you are also using Postgres. You can leverage the latest Postgres features, tooling, and ecosystem. (https://www.citusdata.com/)



### Experiment

Our main target is to understand how to configure the citus cluster for a multi-tenancy application, where we have to server data of same schema but for different tenant or company (in our case). Each company want their data isolated, secure. Tenant/ Company wise we may have to adjust the resource based on the plan/ package each tenant/ company subscribed. We may have to provide additional resource or security or configuration for specific tenant like: additional hardware or nodes, additional backup, additional configuration. Citus allows us to manage this within a same cluster with different nodes. Without managing separate database or cluster for each tenant .

**References:**

- https://docs.citusdata.com/en/stable/get_started/what_is_citus.html#when-to-use-citus
- https://docs.citusdata.com/en/stable/admin_guide/cluster_management.html



We'll have 1 single co-ordinator to orchastrate our cluster and 2 worker nodes. Worker nodes can be increased by adding new container in `docker-compose.yml`file

**References:**

- Concept: https://docs.citusdata.com/en/stable/get_started/concepts.html 
- Docker: https://github.com/citusdata/docker 



We'll experiment, multi-tenancy concept of citus using `distributed table` feature. We'll do `row sharing` based on tenant. In our case we'll create 2 tables `companies` and `campaigns`. 

**References**

- https://docs.citusdata.com/en/stable/use_cases/multi_tenant.html
- https://docs.citusdata.com/en/stable/installation/multi_node.html



```sql
CREATE TABLE companies (
  id bigserial PRIMARY KEY,
  name text NOT NULL
);


CREATE TABLE campaigns (
  id bigserial,
  company_id bigint REFERENCES companies (id),
  name text NOT NULL,
  PRIMARY KEY (company_id, id)
);
```

Each campaign must be under a company. We'll consider each company is an individual tenant, so we'll distribute our data based on company and shard as well.

Experiment steps

- Citus cluster creating using docker compose 
- Creating tables and configure or distributed themusing pgAdmin or command line tools
- Insert some sample data
- Do some query to understand the internal scenario
- Tenant wise node dedication and transfer tenant data to that dedicated node
- Shutdown other worker node(s)
- Do some query to justify the isolated tenancy concept



### Steps

**1. Cluster creation**

Make sure your docker is running. We'll use the local volume inside `./data` folder, so that we can see the postgresql data folder in our host machine, and remove them easily once our experiment completed. We are using `citus:12.1` 

```bash
docker compose -p citus-cluster up -d
```



**2.  Data preparation**

Connect co-ordinator cluster using the credential from docker-compose. 



Verify that all nodes are up and connected. It may take couple of seconds to connect all nodes.

```sql
SELECT * FROM pg_dist_node;
```

```sql
SELECT * FROM master_get_active_worker_nodes();
```



Table creation 

```sql
CREATE TABLE companies (
  id bigserial PRIMARY KEY,
  name text NOT NULL
);


CREATE TABLE campaigns (
  id bigserial,
  company_id bigint REFERENCES companies (id),
  name text NOT NULL,
  PRIMARY KEY (company_id, id)
);
```



Table distribution and sharding

```sql
SELECT create_distributed_table('companies',   'id');
SELECT create_distributed_table('campaigns',   'company_id');
```



Insert some companies (5 items)

```sql
INSERT INTO companies (name)
VALUES
('Google'),('Microsoft'),('Apple'),('Salesforce'),('SAP');
```



Insert some campaigns (50 items, 10 for each company)

```sql
INSERT INTO campaigns (name, company_id)
VALUES
    -- Company 1
    ('camp-1', 1), ('camp-2', 1), ('camp-3', 1), ('camp-4', 1), ('camp-5', 1),
    ('camp-6', 1), ('camp-7', 1), ('camp-8', 1), ('camp-9', 1), ('camp-10', 1),

    -- Company 2
    ('camp-1', 2), ('camp-2', 2), ('camp-3', 2), ('camp-4', 2), ('camp-5', 2),
    ('camp-6', 2), ('camp-7', 2), ('camp-8', 2), ('camp-9', 2), ('camp-10', 2),

    -- Company 3
    ('camp-1', 3), ('camp-2', 3), ('camp-3', 3), ('camp-4', 3), ('camp-5', 3),
    ('camp-6', 3), ('camp-7', 3), ('camp-8', 3), ('camp-9', 3), ('camp-10', 3),

    -- Company 4
    ('camp-1', 4), ('camp-2', 4), ('camp-3', 4), ('camp-4', 4), ('camp-5', 4),
    ('camp-6', 4), ('camp-7', 4), ('camp-8', 4), ('camp-9', 4), ('camp-10', 4),

    -- Company 5
    ('camp-1', 5), ('camp-2', 5), ('camp-3', 5), ('camp-4', 5), ('camp-5', 5),
    ('camp-6', 5), ('camp-7', 5), ('camp-8', 5), ('camp-9', 5), ('camp-10', 5);


```



**3. Understanding internal scenario**

Query citus_shards directly to have a detailed listing for all tables. Where we can see the distribution of tables into shards and shards into nodes

```sql
select * from citus_shards; -- to view table distribution & shards distribution 
select * from pg_dist_shard_placement; -- to view the shards placements only 
select * from pg_dist_shard; -- to view the table id range distribution into shads
```

Citus distributed the tables rows into shards based on the table id range. A default value is configured, in my case the distribution is `134217726`, mean citus stored this amount of entries of each table in a single shard. 

As we created 5 companies and 50 campaigns; so, we;ll the the which shards are containing this data for each table. Before move, we have to understand the result of above 3 queries. 

In my case, I have filtered out the shard ids from this query

```sql
select * FROM pg_dist_shard_placement;
select * from pg_dist_shard WHERE
shardminvalue::bigint = 0;
```
This is the result I got 

| logicalrelid | shardid | shardstorage | shardminvalue | Shardmaxvalue |
| ------------ | ------- | ------------ | ------------- | ------------- |
| companies    | 102024  | t            | 0             | 134217727     |
| campaigns    | 102056  | t            | 0             | 134217727     |

In our case all of our comapnies data are stored in `102024` shard and campaigns in `102056` shard. 



How we'll see which nodes are containing these shards

```sql
SELECT p.shardid, n.nodeid, n.groupid, n.nodename
FROM pg_dist_placement p
JOIN pg_dist_node n ON p.groupid = n.groupid
WHERE p.shardid IN (
    SELECT shardid
	FROM pg_dist_shard
    WHERE shardid=102024 OR shardid=102056
)

```

In my case the result is

| shardid | nodeid | groupid | Nodename               |
| ------- | ------ | ------- | ---------------------- |
| 102056  | 2      | 2       | citus-cluster_worker_1 |
| 102024  | 2      | 2       | citus-cluster_worker_1 |

That means all of our data are stored in worker-1 node in my case.



**4. Tenant Separation**

Consider company 2 want to separate their data into a dedicated node. Before do that we have to verify that the worker nodes are in `logical` replication mode instead of `replica`. 

Check current replication method of each node, connect `psql` from docker container of each worker. This can be configure directly from the docker script as well, but in thie experiment we'll do it manually.

In the worker-1 and worker-2 command line execution, (As we don't set any credential, so just type `psql`)  connect postgresql command line tool

```sql
SHOW wal_level;
```

In our case, both nodes are running in `replica` `wal_level ` mode. So, we'll convert this into `logical` in both nodes

```sql
ALTER SYSTEM SET wal_level = 'logical';
```

Then we have to restart the nodes from command line or directly from docker desktop

```bash
docker compose -p citus-cluster restart worker_1
docker compose -p citus-cluster restart worker_2
```

Again connect both node command line and check whether the `wal_level` is updated or not. 



Now back to the co-ordinator, and isolated the `company 2` data into a separate shard.

```sql
SELECT isolate_tenant_to_new_shard(
'companies', 2, 'CASCADE'
);
```

In the response we'll get the new shard id. In my case the result is

| isolated_tenant_to_new_shard |
| ---------------------------- |
| 102073                       |



Now finding out which node is containing this shard

```sql
SELECT p.shardid, n.nodeid, n.groupid, n.nodename
FROM pg_dist_placement p
JOIN pg_dist_node n ON p.groupid = n.groupid
WHERE p.shardid IN (
    SELECT shardid
	FROM pg_dist_shard
    WHERE shardid=102073
)

```

In my case this shard is also containing by ` worker node 1`.

So, we'll consider the `worker node 2` as a dedicate node for `company 2`. To move all data of company 2 (stored in shardid 102073 in my case) into worker node 2

```sql
SELECT citus_move_shard_placement(
	102073,
	'citus-cluster_worker_1', 5432,
	'citus-cluster_worker_2', 5432
);
```



To verify the movement execute this query again

```sql
SELECT p.shardid, n.nodeid, n.groupid, n.nodename
FROM pg_dist_placement p
JOIN pg_dist_node n ON p.groupid = n.groupid
WHERE p.shardid IN (
    SELECT shardid
	FROM pg_dist_shard
    WHERE shardid=102073
)
```

In my case, this time this shard is containing by ` worker node 2`.



**5. Experiment the dedicated node concept**

Now do some experimental data fetch queries

```sql
select * from companies; -- 5 items in result
select * from campaigns; -- 50 items in result
select * from companies where id=2; -- 1 item in result
select * from campaigns where company_id=2; -- 10 items in result
```



Now shutting down the worker node 1 using command line or directly from docker desktop

```bash
docker compose -p citus-cluster stop worker_1
```



Now again do these fetch queries

```sql
select * from companies; -- Error
select * from campaigns; -- Error
select * from companies where id=2; -- 1 item in result
select * from campaigns where company_id=2; -- 10 items in result
```

Co-ordinator aggregate the result from multiple worker nodes. In this scenario, worker-node-1 is down, so, when co-ordinator try to fetch all companies and all campaigns it failed to connect worker-1 node. But when we specifically fetch company=2 data, it can directly fetch from worker-node-2.



Now restart the worker node 1 

```bash
docker compose -p citus-cluster restart worker_1
```

Then again do these fetch queries

```sql
select * from companies; -- 5 items in result
select * from campaigns; -- 50 items in result
select * from companies where id=2; -- 1 item in result
select * from campaigns where company_id=2; -- 10 items in result
```

Everything works fine!

