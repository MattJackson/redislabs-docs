---
Title: High Availability for Replic Shards
description:
weight: $weight
alwaysopen: false
categories: ["RS"]
---
When you enable [database replication]({{< relref "/rs/concepts/high-availability/replication.md" >}}) for your database,
Redis Enterprise Software replicates your data to a replica node to make sure that your data is highly available.
If the replica node fails or if the master node fails and the replica is promoted to master,
the remaining master node is a single point of failure.

You can configure high availability for replica shards (replica HA) so that the cluster automatically migrates the replica shards to an available node.
An available node is a node that:

1. Meets replica migration requirements, such as [rack-awareness]({{< relref "/rs/concepts/high-availability/rack-zone-awareness.md" >}}).
1. Has enough available RAM to store the replica shard.
1. Does not also contain the master shard.

In practice, replica migration creates a new replica shard and replicates the data from the master shard to the new replica shard.
For example:

1. Node:2 has a master shard and node:3 has the corresponding the replica shard.
1. Either:

    - Node:2 fails and the replica shard on node:3 is promoted to master.
    - Node:3 fails and the master shard is no longer replicated to the replica shard on the failed node.

1. If replica HA is enabled, a new replica shard is created on an available node.
1. The data from the master shard is replicated to the new replica shard.

{{< note >}}
- Replica HA follows all prerequisites of replica migration, such as [rack-awareness]({{< relref "/rs/concepts/high-availability/rack-zone-awareness.md" >}}).
- Replica HA migrates as many shards as possible based on available DRAM in the target node. When no DRAM is available, replica HA stops migrating replica shards to that node.
{{< /note >}}

## Configuring high availability for replica shards

Using rladmin or the REST API, replica HA is controlled on the database level and on the cluster level.
You can enable or disable replica HA for a database or for the entire cluster.

When replica HA is enabled for both the cluster and a database,
replica shards for that database are automatically migrated to another node in the event of a master or replica shard failure.
If replica HA is disabled at the cluster level,
replica HA will not migrate replica shards even if replica HA is enabled for a database.

By default, slave HA is enabled for the cluster and disabled for each database so that o enable replica HA for a database, enable replica HA for that database.

{{< note >}}
For Active-Active databases, replica HA is enabled for the database by default to make sure that replica shards are available for Active-Active replication.
{{< /note >}}

To enable replica HA for a cluster using rladmin, run:

    rladmin tune cluster slave_ha enabled

To disable replica HA for a specific database using rladmin, run:

    rladmin tune db <bdb_uid> slave_ha disabled

## Replica HA configuration options

You can see the current configuration options for replica HA with: `rladmin info cluster`

### Grace period

By default, replica HA has a 10-minute grace period after node failure and before new replica shards are created.
To configure this grace period from rladmin, run:

    rladmin tune cluster slave_ha_grace_period <time_in_seconds>

### Shard priority

Replica shard migration is based on priority so that, in the case of limited memory resources,
the most important replica shards are migrated first.
replica HA migrates replica shards for databases according to this order of priority:

1. slave_ha_priority - The replica shards of the database with the higher slave_ha_priority
    integer value are migrated first.

    To assign priority to a database, run:

    ```sh
rladmin tune db <bdb_uid> slave_ha_priority <positive integer>
    ```

1. Active-Active databases - Active-Active database synchronization uses replica shards to synchronize between the replicas.
1. Database size - It is easier and more efficient to move replica shards of smaller databases.
1. Database UID - The replica shards of databases with a higher UID are moved first.

### Cooldown periods

Both the cluster and the database have cooldown periods.
After node failure, the cluster cooldown period prevents another replica migration due to another node failure for any
database in the cluster until the cooldown period ends (Default: 1 hour).

After a database is migrated with replica HA,
it cannot go through another replica migration due to another node failure until the cooldown period for the database ends (Default: 2 hours).

To configure this grace period from rladmin, run:

- For the cluster:

    ```sh
rladmin tune cluster slave_ha_cooldown_period <time_in_seconds>
    ```

- For all databases in the cluster:

    ```sh
rladmin tune cluster slave_ha_bdb_cooldown_period <time_in_seconds>
    ```

### Alerts

The following alerts are sent during replica HA activation:

- Shard migration begins after the grace period.
- Shard migration fails because there is no available node (Sent hourly).
- Shard migration is delayed because of the cooldown period.
