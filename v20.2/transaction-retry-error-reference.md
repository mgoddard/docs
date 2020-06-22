---
title: Transaction Retry Error Reference
summary: A list of the transaction retry (serialization) errors emitted by CockroachDB, including likely causes and user actions for mitigation.
toc: true
---

Errors with the code `40001` or string `retry transaction` indicate that a transaction failed because it could not be placed into a serializable ordering by CockroachDB.  This is usually due to contention: a conflict with another concurrent or recent transaction accessing the same data; in such cases, the transaction needs to be retried by the client as described in [client-side intervention](#client-side-intervention).  For a list of the types of transaction retry errors emitted by CockroachDB, including information about what causes each type of error, and how to respond to them, see [Transaction retry error reference](transaction-retry-error-reference.html).

This page has a list of the retry error codes emitted by CockroachDB.
