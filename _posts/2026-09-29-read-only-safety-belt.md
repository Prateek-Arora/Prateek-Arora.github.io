---
title: "A read-only safety belt that breaks your app's writes"
description: "Through PgBouncer in transaction mode, a monitoring tool's read-only setting lands on the next client's connection."
image: /assets/og/read-only-safety-belt.png
pglens: true
tags: [postgresql]
---
The setting I added so my tool could never write to your database turned out to break your app's
writes instead.

The tool is [PgLens](https://github.com/Prateek-Arora/pglens), a Postgres index advisor I'm
building. It only needs to read, so it logs in as a role with no write grants, and it used to start
every connection with:

```sql
SET SESSION CHARACTERISTICS AS TRANSACTION READ ONLY;
```

Before the first release I wanted to see what happens if someone points it at a connection pooler,
like Supabase's port 6543 or a Neon `-pooler` host. 
So I tried it with PgBouncer in transaction mode. 
PgLens connected, then the application connected and tried to write:

```
CREATE TABLE orders_archive (id int);
ERROR:  cannot execute CREATE TABLE in a read-only transaction
```

The app did nothing wrong. It just got the connection PgLens had used a moment earlier.
Not great for a tool whose whole pitch is that it's safe to point at production.

<figure>
<video controls muted playsinline preload="none" poster="/assets/video/pgbouncer-read-only-leak.jpg" src="/assets/video/pgbouncer-read-only-leak.mp4"></video>
<figcaption>The leak and the fix, on PgBouncer in transaction mode with a pool of one server connection.</figcaption>
</figure>

## Why

A transaction pooler hands the same server connection to the next client.
Session settings stay on that connection, so my `SET` went with it.

PgBouncer's docs do say session `SET` doesn't work in transaction mode.
I read that as: your setting might not stick.
It sticks, just for the wrong client.

## The fix

Don't set anything on the session. Pass the guards as startup options instead, which last only as
long as that backend:

```
options=-c default_transaction_read_only=on -c statement_timeout=30s -c lock_timeout=5s
```

PgBouncer refuses them (`unsupported startup parameter in options`), which is what I want:
PgLens stops and tells you to connect directly.

Since poolers can be configured to ignore that,
PgLens also checks on connect that the backend pid stays the same and read-only is on.
An integration test runs it against a real PgBouncer.

## If you build tools that connect to other people's databases

- Don't `SET` anything on a connection unless you know it goes straight to Postgres.
- Put guards in startup options, and check they took effect.
- Test against a transaction-mode pooler. Some of your users are behind one.

The alternatives I rejected are in
[ADR-0053](https://github.com/Prateek-Arora/pglens/blob/main/docs/decisions.md#adr-0053).
