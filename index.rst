###############################################
APDB Backup Strategy and Implementation Options
###############################################

.. abstract::
   The data in Alert Production Database (APDB) is critical for the Alert Production pipeline and it needs to be backed up.
   This note explains the motivation behind the APDB backup strategy, its existing implementation and options for its future extension.


APDB Overview
=============

Backup strategy for APDB is influenced by how data is added, updated, and removed from the database.
APDB contains three major logical table types -- ``DiaObject``, ``DiaSource``, and ``DiaForcedSource`` -- and a number of auxiliary tables.
Alert Production pipeline issues these types of queries to APDB:

- retrieve latest version of ``DiaObject`` records in a given region;
- retrieve 12-month history of ``DiaSource`` and ``DiaForcedSource`` records in a given region;
- store new records in all three tables.

Cassandra-based APDB implementation uses various optimizations to improve performance of these queries:

- instead of ``DiaObject`` table the query uses a smaller ``DiaObjectLast`` table which contains latest version of ``DiaObject`` records, it is partitioned spatially;
- ``DiaSource`` and ``DiaForcedSource`` use spatial and temporal partitioning, the size of temporal partition is 30 days.

Queries for ``DiaSource`` and ``DiaForcedSource`` tables only need to query 13 latest partitions instead of whole time history.
The data in earlier partitions is not needed for AP pipeline and can eventually be dropped.
To simplify that cleanup process temporal partitions in a production APDB are mapped to individual physical tables.
For example, instead of a single large ``DiaSource`` table we have a series of smaller tables with names ``DiaSource_677``, ``DiaSource_678``, etc.
Each physical table contains data from a single temporal partition, its suffix corresponds to a partition ID.
Dropping one partition in this case means dropping a table, which is a much more efficient operation compared to dropping individual records.

Data from production APDB are replicated to Prompt Products Database (PPDB).
Partitioning scheme used by the regular tables is not suitable for queries issued by a replication process (e.g. select records added in the last hour).
To support replication queries a separate set of tables was added to APDB that contain duplicated data in regular tables and an additional information used by replication process:

- ``DiaObjectChunks``, ``DiaSourceChunks``, and ``DiaForcedSourceChunks`` -- same data as in original tables but partitioned in a different way;
- ``ApdbUpdateRecordChunks`` containing records that describe updates or deletions of existing data in regular tables;
- ``ApdbReplicaChunks`` containing information about "replica chunks", which are units of replication from APDB to PPDB.

The data chunks that have been replicated to PPDB are not needed in APDB and they will be eventually deleted.

To support a process of DiaObject deduplication that runs during daytime an additional table ``DiaObjectDedup`` was added.
This object contains DiaObject records that were stored since the last deduplication with only a small subset of fields.
The table is cleared after each deduplication.

All tables in Cassandra cluster are replicated with replication factor 3 and compressed.
Total size of the data that will be stored in APDB over survey lifetime is hard to estimate as it depends on many unknown factors.
An order-of-magnitude guess for the data scale after 10 years gives ~100TB of total uncompressed data in three main tables without compression in a single replica.

There are presently three Cassandra clusters at USDF:

- ``prod`` cluster for production data from LSSTCam and a few other instruments,
- ``dev`` cluster used for Prompt Production development activities with a small amount of data,
- ``int`` cluster for integration testing, this cluster is only for temporary data and does not need backups.

Backup Use Cases
================

The most important use case for backups is a recovery after various failures, e.g. when one or more nodes in a Cassandra cluster lose their data or in case of a disaster when the whole cluster is lost.
For such recovery it is sufficient to have the latest backup at hand, and in case of a site-wide disaster it is crucial to have an off-site copy of the backups.

In case there is a data corruption in APDB that was not noticed immediately it is helpful to keep an extended history of backups to be able to understand when such corruption happened and restore APDB to a clean state.
Due to the large volume of the data in APDB it is impossible to keep every backup (assuming that backups are performed daily).
Instead, a less granular set of backups can be kept for a limited time period, e.g. a number of daily, weekly, and monthly backups.

As mentioned in the overview section, some time-partitioned tables become unused after 12-month period and will be removed to reduce disk storage needs.
There may be reasons for accessing that deleted data later, e.g. understanding provenance or debugging science algorithms.
It would be prudent to save a backup copy of these tables before actually deleting them, this could be done either via regular yearly backups or as a separate dump of the tables.


Backup Tools
============

