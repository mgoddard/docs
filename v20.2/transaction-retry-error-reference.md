---
title: Transaction Retry Error Reference
summary: A list of the transaction retry (serialization) errors emitted by CockroachDB, including likely causes and user actions for mitigation.
toc: true
---

Errors with the code `40001` or string `retry transaction` indicate that a transaction failed because it could not be placed into a serializable ordering by CockroachDB.  This is usually due to contention: a conflict with another concurrent or recent transaction accessing the same data; in such cases, the transaction needs to be retried by the client as described in [client-side intervention](#client-side-intervention).  For a list of the types of transaction retry errors emitted by CockroachDB, including information about what causes each type of error, and how to respond to them, see [Transaction retry error reference](transaction-retry-error-reference.html).

This page has a list of the retry error codes emitted by CockroachDB.

## Overview

CockroachDB attempts to find a serializable ordering of transactions.

In all cases where it can, CRDB tries to "refresh" or "auto-retry" a transaction internally, if it can, before it signals an error to the client.  By the time an error listed here is returned to the client, CRDB has already tried to resolve the problem internally, but it could not.  CockroachDB cannot retry automatically in cases where some results have already been returned to the client.

The underlying reason for this is that the SQL language is "conversational" - the client can send statements to the server during a transaction, receive some results, and then decide to issue other statements inside the same transaction based on the server's response.

There's no way for the server to "simply retry" the stuff done so far, because if there are different results than the first time, the client needs to know so that it can decide how to handle that situation.

## RETRY_WRITE_TOO_OLD

The `RETRY_WRITE_TOO_OLD` error occurs when a transaction _A_ tries to write to a row, but another transaction _B_ that was supposed to be serialized after _A_ (i.e., had been assigned a lower timestamp), has already written to that row, and has already committed.

This is a common error when you have too much contention in your workload.   The solution is to [add a retry loop to your application](error-handling-and-troubleshooting.html#transaction-retry-errors).

For more information about the types and causes of contention, and how to mitigate them, see [Understanding and avoiding transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).

## RETRY_SERIALIZATION

The `RETRY_SERIALIZATION` error occurs in the following scenarios:

1. When a transaction _A_ has its timestamp moved forward (also known as _A_ being "pushed") as CockroachDB attempts to find a serializable transaction ordering. Specifically, transaction _A_ tried to write a key that transaction _B_ had read.  _B_ had already committed, but it was supposed to be serialized after _A_ (i.e., _B_ had a lower timestamp than _A_).  CockroachDB will try to serialize _A_ after _B_ by changing _A_'s timestamp, but it cannot do that when another transaction has subsequently written to some of the keys that _A_ has read and returned to the client.  When that happens, the `RETRY_SERIALIZATION` error is signalled.  The solution for this is to [add a retry loop to your application](error-handling-and-troubleshooting.html#transaction-retry-errors).

2. When a high-priority transaction _A_ does a read that runs into a write intent from another lower-priority transaction _B_.  Transaction _B_ will get this error when it tries to commit. The solution for this is to [add a retry loop to your application](error-handling-and-troubleshooting.html#transaction-retry-errors).

3. When a transaction _A_ is forced to refresh (change its timestamp) due to hitting the maximum closed timestamp interval (closed timestamps enable [Follower Reads](follower-reads.html#how-follower-reads-work)).  If this is the cause of the error, the solution is to increase the [`kv.closed_timestamp.target_duration` setting](settings.html) to a higher value.  Unfortunately, there is no indication from the `RETRY_SERIALIZATION` error code that closed timestamps are the issue.  Therefore, you may need to rule out cases 1 and 2 (or experiment with increasing the closed timestamp interval, if that is possible for your application).

## RETRY_ASYNC_WRITE_FAILURE

The `RETRY_ASYNC_WRITE_FAILURE` error occurs when some kind of problem with your cluster's operation occurs at the moment of a previous write in the transaction.  For example, this can happen if you have a networking partition that cuts off access to some nodes in your cluster.

Because this is due to a problem with your cluster, there is no solution from the application's point of view.  You must investigate the problems with your cluster.

## ABORT_REASON_ALREADY_COMMITTED_OR_ROLLED_BACK_POSSIBLE_REPLAY

The `ABORT_REASON_ALREADY_COMMITTED_OR_ROLLED_BACK_POSSIBLE_REPLAY` error is signalled when there are issues at the [Storage Layer](architecture/storage-layer.html) of CockroachDB.

This error should only appear wrapped in an `AmbiguousError`.  For more information about what ambiguous errors are and how to handle them, see [`AmbiguousError`](XXX).

## ReadWithinUncertaintyIntervalError

The `ReadWithinUncertaintyIntervalError` can occur during the first [`SELECT`](xxx) statement in any transaction.

It depends on which node is the leaseholder of the underlying data range. It’s generally a sign of contention. Specifically, as noted below, uncertainty errors are always possible with near-realtime reads under contention.

This behavior is non-deterministic - it depends on which node is the leader of the underlying data [range](xxx).  Uncertainty errors are always possible with near-realtime reads under contention. The solution is to do one of the following:

1. Be prepared to retry on uncertainty (and other) errors, as described in [transaction retry errors](error-handling-and-troubleshooting.html#transaction-retry-errors).
2. Use historical reads with [`SELECT ... AS OF SYSTEM TIME`](as-of-system-time.html)
3. Design your schema and queries to reduce contention.  For more information about contention and how to avoid it, see [Understanding and avoiding transaction contention](performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).
4. If you trust your clocks, you can lower the [maximum clock offset](xxx).

## See also

- [Common Errors](common-errors.html)
- [Transactions](transactions.html)
- [Architecture - Transaction Layer](architecture/transaction-layer.html)

