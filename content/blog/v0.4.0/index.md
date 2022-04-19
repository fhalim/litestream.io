---
title: "Litestream v0.4.0 Released"
description: "The newest version of Litestream includes support for live read replication, improved test coverage, revamped WAL storage, and more."
date: 2022-04-19T00:00:00Z
draft: false
weight: 50
contributors: ["Ben Johnson"]
---

The main focus of this latest release has been on implementing live read
replication and underlying changes to support this. We've also significantly
expanded test coverage as well as added quality-of-life changes.


## Live Read Replication

By far, the most requested feature in Litestream has been
[live read replication](https://github.com/benbjohnson/litestream/issues/8).
The idea with this feature is that you can have a single primary server that
runs SQLite and it can immediately fan out changes to one or more read-only
replicas. It provides both read scalability as well as the ability to move data
out to the edge and provide low latency responses to anyone in the world.

You can try it out with the [Litestream Read Replica Demo](https://litestream-read-replica-demo.fly.dev/)
which has a single primary node in Chicago (`ord`) that immediately streams
out changes to 15 other regions worldwide.

This feature is still considered "beta" but if you want to try it out, we have
an [example repository](https://github.com/benbjohnson/litestream-read-replica-example)
for deploying a Docker image to [fly.io](https://fly.io/).

### File Watchers

Previously, the WAL file was polled every second by Litestream to detect changes.
This worked fine when changes were replicated to S3 every second, however, we
needed lower latency for live replication. To do this, we've changed from
polling to using a file watcher. We use `inotify` for Linux and `kqueue` for
BSD-based systems.

This change allows us to immediately receive change notifications, however,
we still batch up changes every 10 milliseconds to avoid an explosion of small
WAL segment files on disk. You can adjust this threshold with the
`monitor-delay-interval` database configuration setting.

In addition to lower latency, this change also makes it more practical to
replicate a larger number of databases at a time.


### Shadow WAL Changes

The shadow WAL used to store all WAL segments in a single directory which worked
well when we only generated one WAL segment per second. However, the lower
latency file watchers can now generate up to 100 segment files per second so a
single directory was impractical.

The shadow WAL and the replica WAL layout are now grouped in directories by WAL
index. This makes it easier to seek to a given set of WAL files and reduces the
number of files per directory. You will not need to adjust settings when you 
upgrade from v0.3.x to v0.4.0, however, you will not be able to restore WAL
files from the old WAL layout when using the new version.


## Updated Contribution Policy

Litestream originally started off with a strict "no contribution" policy in
order to limit maintainer burn out. However, that turned out to be a bit excessive
as it caused folks to open issues for simple changes like typos. The
contribution policy has been updated so that minor bug fixes will now be
accepted, however, we're still not accepting code for new features. If you have
an idea for a feature, please open an issue to discuss.


## Improved test coverage & CI

Much of the testing done in v0.3.x was unit testing and some integration tests
for the replica clients. In v0.4.0, we've added significantly more unit test
coverage and as well as functional testing against the `litestream` binary itself.

We've also built out the CI pipeline to not only run these tests but it
also generates release artifacts on every PR. This includes binaries uploaded
as GitHub artifacts as well as Docker images that are tagged with the long SHA,
short SHA, and the PR number. It makes it much easier for folks to test new
features before they get included in an official release.


## Removing Windows Support

Windows support was an early request for Litestream and we spent quite a bit of
time implementing it. However, we've found limited usage of the Windows binaries
and we lack Windows expertise so we are dropping support in v0.4.0.

We may reconsider re-adding Windows support in the future though.


## Passing database path to child process

Tobi Lütke had [a great suggestion](https://twitter.com/tobi/status/1401940776465190915)
of passing through the database path to the child process when using the `-exec`
flag. Litestream now sets the `LITESTREAM_DB_PATH` environment variable on the
child process so you don't need to duplicate your database path to both
Litestream and your application. You can see more details on [issue #216](https://github.com/benbjohnson/litestream/issues/216).


## Updating Google Cloud Storage Prefix

[fekbor](https://github.com/febkor) made a good point that Google Cloud Storage uses `"gs"` as their URL
scheme instead of `"gcs"` so we've changed all reference to use "gs". You can
find more details on [issue #336](https://github.com/benbjohnson/litestream/issues/336).


## Miscellaneous fixes

Several quality of life fixes and bugs were resolved:

- The `replicate` command can now enable the web server with the `-addr` flag ([#243](https://github.com/benbjohnson/litestream/issues/243))
- The `litestream.yml` config file will be read from the present working directory ([#282](https://github.com/benbjohnson/litestream/issues/282))
- Updated SQLite driver to use `SQLITE_FCNTL_PERSIST_WAL` to persist WAL after close ([#287](https://github.com/benbjohnson/litestream/pull/287))
- Fix WAL overrun validation bug ([#303](https://github.com/benbjohnson/litestream/pull/303))
- Prevent double-close for SFTP client ([#268](https://github.com/benbjohnson/litestream/issues/268))

## Conclusion

We think there's a lot of potential for improving application latency by using
the live read replicas in v0.4.0. We're also working on some new designs for
SQLite replication that we hope to write about in the coming months. If you're
interested in hearing more, please [join our Slack][slack] or [follow Litestream
on Twitter][twitter] for updates.

[slack]: https://join.slack.com/t/litestream/shared_invite/zt-n0j4s3ci-lx1JziR3bV6L2NMF723H3Q
[twitter]: https://twitter.com/litestreamio
