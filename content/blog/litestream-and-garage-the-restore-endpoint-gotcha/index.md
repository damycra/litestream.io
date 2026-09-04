---
title: "Litestream and Garage: The Restore Endpoint Gotcha"
description: "A step-by-step Litestream setup against Garage, self-hosted S3-compatible storage, plus the one config detail that makes replication look healthy while restore silently talks to AWS instead."
date: 2026-09-04T00:00:00Z
draft: false
weight: 40
contributors: ["damycra"]
---

_tl;dr—Litestream against [Garage](https://garagehq.deuxfleurs.fr/), a self-hosted
S3-compatible object store, works well and is a good fit for a home lab or small cluster.
But `litestream restore <replica URL>` and `litestream replicate -config litestream.yml`
read their S3 settings from different places. If your `endpoint:` only lives in the config
file, replication looks perfectly healthy right up until the day you actually need to
restore — and then it quietly reaches out to AWS instead of your Garage node. This post
walks through a working setup and the one line that avoids the trap._

## Why Garage

[Garage](https://garagehq.deuxfleurs.fr/) is a small, self-hosted, S3-compatible object
store built for exactly this kind of use case: a single node (or a handful) storing backups
without needing a third-party cloud account. It speaks enough of the S3 API for Litestream
to work against it the same way it would against MinIO or AWS itself.

## A working `docker compose` setup

The full stack is three services: Garage itself, a one-shot init container that creates
the bucket and access key, and the application container that restores on boot and then
replicates while running. This is the container-entrypoint pattern the rest of these docs
recommend — `restore -if-replica-exists` before startup, `replicate -exec` after.

```yaml
# docker-compose.yml (trimmed)
services:
  garage:
    image: dxflrs/garage:v2.3.0
    environment:
      GARAGE_RPC_SECRET: <32 bytes of hex>
      GARAGE_KEY_ID: GK00112233445566778899aabb
      GARAGE_SECRET_KEY: <secret key>
    volumes:
      - ./garage.toml:/etc/garage.toml:ro
      - garage-meta:/var/lib/garage/meta
      - garage-data:/var/lib/garage/data

  app:
    build: .
    environment:
      LITESTREAM_ACCESS_KEY_ID: GK00112233445566778899aabb
      LITESTREAM_SECRET_ACCESS_KEY: <secret key>
      REPLICA_URL: s3://mybucket.garage:3900/db
    volumes:
      - app-data:/data
```

And the entrypoint script, doing exactly what these docs recommend:

```sh
#!/bin/sh
set -eu
DB=/data/app.sqlite3

if [ ! -f "$DB" ]; then
  litestream restore -if-replica-exists -o "$DB" "${REPLICA_URL}/app.sqlite3"
fi

exec litestream replicate -config /etc/litestream.yml -exec "/run-app"
```

```yaml
# litestream.yml
dbs:
  - path: /data/app.sqlite3
    replica:
      url: ${REPLICA_URL}/app.sqlite3
      endpoint: http://garage:3900
```

Bring it up, and replication works exactly as expected:

```
time=... level=INFO msg="replicating to" type=s3 sync-interval=1s bucket=mybucket
  path=db/app.sqlite3 endpoint=http://garage:3900
```

Objects show up in the bucket. `litestream databases` reports the right URL. Everything
about this setup looks correct — because it is correct, for replication.

## The gotcha: restore doesn't read the same settings

The entrypoint above restores **before** the database file exists, so it can't be looked
up by `path:` in the config file — there's nothing at that path yet for `Config.DBConfig`
to match against. That forces restore-by-URL, and `litestream restore <replica URL>`
accepts _either_ a replica URL _or_ `-config`, never both. Given a bare URL, it builds its
S3 client from the URL alone: no config file is read, so an `endpoint:` that lives only in
`litestream.yml` — like the one above — is never consulted.

Here's what that actually looks like, reproduced against the current release
(Litestream v0.5.17) with `endpoint:` set only in the config file and the restore URL left
as a plain `s3://mybucket/db/app.sqlite3.db`:

```
run: ---------------- RESTORE ----------------
run: restoring from s3://mybucket/db/app.sqlite3
Error: created at: s3: cannot lookup bucket region: operation error S3:
  GetBucketLocation, https response error StatusCode: 403, RequestID: FQPV8MS8M50PGGWX,
  ... api error InvalidAccessKeyId: The AWS Access Key Id you provided does not exist in
  our records.
run: RESTORE FAILED — read the error above for which host it dialled
run: ---------------- REPLICATE ----------------
...
writer: tables carried over from previous runs:
writer: (none above means nothing was restored)
```

An AWS request ID, from a stack that has no AWS in it anywhere, is the tell. Restore went
straight past Garage and tried to talk to `s3.amazonaws.com`, using the bucket name as-is.

**This is the loud version of the failure, and it's the lucky one.** It only surfaced
because the fake local credentials in this example are meaningless to AWS. If your
environment also happens to have real AWS credentials in it — common enough, since plenty
of setups export `AWS_ACCESS_KEY_ID` globally — and a bucket of that name exists and is
readable, the request can succeed against AWS, find nothing, and `-if-replica-exists` turns
that into an exit-0 "no matching backups found." The entrypoint then quietly initializes an
empty database, indistinguishable from a genuine first boot. That's the case worth
worrying about: nothing crashes, nothing pages you, and you find out later that the "restored"
database was empty the whole time.

## The fix: put the endpoint in the restore URL

The config file's `endpoint:` is invisible to restore-by-URL, so the endpoint has to travel
in the URL itself. Two equivalent ways to do it:

**Host form** — works well when your code builds the base URL once and appends filenames
to it, since it keeps the query string free:

```sh
REPLICA_URL=s3://mybucket.garage:3900/db
```

`mybucket.garage:3900` is parsed as bucket `mybucket`, endpoint `http://garage:3900`.

**Query form** — keeps the scheme (so it also works for TLS-terminating endpoints the host
form can't express), but put it on the base URL, not after appending a path segment:

```sh
REPLICA_URL='s3://mybucket/db?endpoint=http://garage:3900'
```

Re-run the same restore with the endpoint moved into the URL, and it recovers the prior
run's data instead of quietly starting empty:

```
run: ---------------- RESTORE ----------------
run: restoring from s3://mybucket.garage:3900/db/app.sqlite3
run: RESTORED
run: ---------------- REPLICATE ----------------
...
writer: tables carried over from previous runs:
writer:   t_95265e5c
```

## One more wrinkle if your bucket name contains a dot

Litestream recognizes the host form (`bucket.host:port`) as a self-hosted endpoint rather
than a literal AWS bucket name using the presence of a port number — a `.com` check used
to disable this detection for any host containing that substring, which broke both
self-hosted endpoints on a real `.com` domain (`minio.example.com:9000`) and, more subtly,
any bucket name that merely _contains_ the letters "com" after a dot — a company-name
bucket like `my.company` is enough, since `.company` contains the substring `.com`. That
heuristic has since been tightened to check known cloud providers explicitly instead of
guessing from `.com`, but it's still a good reason to prefer the `?endpoint=` query form if
your bucket name has a dot in it — the host form always treats everything before the first
dot as the bucket name, so a bucket name containing one will still be split incorrectly.

## Summary

- Garage works fine with Litestream — treat it like any other S3-compatible endpoint.
- `replicate -config` and `restore <replica URL>` do not read the same settings. If your
  entrypoint restores by URL (because the database doesn't exist yet to look it up by
  `path:`), the endpoint has to be in that URL, not only in `litestream.yml`.
- Prefer `?endpoint=` when you're building the whole URL in one piece; use the
  `bucket.host:port` host form when your code appends a filename to a base URL and a query
  string would be inconvenient — just avoid it for bucket names containing a dot.
- Don't trust `-if-replica-exists` exiting 0 as proof a restore actually found your data.
  Check what it dialled.

See the [S3-Compatible Services guide](/guides/s3-compatible/#restore-ignores-the-config-files-endpoint)
for the full reference version of this gotcha, and the [Garage section](/guides/s3-compatible/#garage)
of the same page for the plain configuration reference.
