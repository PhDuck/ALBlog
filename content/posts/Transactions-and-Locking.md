---
title: "Transactions and Locking"
date: 2022-10-10
draft: false
---

# Introduction

AL in fundamentally a single-threaded language and as a consequence there is no support for shared data in-memory, leaving the need for traditional synchronization primitives unnecessary.
Instead, the synchronization of shared data is delegated to the database.

The capabilities supported by databases is, to some degree, reflected in AL. Operations are [transactional](https://en.wikipedia.org/wiki/Database_transaction) in AL using commits and rollback from the database.
Concurrency control, in form of enforcing exclusive or shared access, is controlled by the database, leaving the orchestration to AL _ergo_ the runtime.

These two areas are intertwined and neither can be explored on its own, so we will cover both in this post. Furthermore, we will explore implicit and explicit interactions with transactions and locking in Business Central. Since this is delegated to the database, we will also touch upon some basic concurrency concepts in SQL Server.

My hope is that the reader will find that locking and transactions have been demystified in Business Central and thereby be more equipped to lessen how often the runtime has to throw the infuriating error:
"We could not update the data right now. Please try again later.[...]"

# Implicit Behavior

While the platform provides methods for interacting with transactions and locking, in this section we will imagine an easier world where they aren't in play and the behavior is just as the platform decides.

## Transactions

When investigating transactions in Business Central two levels of transactions exists which feature careful interplay.
The runtime itself orchestrate transactions in the database by keeping track of transactions and their state and acting upon it such.

```AL
trigger OnAction()
var
    curr: Record Currency;
begin
    // Before executing the first line of AL code a runtime only transaction has been started.

    curr.FindFirst(); // (1)
    // Upon first access to the database a database transaction is started via executing the database query: BEGIN TRANSACTION;
    // The runtime notes down the transaction in DB has begun, since it now has to either COMMIT or ROLLBACK at the end of the operation.
    // FindFirst may execute without a database access becasue of data caching or using non-database backed tables (f.e. temporary), then the transaction would not start.

    curr.Code := 'BTC';
    curr.Description := 'Magic internet money';
    curr."ISO Code" := 'XBT';
    curr.Symbol := '₿';

    curr.Insert();
    // Upon writing to the database the transaction is altered to a write transaction.

    // (2)
    // Transaction is ended by calling COMMIT TRANSACTION in database.
end;
```
Listing 1: Arbitrary AL code annotated with transaction changes, for both runtime and database, annotated via code comments.

Listing 1 shows the transaction's state changes both from the AL code and the implicit state changes by the runtime.
The interaction with the database in the call to `FindFirst` starts a transaction implicitly which will last until it is either rolled back or committed.

I presume most readers already grasp the concept of database transactions, but one element is so crucial that is bears repeating: the transactions in the database supports [atomicity](https://en.wikipedia.org/wiki/ACID#Atomicity).
Atomicity being that all the changes the transaction behave as a single unit and cannot be broken down into separate elements, either all the changes in the transactions are committed or all rolled back.

An interesting implication comes naturally from this, locks taken in an operation will persist to the end of the transaction.
The "length" of a transaction varies depending on the operation, f.e. an action is simple, starting with [before-action event](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/triggers-auto/events/page/devenv-onbeforeactionevent-page-trigger), the code contained in the action, and [after-action event](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/triggers-auto/events/page/devenv-onafteractionevent-page-trigger), the operation can of course be ended early with either rolling back or committing explicitly.

## Locking

In Business Central the locks applied when fetching data from the database is implicit based on the state of the transaction.
However, the runtime does not use SQL Server's implicit locking, rather it explicitly controls locking via table hints. 

The general rule when the runtime reads data from the database is as follows:
1. All reads are done with [READUNCOMMITTED](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16) as long as no writes has been done to the table in the current transaction.
2. If writes has been done to the table (or `LockTable` called) in the current transaction, further reads will be done with [UPDLOCK](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16) against the table.

Listing 2 shows this rule in effect, where the first `FindSet` against the 'Currency' table triggers a read with [READUNCOMMITTED](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16) rule 1.
After the `Insert` call, subsequent reads towards the 'Currency' table is read with [UPDLOCK](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16) in the remainder of the transaction rule 2.

However, the read on the 'Customer' table triggers a read with [READUNCOMMITTED](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16) since that table is yet to be written and therefore is governed by rule 1.

```AL
trigger OnAction()
var
    curr: Record Currency;
    cust: Record Customer;
begin
    if (curr.FindSet()) then
        repeat
        until (curr.Next() = 0);
    // SELECT "timestamp", "Code", ..., "$systemModifiedBy" FROM Currency WITH(READUNCOMMITTED)
    // Which means no locks where taken.

    InsertBestCrypto();
    // Transaction transitions to write state
    // Table states:
    //   Currency: write

    // Afterwards a KEY lock will be held on the indices of the table.

    if (curr.FindSet()) then
        repeat
        until (curr.Next() = 0);
    // SELECT "timestamp", "Code", ..., "$systemModifiedBy" FROM Currency WITH(UPDLOCK)

    // Afterwards a OBJECT level lock will be held on the curreny table.

    if (cust.FindSet()) then
        repeat
        until (cust.Next() = 0);
    // SELECT "timestamp", "Code", ..., "$systemModifiedBy" FROM Customer WITH(READUNCOMMITED)

    // Notice no locks taken against the Customer table, since its table state is read in the transaction.
    // Table states:
    //   Currency: write
    //   Customer: read
end;

local procedure InsertBestCrypto()
var
    curr : Record Currency;
begin
    curr.Code := 'BTC';
    curr."ISO Code" := 'XBT';
    curr.Description := 'Magic internet money';
    curr.Symbol := '₿';
    curr.Insert();
end;
```
Listing 2: Arbitrary AL code annotated with SQL queries and notes on the runtime's transaction state.

# AL Language Support

At the core of AL is the tight integration with the database through the record API (queries, reports...), and declarative table definitions.
What the AL language provides can be likened to that which an [ORM](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping) provides.
In AL the interactions with transactions and locking is mostly implicitly handled by the runtime, with a few exceptions.

Let us then explore a selection of what AL provides for interacting with transactions and locking explicitly:

## Transactions

The AL API allows us to diverge from the implicit behavior through a set of methods, documentation exists and can be consulted to provide a basic understanding, while I will rather focus on how it relates to transactions.

### Error

Likely the only explicit transaction modifier that has seen large scale usage in AL development.

Logically it is used to end an operation indicating an error has occurred that does allow for further execution.
The additional affect, which is intrinsic to AL, is that the previous work in the current transaction is reverted.
Implementation wise this is archived via calling [ROLLBACK TRANSACTION](https://docs.microsoft.com/en-us/sql/t-sql/language-elements/rollback-transaction-transact-sql?view=sql-server-ver16) in the database.

```AL
    procedure DoYouFeelLuckyPunk()
    var
        cust: Record Customer;
    begin
        DeleteAllData();

        if (System.Random(2) = 1) then
            Error('You were lucky! All your data was not deleted! I dare you to try again ;)');

        // It makes no semantic difference if the DeleteAllData() call was moved here.
        // However performance wise it is smarter to first delete when we know we must.
    end;

    local procedure DeleteAllData()
    ...
```
Listing 3: Code listing where the [Error](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/dialog/dialog-error-string-joker-method) method is used to, sometimes, revert the changes made previously in the transaction.

The curious reader might ponder on how to implement the `DeleteAllData` method in AL, I encourage the reader to give it a shot and then contrast their solution with my solution, which can be found [here](https://gist.github.com/PhDuck/6e37cf102309fab7056ce585dc205ec4).
The solution provided shows the efficiency of AL, where many languages would require far more to delete all data in the backing data store.

### Commit

As we explored in the implicit code examples all successful operations (which have begun a transaction) will end up executing an implicit commit which applies all the changes made during the transaction.
But what if one wanted to commit halfway through the operation?
Can we split the operation into multiple transactions? Commit to the rescue!

Imagine the task of processing millions of records, depending on the processing it might take too long to do in one transaction (or maybe just block other users for too long). Instead, one could break it down into a small batch of records to be processed, following a commit and exiting the code. When time allows, the code could be called again, and it continues from where it left off.

Listing 4 presents an example of this pattern, where an amount of tasks is processed within an allotted timeframe, with `BatchSized` checkpoints which serves as checkpoints.

```AL
local procedure ProcessTasksInBatchWithMaximumDuration(BatchSize: Integer; MaxDuration: Duration): Integer
var
    task: Record Task;
    start: DateTime;
    i: Integer;
    j: Integer;
begin
    if (BatchSize <= 0) then
        Error('BatchSize must be at least 1.');

    start := CurrentDateTime();

    task.SetRange(task.Processed, false);
    if (task.FindSet()) then begin
        repeat
            if (i = BatchSize) then begin
                Commit();
                i := 0;

                if ((CurrentDateTime() - start) >= MaxDuration) then begin
                    exit(j);
                end;
            end;

            task.Process();
            task.Processed := true;
            task.Modify();

            i += 1;
            j += 1;

        until task.Next() = 0;
    end;

    if (i < BatchSize) then
        Commit(); // Exited before reaching a BatchSize, make sure to commit.

    exit(j);
end;
```
Listing 4: Generic code for subdividing code into batches with a maximum allotted time to spend executing.

I however encourage the reader to avoid explicit commits as much as possible, unless the entire code path has been built to support them, they can cause problems for other code which expects to be rolled back on errors.
They are considered so problematic that it was necessary to introduce a method to disable them:
[CommitBehavior](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/attributes/devenv-commitbehavior-attribute), which allows one to ignore explicit commits in a scope.
One of the few reasonable usage cases is to allow for calling into [Codeunit.Run](https://docs.microsoft.com/en-us/dynamics-nav/codeunit.run-function--codeunit-) and [isolated events](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-events-isolated), which does not allow an active write transaction to be started.


## Locking

In the implicit case the locks will be taken upon writing to the table and subsequent reads will also take locks.

### LockTable

A remnant of BC's past leaves us with this API, where the behavior isn't consistent with its naming.
[LockTable](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-locktable-method) does not lock the entire table nor does calling the method immediately take any locks in the database.
The actual behavior is the same as writing to table, it will upgrade the table state of the record's table to write in the transaction, thereby, subsequent reads will be performed with [UPDLOCK](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16).

```AL
codeunit 12 "Gen. Jnl.-Post Line"
{
...
    local procedure InitNextEntryNo()
    var
        GLEntry: Record "G/L Entry";
        LastEntryNo: Integer;
        LastTransactionNo: Integer;
    begin
        GLEntry.LockTable();
        GLEntry.GetLastEntry(LastEntryNo, LastTransactionNo);
        NextEntryNo := LastEntryNo + 1;
        NextTransactionNo := LastTransactionNo + 1;
    end;
...
```
Listing 5: Code from the codeunit "Gen. Jnl.-Post Line" showing how [LockTable](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-locktable-method) can be used to ensure exclusive access to the next entry number.

To get the next entry number based on LastEntryNo + 1 does not work without locking, since two sessions could fetch the same last number and both add one to it, both arriving at the same number.
If that number is used as a primary key (or guaranteed unique via a unique index) SQL server will validate the uniqueness at insert/modify time and return an error, else the number would be inserted twice which depending on the usage could be catastrophic.

In SQL Server [READUNCOMMITTED](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16) and [UPDLOCK](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16) does [not conflict](https://docs.microsoft.com/en-us/sql/relational-databases/media/lockconflicttable.png?view=sql-server-ver16) meaning the read proceeds without having to wait for a lock, so multiple users can still read the same row.
Therefore, it is necessary to ensure both reads also are done with [UPDLOCK](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16) which [LockTable](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-locktable-method) allows for, without first performing a write to the table.

Instead of using locking here, one should rather consider using number sequences, which allows for non-blocking drawing of next number. See the [docs](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-number-sequences) for more info.

# Conclusion

Transactions are fundamental to the development model of Business Central, and they are mostly handled implicitly by the runtime. They have natural starting and ending points, without interactions from the developer.
A feature that doesn't really need an API, which is why I think they are rather elegant.
Consider, how one would write the AL that underpins Business Central with transactions to allow for rollbacks or single unit commits.
