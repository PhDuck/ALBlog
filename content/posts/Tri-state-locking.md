---
title: "Tri-state locking"
date: 2023-09-05
draft: false
---

With the introduction of version 23 of Business Central we are preparing for a substantial change to how the runtime implicitly takes locks.
The change will initially be introduced as an opt-in experience for tenants from before v23.
For the foreseeable future it will remain possible to opt-out, if any problems are experienced, while the long-term plan is to have the new behaviour be the default at some point.

## Goals and reasoning
When investigating performance problems in Business Central the reality quickly becomes that interactions with the database is almost always the most expensive part of any operations. Consequently, our partners have become specialists in SQL Server. As soon as one moves into the realm of multi-user setup, it becomes even more import due to waiting for locks or experiencing lock timeouts.

While we do not know how much time is spent waiting for locks, we know that our SaaS customers in aggregate experience tens of thousands of errors per day due to having waited too long on acquiring a lock.
This information together with experience from our and external AL developers lead us to the decision that the implicit locking behaviour is too aggressive.

## Two-state locking
Before we can explore the (near) future, let reiterate the current behaviour.

The two-state locking protocol has previously been explored in-depth in my previous [post](https://bcinternals.com/posts/transactions-and-locking/), however a brief reminder seems necessary.
Without enabling tri-state locking AL code follows a two-state locking protocol:
1. All reads have the [READUNCOMMITTED](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16#readuncommitted) hint applied as long as no writes has been done to the table in the current transaction nor `LockTable` has been called on a record of the table type.
2. If writes have been done against the table (or `LockTable` called on a record of the same table) in the current transaction, further reads will have the [UPDLOCK](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16#updlock) hint applied.

This locking protocol offers a mix of performance and consistency depending on the state of the table in the transaction. Reads against a table not yet written to, are not blocked by other writes, while tables written to maintain a higher consistency guarantees disallow other session to modify them.
```AL
trigger OnAction()
var
    curr1: Record Currency;
    curr2: Record Currency;
begin
    curr1.FindFirst(); // READUNCOMMITTED

    curr1.Code := 'BTC';
    curr1."ISO Code" := 'XBT';
    curr1.Symbol := '₿';
    curr1.Insert();

    curr2.FindLast(); // UPDLOCK
end;
```
##### Listing 1: Two-state locking shown in AL with SQL table hint annotated as a comment.

The idea that writes should affect subsequent reads, is based on a wished higher consistency guarantee, which allows for later modifications of the row without worry others could have modified the records in-between, or simply as selecting the rows for a future update.

Analysing existing AL code it is clear that the majority of reads after writes does not share any relation, the reads are generally logically separate from the writes, often it isn't even the same rows.

While writes in AL often pertain to a single record (Modify/Insert/Delete), reads in AL often affect a multitude of records, particularly due to FlowFields, `Count`, and `IsEmpty` operations.

Without this relation between rows written to and read from, the need for heightened consistency guarantees for rows read is limited to few, well specified, scenarios. Which often anyway are ensured consistent via calling `LockTable`.

## Tri-state locking
With the limited relationship between writes and reads, it is possible to create a solution that allows for higher concurrency levels for reads, while still allowing for the heightened consistency when specific scenarios require it.

The Tri-state locking protocol is our attempt to do this, it does so by introducing a third state in-between the two previously described, leading to the following protocol:
1. All reads have the [READUNCOMMITTED](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16#readuncommitted) hint applied as long as no writes has been done to the table in the current transaction nor `LockTable` called on a record of the table type.
2. If writes have been done against the table in the current transaction, further reads will have the [READCOMMITTED](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16#readcommitted) hint applied.
3. If `LockTable` has been called on a record of the tables type in the current transaction, further reads will have the [UPDLOCK](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16#updlock) hint applied.

The tri-state protocol maintains the previous behaviour when it is explicitly requested by calling `LockTable`, ensuring the previous heightened consistency guarantees for subsequent reads.

```AL
trigger OnAction()
var
    curr1: Record Currency;
    curr2: Record Currency;
begin
    curr1.FindFirst(); // READUNCOMMITTED

    curr1.Code := 'BTC';
    curr1."ISO Code" := 'XBT';
    curr1.Symbol := '₿';
    curr1.Insert();

    curr2.FindLast(); // READCOMMITTED
end;
```
##### Listing 2: Tri-state locking shown in AL with SQL table hint annotated as a comment.

### The opt-in experience

The feature key which is called 'Enable Tri-State locking in AL' will exist from version 23 and can be toggled on and off inside the product via the Feature Management page. It affects the entire environment and is first enabled from the next session.

### Concurrent reads after writes

Using READCOMMITTED, instead of UPDLOCK, it becomes possible to have concurrent reads after a write. This is possible since READCOMMITTED takes a (S)hared lock in SQL Server, which is compatible with itself, as shown [below](#table-1-simplified-compatibility-matrix-of-locks-in-sql-server-with-n-signifying-no-conflict-between-two-requests-and-c-as-a-conflict-leading-the-latter-requested-to-have-to-wait).

UPDLOCK, however, is not self-compatible and therefore the latter reader will be blocked and must wait for the former to relinquish its held locks. In example [three](#listing-3-al-code-that-shows-how-concurrent-reads-are-possible-under-tri-state-locking-but-not-two-state-locking) the `FindLast` in the background session does not block reading the same row in the foreground session while using tri-state locking, but it does with two-state locking.

| Requested lock type | NO LOCK | (S)hared | (U)pdate | E(X)clusive |
|---|---|---|---|---|
| NO LOCK | N | N | N | N |
| (S)hared | N | N | N | C |
| (U)pdate | N | N | C | C |
| E(X)clusive | N | C | C | C |
##### Table 1: Simplified compatibility matrix of locks in SQL Server*. With N signifying no conflict between two requests and C as a conflict, leading the latter requested to have to wait.

```AL

trigger OnAction()
var
    curr: Record Currency;
    sid: Integer;
begin
    StartSession(sid, Codeunit::TakeLockInBackground);
    sleep(100);

    curr.Code := 'DKK';
    if curr.Insert() then;

    curr.FindLast(); // Will fail due to waiting for lock is limited to 30 seconds.

    Message('Only reachable with tri-state locking.');
end;

codeunit 50144 TakeLockInBackground
{
    trigger OnRun()
    var
        curr: Record Currency;
    begin
        curr.Code := 'USD';
        if curr.Insert() then;

        curr.FindLast();

        Sleep(60000); // Ensure we hold a lock longer than lock timeout.
    end;
}
```
##### Listing 3: AL code that shows how concurrent reads are possible under tri-state locking but not two-state locking.
Based on statistical data from SaaS Business Central, roughly 50 per cent of all lock timeout occurs comes when reading data, not modifying data. It is without a doubt that a significant amount of time in SQL can be attributed to waiting for locks.

### Overwriting previous reads in multi-session scenarios
With tri-state locking reads after writes does not ensure consistency of the read rows for the remainder of the transaction, the implication is that other sessions are free to alter the data in-between a read and subsequent write. This behaviour was not possible for the two-state lock protocol since it did guarantee consistency of rows read after writes.
The runtime will ensure that overwrites cannot be done on two different sessions, meaning if another session has modified a row you have read, any subsequent attempts to write to a row from your session will fail.

In a similar manner the two-state protocol allowed for overwrites of data between different records but in the same session. Listing [four](#listing-4-al-code-that-shows-how-it-is-possible-from-a-single-session-to-do-multiple-modifies-against-the-same-record-but-second-write-leads-to-overwriting-the-first-modifys-values) shows how data can be overwritten between two record instances within the same session. Tri-state locking allows for similar behaviour to stay consistent, however certain scenarios tri-state locking cannot keep track of which session modified the row and will fail if the single session modify was done with `ModifyAll`.

```AL
trigger OnAction()
var
    curr1: Record Currency;
    curr2: Record Currency;
    curr3: Record Currency;
begin
    curr1.FindLast();
    curr1.Delete();

    // Two-state: Read with UPDLOCK
    // Tri-state: Read with READCOMMITTED
    curr2.FindFirst();

    curr3.FindFirst();
    curr3.Description := 'FOO';
    curr3.Modify();

    curr2."ISO Numeric Code" := '42';
    curr2.Modify();

    curr1.FindFirst();
    Message(curr1.Description); // United Arab Emirates dirham
end;
```
##### Listing 4: AL code that shows how it is possible from a single session to do multiple modifies against the same record, but second write leads to overwriting the first modify’s values.

This can be resolved by explicitly controlling the read isolation of the read with `rec.ReadIsolation := IsolationLevel:: RepeatableRead`, using `rec.LockTable()`, or not using tri-state locking, options presented in preferential order.


\
\
To ease the reader a simplified version of the lock compatibility matrix was shown, the complete SQL server compatibility matrix can be found [here](https://learn.microsoft.com/en-us/sql/relational-databases/media/sql-server-transaction-locking-and-row-versioning-guide/sql-server-lock-conflict-compatibility.png?view=sql-server-ver16)