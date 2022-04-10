---
title : "Live Read Replication"
date: 2022-04-09T00:00:00Z
layout: docs
menu:
  docs:
    parent: "replica-guides"
weight: 460
---

This guide will show you how to run Litestream with a primary server that
replicates to read-only replicas in real-time.

{{< alert icon="❗️" text="This feature is currently in beta" >}}


## Overview

The live read replication feature of Litestream allows a user to make changes
to a single, primary database and have those changes made available to stream
to one or more read-only replicas. These replicas will pull the change, apply
it to their local copy of the database in a transactionally-safe manner, and
make that data available to the replica's application immediately.

This feature provides the ability to scale SQLite read queries across many
servers as well as provide geographic replication to improve network latency.


## Configuration

### Configuring the primary

To enable replication on the primary server, simply enable the HTTP server by
setting the `addr` flag in the configuration file:

```yml
addr: ":9090"

dbs:
  - path: /data/db
```

### Configuring the replicas

On each of the replica servers, you'll need to set the `upstream` property of
each database that will be replicated:

```yml
addr: ":9090"

dbs:
  - path: /data/db
    upstream:
      url: http://$PRIMARY_HOSTNAME:9090
```

If the path of the replica's database is different than the primary's database
then you can specify `upstream.path` as the primary's database path.


## Application changes

### Ensuring replicas are read-only

Read replication uses physical replication so it simply copies database pages
from one database to another. However, only the primary node is allowed to write
to the database. Writing to the replicas will corrupt their copy of the database.

Because Litestream cannot control whether your application writes to the
database, it is your responsibility to only perform reads (i.e. `SELECT`). It
is recommended that you set SQLite in a read-only mode to ensure that it does
not change. You can do this by passing the URL-style parameter to SQLite in
most driver implementations:

```
/path/to/db?mode=ro
```

If you are using the SQLite API directly, you can pass the `SQLITE_OPEN_READONLY`
flag to [`sqlite3_open_v2()`](https://www.sqlite.org/c3ref/open.html).

If you are using the SQLite CLI, you can pass the `-readonly` flag:

```sh
$ sqlite3 -readonly /path/to/db
```


### Redirect writes to the primary

Since replicas are read-only, it is the responsibility of the application
developer to redirect writes to the primary node. The exact implementation of
this depends on your deployment.

Some hosting providers provide a means to redirect requests. For example,
[fly.io](fly.io) provides a [`fly-replay`](https://fly.io/docs/getting-started/multi-region-databases/#replay-the-request)
header to automatically replay a request to a different node. Your load
balancer may provide a similar mechanism. If not, you can also have your
application proxy the request to the primary server.


