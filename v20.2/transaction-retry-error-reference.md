---
title: Transaction Retry Error Reference
summary: A list of the transaction retry (serialization) errors emitted by CockroachDB, including likely causes and user actions for mitigation.
toc: true
---

This page has a list of the transaction retry error codes emitted by CockroachDB.

Errors with the `SQLSTATE` error code `40001`, along with error messages including the string `restart transaction`, indicate that a transaction failed because it could not be placed into a serializable ordering among all of the currently-executing transactions.  This is almost always due to a conflict with another concurrent or recent transaction accessing the same data - also known as contention.  In cases of contention, the transaction needs to be retried by the client as described in [client-side intervention](#client-side-intervention).

In rare cases, transaction retry errors are not caused by contention, but by other system states.  This page attempts to gather a complete list of transaction retry error codes, both those caused by 

For each error code, we describe:

- Why this error is happening
- What to do about it

{{site.data.alerts.callout_danger}}
This page is meant to provide information about specific transaction retry error codes to make certain types of troubleshooting easier.  In _nearly all cases_, the correct action from a client application's perspective when these errors occur is:
1. Update your app to retry on serialization errors (where `SQLSTATE` is `40001`), as described in [client-side retry handling](transactions.html#client-side-intervention).
2. Design your schema and queries to reduce contention.  For more information about contention and how to avoid it, see [Understanding and avoiding transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).
{{site.data.alerts.end}}

## Overview

CockroachDB attempts to find a serializable ordering of all currently-executing transactions.

Whenever possible, CockroachDB will auto-retry a transaction when it encounters a serialization error, without the client ever knowing. CockroachDB will only send such errors to the client when it cannot resolve the error automatically without client-side intervention.

In other words, by the time these errors bubble up to the client, CockroachDB has already tried to resolve the problem internally, and could not.

The common underlying reason for this is that the SQL language is "conversational" by design. The client can send statements to the server during a transaction, receive some results, and then decide to issue other statements inside the same transaction based on the server's response.

This means that there's no way for the server to "simply retry" the arbitrary SQL statements sent so far inside the transaction, because if there are different results for a given statement than there were earlier (likely due to the operations of other, concurrent transactions operating on the same data), the client needs to know so that it can decide how to handle that situation.

## TransactionRetryWithProtoRefreshError

### Retry write too old

```
TransactionRetryWithProtoRefreshError: RestartsWriteTooOld
```

_Description_:

The `RETRY_WRITE_TOO_OLD` error occurs when a transaction _A_ tries to write to a row, but another transaction _B_ that was supposed to be serialized after _A_ (i.e., had been assigned a lower timestamp), has already written to that row, and has already committed.  This is a common error when you have too much contention in your workload.

_Action_:

1. Retry transaction _A_ as described in [client-side retry handling](transactions.html#client-side-intervention).
2. Design your schema and queries to reduce contention.  For more information about contention and how to avoid it, see [Understanding and avoiding transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).

### Retry serializable

```
TransactionRetryWithProtoRefreshError: RestartsSerializable
```

The `RETRY_SERIALIZABLE` error occurs in the following scenarios:

1. When a transaction _A_ has its timestamp moved forward (also known as _A_ being "pushed") as CockroachDB attempts to find a serializable transaction ordering. Specifically, transaction _A_ tried to write a key that transaction _B_ had read.  _B_ had already committed, but it was supposed to be serialized after _A_ (i.e., _B_ had a lower timestamp than _A_).  CockroachDB will try to serialize _A_ after _B_ by changing _A_'s timestamp, but it cannot do that when another transaction has subsequently written to some of the keys that _A_ has read and returned to the client.  When that happens, the `RETRY_SERIALIZATION` error is signalled.  The solution for this is to [add a retry loop to your application](error-handling-and-troubleshooting.html#transaction-retry-errors).

2. When a high-priority transaction _A_ does a read that runs into a write intent from another lower-priority transaction _B_.  Transaction _B_ will get this error when it tries to commit. The solution for this is to [add a retry loop to your application](error-handling-and-troubleshooting.html#transaction-retry-errors).

3. When a transaction _A_ is forced to refresh (change its timestamp) due to hitting the maximum closed timestamp interval (closed timestamps enable [Follower Reads](follower-reads.html#how-follower-reads-work)).  If this is the cause of the error, the solution is to increase the [`kv.closed_timestamp.target_duration` setting](settings.html) to a higher value.  Unfortunately, there is no indication from the `RETRY_SERIALIZATION` error code that closed timestamps are the issue.  Therefore, you may need to rule out cases 1 and 2 (or experiment with increasing the closed timestamp interval, if that is possible for your application).

### Retry async write failure

```
TransactionRetryWithProtoRefreshError: RestartsAsyncWriteFailure: ...
```

The `RETRY_ASYNC_WRITE_FAILURE` error occurs when some kind of problem with your cluster's operation occurs at the moment of a previous write in the transaction.  For example, this can happen if you have a networking partition that cuts off access to some nodes in your cluster.

Because this is due to a problem with your cluster, there is no solution from the application's point of view.  You must investigate the problems with your cluster.

### Read within uncertainty interval

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

### Retry commit deadline exceeded

```
TransactionRetryWithProtoRefreshError: TransactionPushError: ...
```

The `RETRY_COMMIT_DEADLINE_EXCEEDED` error means that the transaction timed out due to being pushed back in the transaction queue by other concurrent transactions.

TODO: find out from Paul what users can do about this

### Abort reason aborted record found

```
TransactionRetryWithProtoRefreshError:TransactionAbortedError(ABORT_REASON_ABORTED_RECORD_FOUND)
```

The `ABORT_REASON_ABORTED_RECORD_FOUND` error is caused by a write-write conflict.  It means that another transaction _B_ encountered one of our transaction _A_'s write intents, and _B_ tried to push _A_'s timestamp.  This happens in one of the following cases:

1. _B_ is a higher-priority transaction than _A_
2. _B_ thinks that _A_'s transaction coordinator node is dead, because the coordinator node hasn't heartbeated the transaction record for a few seconds.  This case indicates some trouble with the cluster - usually overload.

If you are not using high-priority transactions:

- This error means your cluster has problems.  You are likely overloading it.
- Investigate the source of the overload, and do something about it.  For more information, see [Node liveness issues](cluster-setup-troubleshooting.html#node-liveness-issues).

If you are using high-priority transactions:

1. Update your app to retry on serialization errors (where `SQLSTATE` is `40001`), as described in [client-side retry handling](transactions.html#client-side-intervention).
2. Design your schema and queries to reduce contention.  For more information about contention and how to avoid it, see [Understanding and avoiding transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).

### Abort reason client reject

```
TransactionRetryWithProtoRefreshError:TransactionAbortedError(ABORT_REASON_CLIENT_REJECT)
```

The `ABORT_REASON_CLIENT_REJECT` error means that the client application is trying to use a transaction that has been aborted.  This is usually not due to any coding error on the part of the client application.  It is usually due to the same reasons as the [abort reason aborted record found](#abort-reason-aborted-record-found) error, namely:

- Write-write conflict: Another high-priority transaction _B_ encountered a write intent by our transaction _A_, and tried to push _A_'s timestamp.
- Cluster overload: _B_ thinks that _A_'s transaction coordinator node is dead, because the coordinator node hasn't heartbeated the transaction record for a few seconds.

If you are not using high-priority transactions:

- This error means your cluster has problems.  You are likely overloading it.
- Investigate the source of the overload, and do something about it.  For more information, see [Node liveness issues](cluster-setup-troubleshooting.html#node-liveness-issues).

If you are using high-priority transactions:

1. Update your app to retry on serialization errors (where `SQLSTATE` is `40001`), as described in [client-side retry handling](transactions.html#client-side-intervention).
2. Design your schema and queries to reduce contention.  For more information about contention and how to avoid it, see [Understanding and avoiding transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).

XXX: Does this section need to have approximately all of the same content as the section for `ABORT_REASON_ABORTED_RECORD_FOUND`?  Should we say this differently, somehow?

### Abort reason pusher aborted

```
TransactionRetryWithProtoRefreshError:TransactionAbortedError(ABORT_REASON_PUSHER_ABORTED)
```

The `ABORT_REASON_PUSHER_ABORTED` error can happen when a transaction _A_ is aborted by some other concurrent transaction _B_, probably due to a deadlock.  _A_ tried to push another transaction's timestamp, but while waiting for the push to succeed, it was aborted.

If you are seeing this error:

1. Update your app to retry on serialization errors (where `SQLSTATE` is `40001`), as described in [client-side retry handling](transactions.html#client-side-intervention).
2. Design your schema and queries to reduce contention.  For more information about contention and how to avoid it, see [Understanding and avoiding transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).

XXX: is this one caused by contention?

### Abort reason abort span

```
TransactionRetryWithProtoRefreshError:TransactionAbortedError(ABORT_REASON_ABORT_SPAN)
```

_Description_:

This transaction A tried to read from a range where it had previously laid down intents that have been cleaned up in the meantime, because A was aborted.

_Action_:

- Retry transaction _A_ as described in [client-side retry handling](transactions.html#client-side-intervention).

XXX: is this one caused by contention?

### Abort reason new lease prevents txn

```
TransactionRetryWithProtoRefreshError:TransactionAbortedError(ABORT_REASON_NEW_LEASE_PREVENTS_TXN)
```

_Description_:

The `ABORT_REASON_NEW_LEASE_PREVENTS_TXN` error occurs because the timestamp cache will not allow transaction _A_ to create a transaction record.  A new lease wipes the timestamp cache, so this could mean the leaseholder was moved and the duration of transaction _A_ was unlucky enough to happen across a lease acquisition.  In other words, leaseholders got shuffled out from underneath transaction _A_ (due to no fault of the client application or schema design), and now it has to be retried.

_Action_:

- Retry transaction _A_ as described in [client-side retry handling](transactions.html#client-side-intervention).

### Abort reason timestamp cache rejected

```
TransactionRetryWithProtoRefreshError:TransactionAbortedError(ABORT_REASON_TIMESTAMP_CACHE_REJECTED)
```

_Description_:

The `ABORT_REASON_TIMESTAMP_CACHE_REJECTED` error occurs when the timestamp cache will not allow transaction _A_ to create a transaction record.  This can happen due to a [range merge](range-merges.html) happening in the background, or because the timestamp cache is an in-memory cache, and has outgrown its memory limit (about 64 MB).

_Action_:

- Retry transaction _A_ as described in [client-side retry handling](transactions.html#client-side-intervention).

## See also

- [Common Errors](common-errors.html)
- [Transactions](transactions.html)
- [Client-side retry handling](transactions.html#client-side-intervention)
- [Architecture - Transaction Layer](architecture/transaction-layer.html)