Cassandra does not have any built-in tools to perform backup or restore operations, but there are a couple of third-party packages that can be used for that:

- `cassandra-medusa`_ -- command line tools and gRPC service for uploading SSTable files to external storage and restoring cluster nodes from those files.
- `dsbulk`_ -- command line tool optimized for bulk dump of Cassandra tables in CSV format and loading CSV data into Cassandra.

In addition to that filesystem-level tools could be used to perform per-node backup and restore.
We describe each tool in more detail below.

cassandra-medusa
----------------

``cassandra-medusa`` is a third party tool supported by community.
It includes command line tools doing backup and restore of individual Cassandra nodes or the whole cluster.
Use of the ``cassandra-medusa`` CLI requires the package to be installed on every cluster node.

Additionally ``cassandra-medusa`` implements gRPC service that can run alongside Cassandra on every cluster node, it can perform some backup/restore operations with its gRPC API.
The main user of gRPC API is `k8ssandra`_ (Kubernetes operator for Cassandra) which deploys Docker containers running ``cassandra-medusa`` service.
``cassandra-medusa`` defines client-side API for interacting with gRPC service, it can be used to interact with gRPC service even without Kubernetes operator.

``cassandra-medusa`` performs backups using following sequence on each node:

- Flush all Cassandra in-memory tables to SSTables on disk.
- Create a snapshot of the current SSTables.
- Copy all SSTable files to an external storage (e.g. S3).
- Save current schema definition to an external storage.
- Save the index of files in a backup and additional metadata describing cluster to the same external storage.
- Remove snapshot created earlier.

``cassandra-medusa`` backups can be *full* or *differential*.
Full backups are self-contained and include a copy of each SSTable file.
Differential backups use deduplication by storing SSTable files in a shared storage; as SSTable files do not change this reduces storage needs.

Restoring a backup with ``medusa-cassandra`` involves following steps starting from a fresh state:

- Cassandra service must be stopped, Cassandra data directory must be empty.
- ``medusa-cassandra`` gRPC service container must be started in a special restore mode and be given the name of a backup to restore.
- The service will download all SSTable files (including system catalogs) to Cassandra data directory.
- The container will stop when download finishes.

Cassandra service can be started as usual after this restore.
The restore operation can be performed on individual hosts or on the whole cluster.

The speed of backup and restore is limited by the network and storage speed on both ends of connection, and this is the fastest option of all.
``medusa-cassandra`` restore process has significant limitations:

- With gRPC service the cluster being restored must have the same topology, i.e. the same number of nodes and the same token allocation.
- CLI tools have an option to restore to a cluster with different topology, but it will fail to work in many cases.
- There is no simple way to restore individual tables on the existing cluster.

In general ``medusa-cassandra`` backups are very suitable for disaster recovery when one needs to restore the latest known state of the same cluster.


dsbulk
------

``dsbulk`` can be used to efficiently dump contents of individual Cassandra tables in CSV format and load CSV files into Cassandra tables.
Compared to ``medusa-cassandra`` the speed of reading and writing is significantly slower as it needs to go through a regular CQL protocol.

The benefit of ``dsbulk`` is that it can load data into an existing cluster which can have different topology from the original cluster.
This can be helpful for cases when we want to restore one of the time-partitioned tables (e.g. ``DiaSource_NNN``) that were removed previously and cluster topology changed since then.

JIRA ticket `DM-54542`_ experimented with different options of ``dsbulk`` to study its performance and Cassandra behavior.
Bulk loading can introduce significant strain on Cassandra resources and needs to be performed with close monitoring of the resources.

`dax_apdb_deploy`_ package adds ``clone-keyspace`` script which wraps complexity of ``dsbulk`` and integrates it with Cassandra cluster configuration management tools.


Filesystem Backups
------------------

In addition to Cassandra-specific backup tools it may be possible to use OS-level filesystem tools to backup Cassandra data.
This option would require a careful approach to keep data in consistent state.
The approach may be similar to what ``medusa-cassandra`` does, assuming that Cassandra local storage is on ZFS:

- Flush all Cassandra in-memory tables to SSTables on disk.
- Make a ZFS snapshot of the filesystem including Cassandra data, commitlog, and hints directories.
- Backup that snapshot using some ZFS tools.
- Drop snapshot.

It is unclear if there are ZFS backup tools that can do deduplication of the data during backup.
Restoring backups in this case would replace original filesystem with a backup copy.
If restoring to a different host name or IP address, an extra step may be needed when bringing the cluster up to replace old cluster IP addresses.

