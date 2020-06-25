---
title: Transaction Retry Error Reference
summary: A list of the transaction retry (serialization) errors emitted by CockroachDB, including likely causes and user actions for mitigation.
toc: true
---

This page has a list of the transaction retry error codes emitted by CockroachDB.

Errors with the `SQLSTATE` error code `40001` or string including `restart transaction` indicate that a transaction failed because it could not be placed into a serializable ordering among all of the currently-executing transactions.  This is almost always due to a conflict with another concurrent or recent transaction accessing the same data - also known as contention.  In cases of contention, the transaction needs to be retried by the client as described in [client-side intervention](#client-side-intervention).

In rare cases, transaction retry errors are not caused by contention, but by other system states.  This page attempts to gather a complete list of transaction retry error codes and describe:

- Why this error is happening
- What to do about it

{{site.data.alerts.callout_danger}}
This page is meant to provide information about specific transaction retry error codes to make certain types of troubleshooting easier.  In _nearly all cases_, the correct action from a client application's perspective when these errors occur is:
1. Check for the `SQLSTATE` error code `40001` and/or the error text including `restart transaction`, and retry the transaction as described in [client-side retry handling](transactions.html#client-side-intervention)
2. If adding retry handling to your application doesn't help, the next step is to review your schema and data access patterns for causes of contention as described in [Understanding and avoiding transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).
{{site.data.alerts.end}}

## Overview

CockroachDB attempts to find a serializable ordering of all currently-executing transactions.

Whenever possible, CockroachDB will try to auto-retry a transaction when it encounters a retry error, without ever bothering the client. It will only respond to the client with an error when it cannot resolve the error automatically without client-side intervention.

In other words, by the time these errors bubble up to the client, CockroachDB has already tried to resolve the problem internally, and could not.

The underlying reason for this is that the SQL language is "conversational" by design. The client can send statements to the server during a transaction, receive some results, and then decide to issue other statements inside the same transaction based on the server's response.

This means that there's no way for the server to "simply retry" the arbitrary SQL statements sent so far inside the transaction, because if there are different results for a given statement than there were earlier (likely due to the operations of other, concurrent transactions operating on the same data), the client needs to know so that it can decide how to handle that situation.

## Retry write too old

```
TransactionRetryWithProtoRefreshError: RestartsWriteTooOld
```

The `RETRY_WRITE_TOO_OLD` error occurs when a transaction _A_ tries to write to a row, but another transaction _B_ that was supposed to be serialized after _A_ (i.e., had been assigned a lower timestamp), has already written to that row, and has already committed.

This is a common error when you have too much contention in your workload.   The solution is to [add a retry loop to your application](error-handling-and-troubleshooting.html#transaction-retry-errors).

For more information about the types and causes of contention, and how to mitigate them, see [Understanding and avoiding transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).

## Retry serializable

```
TransactionRetryWithProtoRefreshError: RestartsSerializable
```

The `RETRY_SERIALIZABLE` error occurs in the following scenarios:

1. When a transaction _A_ has its timestamp moved forward (also known as _A_ being "pushed") as CockroachDB attempts to find a serializable transaction ordering. Specifically, transaction _A_ tried to write a key that transaction _B_ had read.  _B_ had already committed, but it was supposed to be serialized after _A_ (i.e., _B_ had a lower timestamp than _A_).  CockroachDB will try to serialize _A_ after _B_ by changing _A_'s timestamp, but it cannot do that when another transaction has subsequently written to some of the keys that _A_ has read and returned to the client.  When that happens, the `RETRY_SERIALIZATION` error is signalled.  The solution for this is to [add a retry loop to your application](error-handling-and-troubleshooting.html#transaction-retry-errors).

2. When a high-priority transaction _A_ does a read that runs into a write intent from another lower-priority transaction _B_.  Transaction _B_ will get this error when it tries to commit. The solution for this is to [add a retry loop to your application](error-handling-and-troubleshooting.html#transaction-retry-errors).

3. When a transaction _A_ is forced to refresh (change its timestamp) due to hitting the maximum closed timestamp interval (closed timestamps enable [Follower Reads](follower-reads.html#how-follower-reads-work)).  If this is the cause of the error, the solution is to increase the [`kv.closed_timestamp.target_duration` setting](settings.html) to a higher value.  Unfortunately, there is no indication from the `RETRY_SERIALIZATION` error code that closed timestamps are the issue.  Therefore, you may need to rule out cases 1 and 2 (or experiment with increasing the closed timestamp interval, if that is possible for your application).

## Retry async write failure

```
TransactionRetryWithProtoRefreshError: RestartsAsyncWriteFailure: ...
```

The `RETRY_ASYNC_WRITE_FAILURE` error occurs when some kind of problem with your cluster's operation occurs at the moment of a previous write in the transaction.  For example, this can happen if you have a networking partition that cuts off access to some nodes in your cluster.

Because this is due to a problem with your cluster, there is no solution from the application's point of view.  You must investigate the problems with your cluster.

## Read within uncertainty interval

```
TransactionRetryWithProtoRefreshError: ReadWithinUncertaintyIntervalError: 
        read at time 1591009232.376925064,0 encountered previous write with future timestamp 1591009232.493830170,0 within uncertainty interval `t <= 1591009232.587671686,0`;
        observed timestamps: [{1 1591009232.587671686,0} {5 1591009232.376925064,0}] 
```

The `ReadWithinUncertaintyIntervalError` can occur during the first [`SELECT`](select-clause.html) statement in any transaction.  This behavior is non-deterministic: it depends on which node is the leaseholder of the underlying data range; it’s generally a sign of contention. Uncertainty errors are always possible with near-realtime reads under contention.

The solution is to do one of the following:

1. Be prepared to retry on uncertainty (and other) errors, as described in [client-side retry handling](transactions.html#client-side-intervention).
2. Use historical reads with [`SELECT ... AS OF SYSTEM TIME`](as-of-system-time.html)
3. Design your schema and queries to reduce contention.  For more information about contention and how to avoid it, see [Understanding and avoiding transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).
4. If you trust your clocks, you can try lowering the [maximum clock offset setting](cockroach-start.html#flags-max-offset).

## Retry commit deadline exceeded

```
TransactionRetryWithProtoRefreshError: TransactionPushError: ...
```

The `RETRY_COMMIT_DEADLINE_EXCEEDED` error ... XXX

## See also

- [Common Errors](common-errors.html)
- [Transactions](transactions.html)
- [Client-side retry handling](transactions.html#client-side-intervention)
- [Architecture - Transaction Layer](architecture/transaction-layer.html)

