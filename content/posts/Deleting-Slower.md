---
title: "Deleting Slower"
author: "Mads Gram"
date: 2024-01-27
draft: false
---

# Introduction
Many optimizations are achieved through an expansion in lines of code, producing efficient but verbose code. These are often done by having intimate knowledge of the internals of technology, see for example buffering of reads and writes in disks in operating systems, loop unrolling of repeated access, etc.

For runtime developers, the holy grail of optimizations is when abstractions don’t have to leak, and the best performance is achieved with the simplest possible solution, no internal knowledge is necessary for the AL developer, nor unreasonable limits need be applied.

This blog cost will cover mass deletion of data in Business Central and the way your choices can lead you away from the optimal performance.

# Warning
It seems pertinent to remind the reader that all measurements here are captured with SQL Server and Business Central co-located on my local machine. Since SaaS Business Central does not run on my local machine, these numbers do not necessarily reflect the performance one can expect in the cloud.

# Baseline deletion measurements

In this entire blog post we will use the SQL backed table Currency, due to its inherent simplicity. The time for Business Central to delete a million rows with the code in [listing 1](#listing-1-al-code-for-deleting-all-1-000-000-currency-rows-while-the-measuring-wall-time-the-time-excluding-the-cost-of-committing-the-transaction).

```AL
procedure DeleteAll()
var
    curr: Record Currency;
    startTime: DateTime;
begin
    startTime := CurrentDateTime();

    curr.DeleteAll(false);

    Message('Delete 1 000 000 rows took: %1', Format(CurrentDateTime() - startTime));
end;
```
#### Listing 1: AL code for deleting all (1 000 000) Currency rows while the measuring wall time the time, excluding the cost of committing the transaction.

```Powershell
Invoke-NAVCodeunit -CodeunitId 50115  -CompanyName 'CRONUS International Ltd.' -ServerInstance 'PlatformCore' -MethodName DeleteAll
WARNING: Delete 1 000 000 rows took: 4 seconds 681 milliseconds
```
#### Listing 2: Execution output of running the deletion procedure.
As can be seen in [listing 2](#listing-2-execution-output-of-running-the-deletion-procedure), the baseline execution time deleting 1 million rows is ~4.6 seconds. If that is fast or slow? I will let the reader decide that themselves, however the remainder of the blog post will show how we can stray from this number.

# Triggers

The careful reader has likely noticed the 'false' parameter passed to DeleteAll, which is of course intentional. [Listing 3](#listing-3-execution-output-of-running-the-deletion-procedure-with-empty-triggers) shows an 24 x slowdown, by just changing the value of that parameter to 'true'.

```Powershell
Invoke-NAVCodeunit -CodeunitId 50115  -CompanyName 'CRONUS International Ltd.' -ServerInstance 'PlatformCore' -MethodName DeleteAll_EmptyTriggers
WARNING: Delete all with empty triggers took: 1 minute 38 seconds 894 milliseconds
```
#### Listing 3: Execution output of running the deletion procedure with empty triggers.

The explanation for such a slowdown is that a parameter value of 'true' necessitates the execution of AL triggers. AL triggers are currently only implemented in a row-by-row approach, so every row must be fetched to the server, trigger executed, that row deleted, rinse and repeated.

Pseudocode of how the platform implements this behaviour can be seen in [listing 4](#listing-4-pseudocode-of-deleteall-when-triggers-need-to-be-executed-for-deleteall).

Without pondering about the possibility of SQL server optimizing a bulk delete vs many singular deletes, the inefficiencies are still clear. Just the latency of doing 2 calls per row become significant if the number of rows to delete is large.

For a million rows, at 0.2ms latency, latency allow will be 0.2ms * 2 * 1 000 000 = 6.6 minutes.
In our case with co-located SQL, the latency is negligible which is why we can avoid the 6.6 minutes, but in a SaaS system, the latency immediately becomes a problem*.

Of course, AL triggers can execute arbitrary AL code, so triggers can theoretically slow down the operation arbitrarily. In these measurements an empty AL trigger/subscriber is used.

```AL
rec.LockTable();

if rec.Find('-') then begin
    repeat
        rec.Delete(true);
    until rec.Next() = 0;
end;
```
#### Listing 4: Pseudocode of DeleteAll when triggers need to be executed for DeleteAll.

It is not just defined triggers combined with a 'true' valued parameter value that can cause the slower path to be taken. If any of the following are 'true', the row-by-row path must be taken:
- RunTrigger = 'true' AND OnDelete trigger or table extensions triggers.
- Global deletion triggers are defined for that table via [GetGlobalTableTriggerMask](https://learn.microsoft.com/en-us/dynamics365/business-central/application/system/codeunit/system.environment.global-triggers#getglobaltabletriggermask) or [GetDatabaseTableTriggerSetup](https://learn.microsoft.com/en-us/dynamics365/business-central/application/system/codeunit/system.environment.global-triggers#getdatabasetabletriggersetup), leading to [OnDatabaseDelete](https://learn.microsoft.com/en-us/dynamics365/business-central/application/system/codeunit/system.environment.global-triggers#ondatabasedelete) or [OnGlobalDelete](https://learn.microsoft.com/en-us/dynamics365/business-central/application/system/codeunit/system.environment.global-triggers#onglobaldelete) needing to be invoked.
- Any field defined as Media or MediaSet either on the table or in a table extension to the table.
- Security filtering is applied for deleting.
- Either OnBeforeDelete or OnAfterDelete event subscribers exist for the executing environment.

*:The SQL client could send more than one row per request, sometimes removing the latency of the fetch, but even with all the fetches removed, that is still 3.3 minutes. 

# Indices

It is well understood that the cost of insertions in SQL Server grows with the amount of indices defined for a given table. Unfortunately, that is also the case for deletions, [table 1](#table-1-the-time-it-takes-to-delete-1-000-000-rows-without-triggers-at-n-indices-for-the-given-table) documents the increase.

|     3 (indices)   	|     4      	|     6     	|     8     	|     10    	|     12    	|     14      	|
|-------------------	|------------	|-----------	|-----------	|-----------	|-----------	|-------------	|
|     4.7 (seconds) 	|     7.6    	|     12    	|     16    	|     21    	|     30    	|     35.6    	|

#### Table 1: The time it takes to delete 1 000 000 rows without triggers at n indices for the given table.

It should be noted that indices might be beneficial for deletions when they can support the more efficient determination of what rows need to be deleted.
When not all rows need to be deleted it may be more efficient to have an index that supports that operation.

## Disabling indices

In v24 of Business Central we are previewing an ability for developers to delete faster by adding more code, quite an exception compared to the rest of this blog post. The functionality is scoped OnPrem, so only first-party code will be able to use it.
I introduced the new platform API AlterKey and in this [pull request](https://github.com/microsoft/BCApps/pull/506) [Ihor H.](https://github.com/IhorHandziuk) revealed it to the world and presented the first use-case, solving a recurring issue for many users.

AlterKey allows for disabling (and re-enabling) of indices for the remainder of the current transaction. At either the end of the transaction or explicit re-enabling, the index will be [rebuilt](https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-index-transact-sql?view=sql-server-ver16#rebuilding-indexes), therefore the feature does not allow the permanent disabling (or enabling) of indices, that remains only possible through AL development.

[Listing 6](#listing-6-example-code-for-that-deletes-a-small-subset-of-rows-with-indices-disabled-and-explicitly-re-enabled) shows the measurements of deleting 1 million rows with or without indices, to no surprise the deletion with fewer indices enabled is much faster, as we already explored in the previous section.

With AlterKey all non-unique non-clustered indices can be disabled. In the case of Currency table, the amount keys disabled are limited, so the gain is only about 2x. For tables with many indices gains should be inferable based on the data in [table 1](#table-1-the-time-it-takes-to-delete-1-000-000-rows-without-triggers-at-n-indices-for-the-given-table).

```Powershell
WARNING: Delete 1 000 000 rows took: 4 seconds 681 milliseconds
WARNING: Delete 1 000 000 rows with only primary key index enabled took: 1 second 974 milliseconds
```
#### Listing 5: The time it took to delete 1 million rows with two indices enabled and disabled.

If AlterKey makes it faster to delete, why does the platform not just disable all keys before calling DeleteAll and then re-enable them afterwards?

While we might do it for deletes without filters, the issue arises when the filters applied to DeleteAll leaves many rows around in the table. Re-enabling (rebuilding) the index is a compute intensive operation that therefore is also timely.
[Listing 6](#listing-6-example-code-for-that-deletes-a-small-subset-of-rows-with-indices-disabled-and-explicitly-re-enabled) shows example code where a filter is applied to DeleteAll that only triggers the deletion of 1000 out of 1 million rows.

[Listing 7](#listing-7-measurements-of-deleting-1000-rows-with-disable-and-re-enable-where-needed) shows the measurements of the two deletes, where the deletion with indices performs better, due to the extra cost of rebuilding the indices.

The cost of rebuilding indices grows with the amount of rows in the table and the amount of indices that must be rebuilt. A trade-off of when it pays off to disable indices can only be established when the number of remaining rows is known.
```AL
procedure DeleteAll_WithoutTriggers_WithoutIndex_Half()
var
    curr: Record Currency;
    startTime: DateTime;
begin
    startTime := CurrentDateTime();
    DisableKeys(Database::Currency);

    curr.SetRange(curr.Symbol, '1');
    curr.DeleteAll(false);

    // The re-enable will happen automatically, but to include it in our measurement
    // we must do it before we take the end measurement.
    EnableKeys(Database::Currency);

    Message('Delete all without triggers took: %1', Format(CurrentDateTime() - startTime));
end;
```
#### Listing 6: Example code for that deletes a small subset of rows with indices disabled and explicitly re-enabled.

```Powershell
WARNING: Delete 1000 rows with indices took: 126 milliseconds
WARNING: Delete 1000 rows without indices took: 740 milliseconds
```
#### Listing 7: Measurements of deleting 1000 rows, with disable and re-enable where needed.

# Table extension:

In my previous list of clauses that could lead to a row-by-row based deletion, I omitted two remaining conditions (to not spoil the surprise), if filters are set on flowfilters OR the table is extended with table extension and the deletion has filters.

If no filters are set, but a table extension exists, [listing 5](#listing-5-the-time-it-took-to-delete-1-million-rows-with-two-indices-enabled-and-disabled) shows that the performance is near optimal. The reason for nearly optimal, is that only two SQL statements are executed, one for main table and one for the table extension table. The amount of SQL statements would increase linearly with the amount of table extensions.

```Powershell
Invoke-NAVCodeunit -CodeunitId 50115  -CompanyName 'CRONUS International Ltd.' -ServerInstance 'PlatformCore' -MethodName DeleteAll_TableExtension_Fast
WARNING: Delete all without triggers took: 8 seconds 996 milliseconds
```
#### Listing 8: DeleteAll without filters but with table extension on the Currency table.

That isn’t the case when any filters are applied, then it unfortunately gets far worse. [Listing 6](#listing-6-example-code-for-that-deletes-a-small-subset-of-rows-with-indices-disabled-and-explicitly-re-enabled) shows that case. At 33x slower than baseline the goal of deleting slower has certainly been accomplished.

The explanation follows along the lines of triggers, a row-by-row approach is applied. The filters applied are only evaluated on either main or table extension tables, but the corresponding row must also be deleted in the opposite table. The reason for it being worse than just triggers, is that each row fetched must query both tables and each delete must delete in two tables.

```Powershell
Invoke-NAVCodeunit -CodeunitId 50115  -CompanyName 'CRONUS International Ltd.' -ServerInstance 'PlatformCore' -MethodName DeleteAll_EmptyTriggers_TableExt
WARNING: Delete all without triggers took: 2 minutes 14 seconds 704 milliseconds
```
#### Listing 9: DeleteAll with filters and table extension on the Currency table.

# Conclusion
I hope that it is abundantly clear that simplicity in deletions generally leads to better performance, and additional logic in AL usually leads to slower deletions in BC.
The reality is of course that designing for the fastest deletion of data is likely never the goal, rather solving common user problems is.

Instead, consider this blog post as an exploration of cost from features in Business Central. Almost everything is a trade-off, disabling an index might speed up the deletions or insertion, but the rebuild can be tremendously expensive.
