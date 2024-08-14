---
title: "BCTechDays 2024 & RCSI size question"
author: "Mads Gram"
date: 2024-08-14
draft: false
---

### BCTechDays
Another amazing TechDays conference has passed, it serves as a good early reminder that technical depth is lusted for. [Luc](https://x.com/luc_vandyck) opened the ball with the cheeky remark “thanks to Microsoft for sending their engineering team, not their PMs”. With technical sessions galore, I think the assignment was executed with gusto by all the speakers.

For my own session, for which a recording can be found [here](https://www.youtube.com/watch?v=Z1avFd77I18), I had the pleasure of presenting with a legend of the product Torben W. Meyhoff, who I sat next to in my early days of my career at Microsoft.
I am overall satisfied with the presentation and consider it a success.
Again this year it was exceptionally well visited, almost filling the large room eight.
![Audience](/images/posts/BCTechDays2024/BCTechDays_Audience.png)

Surprisingly, there was not a lot of overlap with last year's participants, suggesting that there isn't just a small core of developers who are interested in the technical aspects of locking.
Which I think is a testament to the understanding the AL requires a good understanding of the interplay with the database, or maybe customers are too often annoyed with locking issues.

### Questions
The first question confirmed a worry of mine, [LockTable](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-locktable-method) is still believed to "lock the table".
If the reader is under such belief, I suggest they read my [blog post on the matter](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-locktable-method) followed up by their own experimenting with it.

One of the questions we unfortunately didn't have the time to fully answer, was on the matter of “Read Committed Snapshot Isolation (RCSI)”, specifically the size addition of enabling it.

The main goals of the presentation was to explain how “Tri-State locking” changes how the AL runtime implements locking and to ensure developers are aware of it, before we switch it on for all environments in v25, and it becomes the default in v26.
Both to avoid listeners from conflating Tri-State locking and RCSI and due to time constraints, we decide not to explore RCSI in the same presentation.

### Size increase
The very knowledgeable questioner states that a 14 byte per clustered index.

A table in AL is represented as a logical row exposed on a record instance, will all fields (columns) combined and equally accessible.
However, this does not make any guarantees about its underlying data model.

The current data model stores table fields in one SQL table and table extension fields in another SQL table and primary key to allowing direct linkage.
This of course means tables without table extensions have an additional 14 byte per-row overhead and tables with table extensions have a 28 byte per-row overhead.

While it wasn't an impacting factor in moving away from the "old" data model, of a SQL table per-table extension, it again confirms that change in data model was the right choice.

A significant point to remember is that the overhead is [only applied for data was created or altered after snapshot isolation was enabled on the database](https://techcommunity.microsoft.com/t5/sql-server-blog/overhead-of-row-versioning/ba-p/383379).
This will likely leave large amounts of historical data not affected by the change.

### Row-level overhead
Adding extra overhead is always a hard decision, which requires careful consideration between what is gained and the imposed overhead.
This time the decision was taken by Azure SQL with them enabling snapshot isolation on all Azure SQL databases by-default, we could of course have disabled it on our databases, however the synergy with Tri-State locking made that proposition untenable.
A similar decision had to be made when we added SystemId to all AL tables, it is now used throughout much of the base application, enabling many AL scenarios.

It is further useful to put the size of the overhead into perspective, what is the size of an average row in some of the most populated tables in Business Central?
Putting the [complexities of row size in SQL](https://learn.microsoft.com/en-us/sql/relational-databases/pages-and-extents-architecture-guide?view=sql-server-ver16#large-row-support) aside, a naive view of the maximum row size in demo data is 296 for the Item table, 458 for Sales Line, 506 for Purchase Header, and 181 for G\L Entry.
These values are without the 14 bytes overhead, and demo data likely isn't quite as "full" as real world data.

For the readers whom are still concerned with the increase in database size after enabling snapshot isolation (which has already happened for SaaS), I would recommend that they have a quick listen to what [Rayner](https://x.com/RaynerVaz) vaguely hints at the ["Designing for scale in Business Central online" session](https://youtu.be/r48KmhfJ_FQ?t=4050).

### RCSI Tease
RCSI pairs perfectly with Tri-State locking due to its inherent increase in the usage of `READCOMMITTED` which before was essentially non-existent in AL, only used for reading BLOBs or if explicitly requested via [ReadIsolation](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-readisolation-method), so it is only naturally that we must explore the subject in full in a future blog post.


Until then, I would like to direct the curious reader to two excellent posts on the matter from BC community members.
[Stefano Demiliani's post](https://demiliani.com/2023/11/23/dynamics-365-business-central-sql-server-and-read-committed-snapshot-isolation-impact/) is a great starting point which can be followed by [Alexander Drogin's post](https://www.keytogoodcode.com/post/readcommitted-isolation-in-azure-sql).
