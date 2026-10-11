# PostgreSQL Replication & High Availability

A hands-on guide to PostgreSQL replication, troubleshooting, and database failover, covering streaming replication, logical replication, and work in progress on Patroni-based high availability.

## Repository Structure

```text
Postgresql-Replication/
├── Logical Replications/
├── Streaming Replications/
└── README.md
```

## What's Covered?

### 1. Streaming Replications

Learn how PostgreSQL physical streaming replication works between primary and standby database instances.

Topics covered:
- Primary and standby configuration.
- WAL streaming and replication.
- Replication status verification.
- Replication lag and monitoring.
- Setup procedures and troubleshooting.
- Failover concepts and standby promotion.

**Explore:** [Streaming Replications](https://github.com/VigneshSelvam92/Postgresql-Replication/tree/main/Streaming%20Replications)

### 2. Logical Replications

Learn how PostgreSQL logical replication works using publications and subscriptions to replicate selected tables between databases.

Topics covered:
- Publisher and subscriber configuration.
- Publications and subscriptions.
- Table-level replication.
- Replication status verification.
- Setup procedures and troubleshooting.
- Common replication errors and recovery considerations.

**Explore:** [Logical Replications](https://github.com/VigneshSelvam92/Postgresql-Replication/tree/main/Logical%20Replications)

### 3. PostgreSQL High Availability & Failover — WIP

This section will explore how to improve PostgreSQL availability and recover from database failures.

Planned topics:
- Manual failover and standby promotion.
- Primary failure detection and recovery.
- Rejoining a former primary.
- Patroni-based automated failover.
- Cluster leader election and health monitoring.
- Split-brain prevention.
- Application traffic routing and high availability.
- Replication lag, recovery, and disaster recovery scenarios.

## Project Goals

- Understand PostgreSQL replication from setup to troubleshooting.
- Practise physical and logical replication in hands-on environments.
- Learn how to diagnose replication failures.
- Understand the difference between replication and high availability.
- Build towards a PostgreSQL HA architecture with automated failover.

## Official Documentation

- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/)
- [Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
- [Patroni Documentation](https://patroni.readthedocs.io/)

## Author

**Vignesh Selvam**

GitHub: [VigneshSelvam92](https://github.com/VigneshSelvam92)
