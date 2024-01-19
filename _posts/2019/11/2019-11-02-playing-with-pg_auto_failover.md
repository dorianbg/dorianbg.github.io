---
title: "Introduction to PostgreSQL High availability with pg_auto_failover"
date: "2019-11-02"
categories: 
  - "data-engineering"
  - "data-systems"
  - "data-warehousing"
---

## Introduction

I am an avid fan of PostgreSQL database due to many reasons.

What has been lacking in PostgreSQL is a simple, "built-in", high-availability setup like Cassandra, MongoDB, Kafka... have.

I think the **[pg_auto_failover](https://github.com/citusdata/pg_auto_failover)** project open-sourced by CitusDB team (now Microsoft) is a huge step in that direction.

Setting up streaming replication in PostgreSQL is pretty easy. I recommend this [video](https://www.youtube.com/watch?v=NaPnYQBBdyU) and the related tutorial to setting it up for PG 10,11. PG 12 has a minor change that makes it even simpler.

Unfortunately the native streaming replication in PostgreSQL does not know when to failover from the primary node (leader) to the secondary node (follower). There are many solutions for this problem though the most recommended one is [Patroni.](https://github.com/zalando/patroni) Patroni is unfortunately specialised for dockerized environments and is fairly complex to setup compared to pg_auto_failover.

The way pg_auto_failover handles identifying when to failover from the primary to secondary node is using an additional process called **monitor** that basically monitors your HA setup (formation) and performs failover if it identifies one of your nodes is down. Technically you can implement this step yourself with eg. a Python script running every 1 second to check the heartbeat of PostgreSQL processes in the HA setup.

So if you want a simple solution and don't have Docker/Kubernetes, I'd recommend looking into pg_auto_failover.

Below are the steps to manually setup pg_auto_failover (on my Mac machine). The setup would be fairly similar on most Linux boxes. Although databases are stateful, you'd probably want to wrap this up in a script or something like Ansible playbook.

## **Local pg_auto_failover setup** 

#### 1. Install PostgreSQL to get the binaries (you might need postgresql-dev binaries for pg_auto_failover to be able to run make)

brew install postgresql

#### 2. Install pg_auto_failover using make

git clone [https://github.com/citusdata/pg_auto_failover.git](https://github.com/citusdata/pg_auto_failover.git)
cd pg_auto_failover
make

To see where the postgresql binaries are:

pg_config --bindir

Install the pg_auto_failover binaries to pg_config --bindir

sudo make install

Add the bins to path (including auto failover related ones):

export PATH="/usr/local/Cellar/postgresql/11.3/bin/:$PATH"

#### 3. Create the monitor (ie. a PostgreSQL and a deamon agent)

pg_autoctl create monitor --pgdata /usr/local/var/postgres_monitor --nodename localhost --pgport 5444

Output:

21:59:01 INFO  Initialising a PostgreSQL cluster at "/usr/local/var/postgres_monitor"
21:59:01 INFO /usr/local/bin/pg_ctl --pgdata /usr/local/var/postgres_monitor --options "-p 5444" --options "-h *" --wait start
21:59:01 INFO Granting connection privileges on ::1/128
21:59:01 INFO Your pg_auto_failover monitor instance is now ready on port 5444.
21:59:01 INFO pg_auto_failover monitor is ready at postgres://autoctl_node@localhost:5444/pg_auto_failover
21:59:01 INFO Monitor has been succesfully initialized.

#### 4. Create primary node (specifying the monitor)

pg_autoctl create postgres --pgdata /usr/local/var/primary --monitor postgres://autoctl_node@localhost:5444/pg_auto_failover --pgport 5432 --pgctl /usr/local/Cellar/postgresql/11.3/bin/pg_ctl

Output - you can notice the DB also gets started automatically:

22:00:55 WARN  Failed to resolve hostname from address "192.168.0.17": nodename nor servname provided, or not known
22:00:55 INFO Using local IP address "192.168.0.17" as the --nodename.
22:00:55 INFO Found pg_ctl for PostgreSQL 11.3 at /usr/local/bin/pg_ctl
22:00:55 INFO Registered node 192.168.0.17:5432 with id 1 in formation "default", group 0.
22:00:55 INFO Writing keeper init state file at "/Users/dbg/.local/share/pg_autoctl/usr/local/var/primary/pg_autoctl.init"
22:00:55 INFO Successfully registered as "single" to the monitor.
22:00:56 INFO Initialising a PostgreSQL cluster at "/usr/local/var/primary"
22:00:56 INFO Postgres is not running, starting postgres
**22:00:56 INFO /usr/local/bin/pg_ctl --pgdata /usr/local/var/primary --options "-p 5432" --options "-h *" --wait start**
22:00:56 INFO CREATE DATABASE postgres;
22:00:56 INFO The database "postgres" already exists, skipping.
22:00:56 INFO FSM transition from "init" to "single": Start as a single node
22:00:56 INFO Initialising postgres as a primary
22:00:56 INFO Transition complete: current state is now "single"
22:00:56 INFO Keeper has been succesfully initialized.
pg_autoctl run --pgdata /usr/local/var/primary
22:03:57 INFO Managing PostgreSQL installation at "/usr/local/var/primary"
22:03:57 INFO pg_autoctl service is starting
22:03:57 INFO Calling node_active for node default/1/0 with current state: single, PostgreSQL is running, sync_state is "", WAL delta is -1.
22:04:02 INFO Calling node_active for node default/1/0 with current state: single, PostgreSQL is running, sync_state is "", WAL delta is -1.

#### 5. Create and start the secondary node (also specifying the monitor)

pg_autoctl create postgres --pgdata /usr/local/var/secondary --monitor postgres://autoctl_node@localhost:5444/pg_auto_failover --pgport 5433 --pgctl /usr/local/Cellar/postgresql/11.3/bin/pg_ctl

pg_autoctl run --pgdata /usr/local/var/secondary/

* * *

#### 6. Connection string from clients to postgres is the following

pg_autoctl show uri --formation default --pgdata /usr/local/var/primary

Output:

postgres://192.168.0.17:5433,192.168.0.17:5432/postgres?target_session_attrs=read-write

The PostgreSQL client will automatically failover from 192.168.0.17:5433 to 192.168.0.17:5432 in case of the DBs is down

## Testing failover

Now with the primary and secondary running, we will test some failover scenarios.

### Testing failover 1 -- killing the secondary

Lets see what PostgreSQL processes are running locally:

[~] => ps -ef | grep postgresql

Output:

  501   642     1   0  9:59pm ??         0:00.96 /usr/local/Cellar/postgresql/11.3/bin/postgres -D /usr/local/var/postgres_monitor -p 5444 -h *  
5017529 1 0 11:56pm ?? 0:00.05 /usr/local/Cellar/postgresql/11.3/bin/postgres -D /usr/local/var/primary -p 5432 -h *
5017680 1 0 11:57pm ?? 0:00.04 /usr/local/Cellar/postgresql/11.3/bin/postgres -D /usr/local/var/secondary -p 5433 -h *
50177391123 0 11:57pm ttys0030:00.00 grep postgresql

We can notice the 3 components - monitor, primary and secondary. Let's kill the secondary:

kill -9 7680

Secondary node logs after executing kill -9:

23:58:10 INFO Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 0.
23:58:15 ERROR Failed to signal pid 7680, read from Postgres pid file.
23:58:15 INFO Is PostgreSQL at "/usr/local/var/secondary" up and running?
23:58:15 INFO Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is not running, sync_state is "", WAL delta is -1.
23:58:15 INFO Postgres is not running, starting postgres
23:58:15 INFO /usr/local/Cellar/postgresql/11.3/bin/pg_ctl --pgdata /usr/local/var/secondary --options "-p 5433" --options "-h *" --wait start
23:58:15 ERROR Failed to signal pid 7680, read from Postgres pid file.
23:58:15 INFOIs PostgreSQL at "/usr/local/var/secondary" up and running?
23:58:15 ERROR Failed to get Postgres pid, see above for details
23:58:15 ERROR Given --pgport 5433 doesn't match PostgreSQL port 0 from "/usr/local/var/secondary/postmaster.pid"
23:58:15 FATAL Failed to discover PostgreSQL setup, please fix previous errors.
23:58:15 ERROR Failed to restart PostgreSQL, see PostgreSQL logs for instance at "/usr/local/var/secondary".
23:58:15 WARNpg_autoctl failed to ensure current state "secondary": PostgreSQL is not running
23:58:20 INFO Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 589336.
23:58:20 INFO FSM transition from "secondary" to "catchingup": Failed to report back to the monitor, not eligible for promotion
23:58:20 INFO Transition complete: current state is now "catchingup"
23:58:20 INFO Calling node_active for node default/2/0 with current state: catchingup, PostgreSQL is running, sync_state is "", WAL delta is -1.
23:58:20 INFO FSM transition from "catchingup" to "secondary": Convinced the monitor that I'm up and running, and eligible for promotion again
23:58:20 INFO Transition complete: current state is now "secondary"
23:58:20 INFO Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 589336.
23:58:25 INFO Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 589336.
23:58:30 INFO Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 589336.
23:58:35 INFO Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 589336.

We notice that:

1. The secondary node was nicely getting the data from the primary (no WAL lag)
2. We then killed it and the pg_auto_ctl saw it was down
3. pg_auto_ctl restarted the PostgreSQL server by re-running the server
4. we can notice that the FSM (finite state machine) state was "catching-up" and later it was back to "secondary" once it caught up with the streaming replication from the primary.

Simultaneous logs on the primary after we killed the secondary node:

23:58:05 INFO  Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
23:58:10 INFO Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
23:58:15 INFO Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
23:58:15 INFO FSM transition from "primary" to "wait_primary": Secondary became unhealthy
**23:58:15 INFO Disabling synchronous replication**
23:58:15 INFO Transition complete: current state is now "wait_primary"
23:58:15 INFO Calling node_active for node default/1/0 with current state: wait_primary, PostgreSQL is running, sync_state is "async", WAL delta is 0.
23:58:20 INFO Calling node_active for node default/1/0 with current state: wait_primary, PostgreSQL is running, sync_state is "async", WAL delta is 0.
23:58:20 INFO FSM transition from "wait_primary" to "primary": A healthy secondary appeared
**23:58:20 INFO Enabling synchronous replication**
23:58:20 INFO Transition complete: current state is now "primary"
23:58:20 INFO Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
23:58:25 INFO Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
23:58:30 INFO Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
23:58:35 INFO Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
23:58:40 INFO Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
23:58:45 INFO Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
The log output here shows us why pg_auto_failover is a great failover solution.

It saw that the secondary was down, and instead of continuing **synchronous** streaming replication it switched to **asynchronous.** This means commits can still be done on the primary node, otherwise the primary would stop accepting writes (since they couldn't be synced ie. commited at the same time on the secondary).

### Test failover 2 -- killing the primary

* * *

[~] => ps -ef | grep postgresql

Output:

501   642     1   0  9:59pm ??         0:01.00 /usr/local/Cellar/postgresql/11.3/bin/postgres -D /usr/local/var/postgres_monitor -p 5444 -h *
501  7529     1   0 11:56pm ??         0:00.08 /usr/local/Cellar/postgresql/11.3/bin/postgres -D /usr/local/var/primary -p 5432 -h *
501  7773     1   0 11:58pm ??         0:00.05 /usr/local/Cellar/postgresql/11.3/bin/postgres -D /usr/local/var/secondary -p 5433 -h *
501  8042  1123   0 12:01am ttys003    0:00.00 grep postgresql

[~] => kill -9 7529

Simultaneous logs on the secondary after we killed the primary node:

00:01:06 INFO  Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 589336.
00:01:11 INFO  Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 589336.
**00:01:16 ERROR PostgreSQL cannot reach the primary server: the system view pg_stat_wal_receiver has no rows.**
00:01:16 INFO  Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is -1.
00:01:21 INFO  Calling node_active for node default/2/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 0.
**00:01:21 INFO  FSM transition from "secondary" to "prepare_promotion": Stop traffic to primary, wait for it to finish draining.**
**00:01:21 INFO  Transition complete: current state is now "prepare_promotion"**
00:01:21 INFO  Calling node_active for node default/2/0 with current state: prepare_promotion, PostgreSQL is running, sync_state is "", WAL delta is -1.
**00:01:21 INFO  FSM transition from "prepare_promotion" to "stop_replication": Prevent against split-brain situations.**
**00:01:21 INFO  Prevent writes to the promoted standby while the primary is not demoted yet, by making the service incompatible with target_session_attrs = read-write**
00:01:21 INFO Setting default_transaction_read_only to on
00:01:21 INFO Promoting postgres
00:01:21 INFO Other node in the HA group is 192.168.0.17:5432
00:01:21 INFO Create replication slot "pgautofailover_standby"
**00:01:21 INFO Disabling synchronous replication**
**00:01:21 INFO Transition complete: current state is now "stop_replication"**
00:01:21 INFO Calling node_active for node default/2/0 with current state: stop_replication, PostgreSQL is running, sync_state is "", WAL delta is -1.
00:01:26 INFO Calling node_active for node default/2/0 with current state: stop_replication, PostgreSQL is running, sync_state is "", WAL delta is -1.
**00:01:26 INFO FSM transition from "stop_replication" to "wait_primary": Confirmed promotion with the monitor**
00:01:26 INFO Setting default_transaction_read_only to off
**00:01:26 INFO Transition complete: current state is now "wait_primary"**
00:01:26 INFO Calling node_active for node default/2/0 with current state: wait_primary, PostgreSQL is running, sync_state is "", WAL delta is -1.
00:01:31 INFO Calling node_active for node default/2/0 with current state: wait_primary, PostgreSQL is running, sync_state is "async", WAL delta is 0.
**00:01:31 INFO FSM transition from "wait_primary" to "primary": A healthy secondary appeared**
00:01:31 INFO  Enabling synchronous replication
**00:01:31 INFO  Transition complete: current state is now "primary"**
00:01:31 INFO  Calling node_active for node default/2/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.

Logs on the primary after we killed the (primary) PostgreSQL process:

00:01:01 INFO  Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
00:01:06 INFO  Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
00:01:11 INFO  Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "sync", WAL delta is 0.
00:01:16 ERROR Failed to signal pid 7529, read from Postgres pid file.
00:01:16 INFO  Is PostgreSQL at "/usr/local/var/primary" up and running?
00:01:16 INFO  Calling node_active for node default/1/0 with current state: primary, PostgreSQL is not running, sync_state is "", WAL delta is -1.
**00:01:16 INFO  Postgres is not running, starting postgres**
00:01:16 INFO   /usr/local/bin/pg_ctl --pgdata /usr/local/var/primary --options "-p 5432" --options "-h *" --wait start
00:01:17 WARN  PostgreSQL was not running, restarted with pid 8064
00:01:22 ERROR PostgreSQL primary server has lost track of its standby: pg_stat_replication reports no client using the slot "pgautofailover_standby".
00:01:22 INFO  Calling node_active for node default/1/0 with current state: primary, PostgreSQL is running, sync_state is "", WAL delta is -1.
**00:01:22 INFO  FSM transition from "primary" to "demote_timeout": A failover occurred, no longer primary**
00:01:22 INFO  Transition complete: current state is now "demote_timeout"
00:01:22 INFO  Calling node_active for node default/1/0 with current state: demote_timeout, PostgreSQL is not running, sync_state is "", WAL delta is -1.
00:01:27 INFO  Calling node_active for node default/1/0 with current state: demote_timeout, PostgreSQL is not running, sync_state is "", WAL delta is -1.
**00:01:27 INFO  FSM transition from "demote_timeout" to "demoted": Demote timeout expired**
**00:01:27 INFO  pg_ctl: no server running**
00:01:27 INFO  pg_ctl stop failed, but PostgreSQL is not running anyway
00:01:27 INFO  Transition complete: current state is now "demoted"
00:01:27 INFO  Calling node_active for node default/1/0 with current state: demoted, PostgreSQL is not running, sync_state is "", WAL delta is -1.
**00:01:27 INFO  FSM transition from "demoted" to "catchingup": A new primary is available. First, try to rewind. If that fails, do a pg_basebackup.**
00:01:27 INFO  The primary node returned by the monitor is 192.168.0.17:5433
00:01:27 INFO  Rewinding PostgreSQL to follow new primary 192.168.0.17:5433
00:01:27 INFO  pg_ctl: no server running
00:01:27 INFO  pg_ctl stop failed, but PostgreSQL is not running anyway
**00:01:27 INFO  Running /usr/local/bin/pg_rewind --target-pgdata "/usr/local/var/primary" --source-server " host='192.168.0.17' port=5433 user='pgautofailover_replicator' dbname='postgres'" --progress ...**
00:01:28 INFO  connected to server
servers diverged at WAL location 0/3092C58 on timeline 5
rewinding from last common checkpoint at 0/308FE18 on timeline 5
reading source file list
reading target file list
reading WAL in target
need to copy 115 MB (total source directory size is 135 MB)
118766/118766 kB (100%) copied
creating backup label and updating control file
syncing target data directory
Done!
00:01:28 INFO  Writing recovery configuration to "/usr/local/var/primary/recovery.conf"
00:01:28 INFO  Postgres is not running, starting postgres
00:01:28 INFO   /usr/local/bin/pg_ctl --pgdata /usr/local/var/primary --options "-p 5432" --options "-h *" --wait start
00:01:28 INFO  Transition complete: current state is now "catchingup"
00:01:28 INFO  Calling node_active for node default/1/0 with current state: catchingup, PostgreSQL is running, sync_state is "", WAL delta is -1.
00:01:28 INFO  FSM transition from "catchingup" to "secondary": Convinced the monitor that I'm up and running, and eligible for promotion again
00:01:28 INFO  Transition complete: current state is now "secondary"
00:01:28 INFO  Calling node_active for node default/1/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 0.
00:01:33 INFO  Calling node_active for node default/1/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 0.
00:01:38 INFO  Calling node_active for node default/1/0 with current state: secondary, PostgreSQL is running, sync_state is "", WAL delta is 0.

Events:

- primary is down so writes and replication are stopped

- the secondary then gets promoted to a wait_primary (not a full primary since it doesn't have a secondary)

- in the meantime ex-primary gets back to lifeit catches up with the primary (ex-secondary) node using pg_rewind and what is especially interesting is that it would revert to pg_base_backup if it wasn't able to rewind

- the ex-primary then becomes a secondary

- the ex-secondary becomes a primary with synchronous replication