This sort of backups has the same drawback as ``medusa-cassandra`` -- it can only be restored into the cluster with the same topology.


Backup Strategy
===============

This section covers already implemented options and possible future extensions.


Same Cluster Backup-Restore
---------------------------

To be able to recover a single node or the whole cluster after the data loss, e.g. after a catastrophic disk failure, we use ``medusa-cassandra`` backups.
Usually for this sort of recovery it is sufficient to have the latest backup, but there are other possible scenarios, as mentioned above, when it is beneficial to have an access to older backups.

To satisfy these two types of recovery we perform a set of backups in differential (deduplication) mode.
Backups are performed by the cron jobs on this schedule:

- *daily* backups - every day at 7:00 USDF time,
- *weekly* backups - every Monday at 7:10 USDF time,
- *monthly* backups - every 1st day of month at 7:20 USDF time,
- *yearly* backups - every 1st of January at 7:30 USDF time (for ``prod`` cluster only).

The time of the backup is chosen to happen after all nightly and daily APDB operations are expected to finish.
Backups are stored at USDF S3 bucket ``usdf-apdb-prod``, both ``prod`` and ``dev`` cluster backups are stored there in separate paths.

A separate cron job performs removal of obsolete backups, presently we keep a history of several backups of each type:

- 10 daily backups,
- 8 weekly backups,
- 12 monthly backups.

There is no cleanup of yearly backups at this time.
The purpose of yearly backups is to keep the history of the time partitioned tables (e.g. ``DiaSource_NNN``) which will be removed.
It is possible that yearly backups will be replaced with a different backup mechanism based on ``dsbulk`` tool.

As described above restoring of ``medusa-cassandra`` backups can only be done to the cluster with the same topology (same number of nodes).
Restore of the selected backup is done either to a single node or to the whole cluster.
The existing data on the restored nodes, if any, needs to be deleted as it will be replaced by a backup.
The procedure for restoring consists in running an Ansible playbook as described in `dax_apdb_deploy`_.
If backup is restored to a new cluster with different node names, the Ansible inventory will need to define mapping of the old host names to the new ones.

Backup of Deleted Tables
------------------------

From time to time the tables in APDB may need to be deleted entirely or purged of the existing data.
To avoid losing the history of the deleted data we will use one or both approaches:

- Create a ``medusa-cassandra`` backup with a special name that does not start with ``daily-``, ``weekly-``, etc.
  Those backups will include everything, not just deleted tables and it may be expensive to keep forever.
- Use ``clone-keyspace`` script to dump a specific set of tables to compressed CSV and copy it to the backup storage.
  The size of this sort of backups is limited, due to that it can be kept longer.

The `APDB Backups`_ page in Confluence contains a list of all special backups with description and possible expiration date.

In the future the ``clone-keyspace`` should be a preferred way to make one-off backups of a small number of tables as it allows more flexibility when restoring those backups (at a price of speed).

Off-site Backups
----------------

As mentioned above, to be able to recover after a site-wide disaster an off-site copy of the backups is needed.
Presently we do not have an external storage setup for off-site backups.

There could be several ways for exporting backups to external storage:

- S3-to-S3 mirroring which synchronizes an off-site S3 bucket with the contents of USDF bucket with APDB backups.
- Use the same tools (``medusa-cassandra`` and ``clone-keyspace``) to backup directly to off-site storage (in addition to USDF S3 storage).
- Implement in-house solution for transferring USDF backups to an off-site storage.

S3-to-S3 is likely the fastest and easiest way to implement off-site backups.

The main concern with the off-site backups is probably storage cost.
Potentially the history of the backups in for off-site storage can be reduced to limit the storage volume.
Some storage solutions offer "cold" storage with reduced pricing, but with additional limitations.

Off-site backups are the most pressing issue that needs to be resolved on the nearest time scale.


.. _cassandra-medusa: https://github.com/thelastpickle/cassandra-medusa
.. _dsbulk: https://docs.datastax.com/en/dsbulk
.. _k8ssandra: https://docs.k8ssandra.io/
.. _DM-54542: https://rubinobs.atlassian.net/browse/DM-54542
.. _dax_apdb_deploy: https://github.com/lsst-dm/dax_apdb_deploy
.. _APDB Backups: https://rubinobs.atlassian.net/wiki/x/KYDnaw
