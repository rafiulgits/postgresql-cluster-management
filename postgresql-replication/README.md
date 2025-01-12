## PostgreSQL Replication 

Configuring replication between two databases is considered to be the best strategy for achieving high availability during disasters and provides fault tolerance against unexpected failures. PostgreSQL satisfies this requirement through streaming replication. We shall talk about another option called logical replication and logical decoding in our future blog post.

**References**

* Blog: https://www.percona.com/blog/setting-up-streaming-replication-postgresql/
* Documentation: https://www.postgresql.org/docs/current/runtime-config-replication.html
* PostgreSQL H/ATutorial: https://www.youtube.com/playlist?list=PLBrWqg4Ny6vVwwrxjgEtJgdreMVbWkBz0
* Docker: https://github.com/eremeykin/pg-primary-replica/blob/main/README.md



### Types of Replication

| Feature              | **Physical Replication**                        | **Logical Replication**                       | **Streaming Replication**                     |
| -------------------- | ----------------------------------------------- | --------------------------------------------- | --------------------------------------------- |
| Replication Type     | Block-level (binary) replication of WAL files   | Row-level replication of individual tables    | Physical replication using WAL data streaming |
| Data Granularity     | Entire database cluster                         | Specific tables or databases                  | Entire database cluster                       |
| Direction            | One-way                                         | One-way                                       | One-way                                       |
| Data Consistency     | Fully consistent with the primary (binary copy) | May diverge; DDL changes are not replicated   | Fully consistent                              |
| Supports DDL Changes | No                                              | Yes                                           | No                                            |
| Setup Complexity     | Moderate                                        | Simple                                        | Moderate                                      |
| Use Cases            | Backup, disaster recovery, read scaling         | Data warehousing, ETL, selective replication  | High availability, disaster recovery          |
| Supported Versions   | Requires same major version of PostgreSQL       | Can work across different PostgreSQL versions | Requires same major version of PostgreSQL     |
| Downstream Write     | No (read-only replicas)                         | Yes (on non-replicated tables)                | No (read-only replicas)                       |



 #### Detailed Explanations and Use Cases

**1. Physical Replication**

- How It Works:
  - Physical replication works by replicating the entire database cluster (data directory) at the block level using Write-Ahead Logs (WAL). It creates an exact binary copy of the primary server on the replica.
- Features:

  - Only supports **read-only replicas**.
  - Replication is at the cluster level, meaning all databases and tables are replicated.
  - DDL changes (e.g., table creation or schema changes) are automatically included.

- Use Cases:
  - **High Availability**: Provides a full copy of the database for failover in case of primary server failure.
  - **Disaster Recovery**: Ensures data integrity and redundancy.
  - **Read Scaling**: Offloads read queries to replicas.



**2. Logical Replication**

- How It Works:

  - Logical replication works by replicating changes at the SQL level (e.g., INSERT, UPDATE, DELETE) for specific tables. It uses a publication/subscription model.

- Features:

  - Supports replication of **specific tables or subsets of data**.
  - Allows DDL changes but does **not replicate them**.
  - Enables downstream servers to write data into non-replicated tables.

- Use Cases:
  - **Data Integration**: Sync specific tables with external systems (e.g., analytics or ETL pipelines).
  - **Cross-Version Replication**: Migrate data between different PostgreSQL versions or clusters.
  - **Selective Replication**: Use when only specific tables need to be replicated, not the entire cluster.



**3. Streaming Replication**

- **How It Works**:

  - Streaming replication is a form of physical replication where WAL records are streamed in real time from the primary to the replica server over a network connection.

- **Features**:

  - Provides **real-time replication** with low latency.
  - Replicas are **read-only** and consistent with the primary.
  - Requires the same major version of PostgreSQL.

- **Use Cases**:
  - **Low-Latency HA**: Ensure near-instant failover with minimal data loss.
  - **Scalable Reads**: Offload read-heavy workloads to replicas.
  - **Disaster Recovery**: Maintain up-to-date replicas for recovery.



####  Choosing Between Them

1. **Physical Replication**:

   - Use when you need **complete cluster-level replication** with strict consistency, e.g., for HA or backup.

2. **Logical Replication**:

   - Use when you need **table-level replication** or cross-version replication, e.g., for ETL or selective data sharing.

3. **Streaming Replication**:
   - Use when you need **real-time updates** to replicas with minimal latency, e.g., for HA or scalable read workloads.





### Experiment

In this experiment, we'll create 2 separate docker container, one is for primary database and another is replica. We created 2 different docker compose file for them and using a  **common docker network** so that they can communicate with each other. For multiple replication instance please adjust the configuration under `00_init.sql` for setting up the credential and inside the `docker-compose.primary.yml` command; as we set max 10 replication slot there. 



#### Steps

**1. Create Docker Network**

```bash
docker network create pg_cluster_network
```



**2. Creating the Primary Database Instance**

```bash
docker compose -f docker-compose.primary.yml -p pg-cluster-primary up -d
```



**3. Insert Data into Primary Database**

```sql
create table companies (
	id serial primary key,
	name varchar(100) not null
);


insert into companies (name)
values
('abc inc'),
('apple'),
('microsoft'),
('google');

select * from companies;
```



**4. Creating the Replica Instance**

```bash
docker compose -f docker-compose.replica.yml -p pg-cluster-replica up -d
```

After replica instance up, connect to database through pgAdmin or command line tool using the credential (credential available in `docker-compose.replica.yml`). Then verify the the `companies` tables are created inside the replica instance. This may take some time, as the replica need to connect with primary database and do a backup operation to get all data.



**5. Query inside Replicate**

To do fetch query inside the replica instance we need to authorize the replica. To this authorization connect to primary instance and provide authorization. We can provide authorization on full schema or to a specific table only.

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicator; -- schema based
GRANT USAGE ON SCHEMA public.companies TO replicator; -- table based
```

After providing authorization, connect to replica instance and do fetch query

```sql
select * from companies;
```

To verify the continuous replication, insert some more data into primary database and query inside replica instance and check all the data are replicated. Replication may take couple of time.
