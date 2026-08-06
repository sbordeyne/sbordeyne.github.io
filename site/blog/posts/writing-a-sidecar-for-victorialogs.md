---
tags:
  - victorialogs
  - observability
  - SRE
date: 2026-08-06
---
# Writing a sidecar for VictoriaLogs

## What is VictoriaLogs


For those who don't know, victorialogs is a log storage database made by the
same people that released [victoriametrics](https://victoriametrics.com).

Honestly I was surprised by the performance claims, and even more surprised after
deploying it to replace an aging ELK stack (Elasticsearch - Logstash - Kibana).

It proved so good, that it is now my default choice when I think "logging", as it
handles so many ingestors, has insane levels of compression, and is super snappy to
query logs. The grafana datasource is lacking a little, but is definitely useable as is.

## The problem

As it is, VictoriaLogs already provides an API to handle snapshotting, backups and restore.
The problem was that this API is not easy to use as-is. A few problems:

- You, the SRE, are expected to handle the lifecycle of the partition and the snapshot
- Snapshotted partition is copied on the local disk (in the `-storageDataDir`)
- There is no retry mechanism in place in case a snapshot fails

VictoriaLogs also advises to use this mechanism to perform 2-tier log storage,
by:

- Creating a snapshot on the "hot" instance
- Transfering (`rsync`) the created snapshot to the "cold" instance
- Restoring the snapshot on the "cold" instance
- Dettaching the partition on the "hot" instance
- Attaching the partition on the "cold" instance

This is a hell of a workflow to perform in a simple bash script. Not only that,
but when running in kubernetes, you don't have access to the underlying volume
that easily, meaning I needed something to perform that process, automatically.

## The sidecar pattern

In kubernetes, it's a common pattern: use a sidecar when you need something done
at the edge of a Pod. In my case, I need to use a sidecar to be able to share
the same volume as victorialogs, in order to access the snapshots and partitions

## VLBackup is born

I decided to call that utility `vlbackup`, for what I guess should be obvious
reasons.

VLBackup is deployed as a sidecar to a victorialogs single instance, or a vlstorage
node.

Its goal is relatively simple: expose a straightforward API that a cronjob can call
to perform actions on victorialogs, namely:

- restoring a snapshot
- snapshot a partition to object storage
- transfer a snapshot between two instances for 2-tier log storage.

The API design follows what I believe to be the best practices: reliance on open
standards (OpenAPI), generated code (note: I didn't say AI-generated) and small,
atomic endpoints. And because I'm a sucker for CI/CD and linters, I decided to
use one of the most strict linters out there: the IBM OpenAPI linter. It resulted
in these endpoints:

- `POST /v1/vlbackup/snapshot`
- `POST /v1/vlbackup/transfer`
- `POST /v1/vlbackup/migrate`
- `POST /v1/vlbackup/restore`

Just 4 endpoints that do all of the magic! Some other endpoints are also defined,
but are mostly there as internal API machinery for two sidecars to communicate
with one another.

With VLBackup, snapshotting is as easy as

```sh
curl -sL -XPOST http://victorialogs:8080/v1/vlbackup/snapshot \
    -H "Content-Type: application/json"
    -d'{"destination_url": "s3://my-bucket/victorialogs", "range": {"from": "now-1d/d", "to": "now/d"}}'
```

## It still is not user-friendly enough

Because I find `curl` to be great, but brittle, I decided to add another component
to `vlbackup`: `vlbackupctl`

It's just a CLI that performs the same API calls, but in a more user-friendly manner.

```sh
vlbackupctl snapshot --dest-url s3://my-bucket/victorialogs --from now-1d/d --to now/d
```

## Conclusion

I worked on this project to do some programming in go and to solve a real problem
I had: handling snapshotting of my preferred log database. Of course, none of
that would have been needed had I chosen another log DB. After all Grafana Loki
supports object storage out of the box making snapshots a non-issue, and ELK
has the amazing `curator` tool that makes snapshots trivial with just a little
bit of configuration.

The API design part was fun, but the most fun was seeing my first transfer jobs
work without any issues. With VLBackup, I was able to successfully implement
a 2-tier log storage system that saved the company ~800$ in persistent disk costs.