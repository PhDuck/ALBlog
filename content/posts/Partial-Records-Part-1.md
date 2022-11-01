---
title: "Partial Records: Part 1"
author: Mads Gram
date: 2022-01-10
draft: false
---

# Introduction
Previously on this blog the [Record](https://bcinternals.com/posts/on-records/) type was presented and explored. In that post it was pointed out that the Record type and its accompanying API is large enough to warrant [Directions](https://directions4partners.com/) sessions (and blog posts) for merely subsets of it. That was written with great confidence since I did so in our technical presentation on Partial Records at Directions.

Having led the development and design of Partial Records I will allow myself some praise:
![Marije](/images/posts/Partial-Records-Part-1/MarijeBrummelComment.png)

The post will cover the history of Partial Records and show how it came to be. It will discuss and rationalize its very existence through experiments. Finally, the implementation will be shown in a simplified form, to provide a mental model for using Partial Records.

# History
With the impressive age of [Business Central (counting its previous incarnations)](https://web.archive.org/web/20140513013103/http://dynamicsuser.net/wikis/navdev/the-history-of-dynamics-nav-navision.aspx) much of the design and implementation has been done previous to my employment at Microsoft (some even before my birth). So, many of the posts on here will cover code written by others, years before I was writing code.

Partial Records could have been another of these since there was another attempt to produce a feature similar to Partial Records some 10 years ago. While it evidently wasn’t successful, the idea was never fully scrapped and came back into play with extra importance after table extensions showed its dark side.

Much of the AL community have likely read the famed [post]( https://www.linkedin.com/pulse/avoiding-table-extension-hell-james-crowter/) by James Crowter, and one could be excused to suspect his post inspired the development of Partial Records, the timing even fits. However, the feature was prioritized a few months before his post and after a frantic Easter holiday, a prototype emerged to prove the current design and that we didn’t need any more meetings to discuss the design.

The prototype proved it was not just possible, but a strategy of specifying the fields, which in turn defines which table extensions need to be loaded, was the way to go. That diverged from the original task of just skipping joining of table extensions. It left the obvious question of how to determine the set of fields to load. In early versions it was defined on the AL table definition, which clearly left a lot to be desired since different AL code paths need different fields. A few internal developers suggested being inspired by the SetAutoCalcFields API. That merely left us with the naming of the SetLoadFields API, which I can be blamed/praised for.

# What?
For the ones who don’t know what Partial Records is, a quick overview will be presented, but this post’s main focus isn’t to present it as a new feature, for that, please go read the [official documentation](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-partial-records).

Without Partial Records any data fetching operation (Get and Find*) on a record in BC will load out all normal fields. So, a Get on the Item table will load out ~140 fields from SQL. Furthermore, if any table extension exists against the Item table, these will be joined in and their normal fields also loaded out.

Partial Records allows the proficient AL developer to hint which fields should be loaded via its record API: [SetLoadFields](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-setloadfields-method). By specifying the fields to load, implicitly the table extensions to load are also specified.

# Why?
Most developers have heard the shortened (bastardized) version of the famous Donald Knuth quote:

> “We should forget about small efficiencies, say about 97% of the time: *premature optimization is the root of all evil.* Yet we should not pass up our opportunities in that critical 3%. A good programmer will not be lulled into complacency by such reasoning, he will be wise to look carefully at the critical code; but only after that code has been identified.” - Structured Programming with go to Statements.[emphasis added]

Listening to the founders of computer science is most likely wise. Therefore, it should be established that joining of tables and overloading fields is inefficient by empirical measures. To do so two experiments are presented below.

## Number of fields
In the first experiment six tables have been created, all with varying amounts of fields: 1, 9, 19, 39, 79, and 139. The fields are of various BC data types, all tables are filled with 1000 rows and values in all fields. Each table is looped over all rows loading out either a single field or all fields.

| Fields loaded | 1   | 9     | 19    | 39  | 79    | 139   |
|---------------|-----|-------|-------|-----|-------|-------|
| All fields    | ~1x | ~1.5x | ~1.75 | ~9x | ~154x | ~224x |

Table 1: Shows the ratio of which loading all fields is slower compared to single field load, larger is worse.

The experiment’s results clearly show proportionality between the number of fields loaded and execution time. Establishing the growth rate accurately would most likely require more data points, but a sensible estimate seems to be linear growth.

## Number of table extensions

Eight tables have been created, all with varying amounts of table extensions, 1 to 8. These table extensions each add five fields to the table. The fields are of various BC data types, all tables are filled with 1000 rows and values in all fields. In each table all rows are looped overloading out either a single field or all fields (thereby also table extensions).

| Table extensions loaded | 0   | 1     | 2      | 3      | 4      | 5      | 6      | 7     | 8      |
|-------------------------|-----|-------|--------|--------|--------|--------|--------|-------|--------|
| All fields              | ~1x | ~5.5x | ~20.5x | ~44.5x | ~51.2x | ~57.0x | ~65.2x | ~69.3 | ~76.7x |

Table 2: Shows the ratio of which loading all fields is slower compared to single field load, larger is worse.

The experiment’s results clearly show an increase between each additional table extension. The difference is larger than just the cost of reading more fields, so we can establish that the joining must have a cost.

# How?
The experiments unsurprisingly show that loading more fields and joining in more tables cause a substantial increase in execution time. The goal of Partial Records is exactly to combat this increase. A later post in this series on Partial Records will feature an in-depth exploration of the optimizations. While the remainder of this post will focus on the implementation of Partial Records and JIT loads.
## Implementation

As previously presented in [Record](https://bcinternals.com/posts/on-records/), records maintain state which impacts subsequent accesses to a data provider. With the introduction of Partial Records another member of state was added onto record type, internally the structure is called “FieldLoadInfo”. In theory FieldLoadInfo need only maintain a set of fields, which represents the fields needed to be loaded on subsequent load. In reality, it also maintains the opposite to quickly answer if a field is already selected for load.

```AL
trigger OnAction()
var
    Item: Record Item; // 1: Create the record handle.
begin
    Item.SetLoadFields(Item."No."); // 2: Initialize the record and alter the state.
    Item.Find('-'); // 3: Access the data provider and fetch the record.
end;
```
Listing 1: Trivial Partial Records usage example.

Based on the code snippet’s steps 1 to 3 the record’s state is changed and used as follows:
1.	The record is initialized, regarding state that implies the default values will be imposed, for which fields are to be loaded - all normal fields are selecting for loading, except Obsolete=Removed fields.

2.	The record’s FieldLoadInfo is replaced by a new FieldLoadInfo. The new FieldLoadInfo does not merely contain the field: “No.” it also contains [other fields needed by the runtime]( https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-setloadfields-method#remarks).

3.	The record’s state is collected, and that state is used to:
    1.	First query the in-memory cache, which hopefully can return a response, saving one from accessing SQL. Results are here allowed to have more or the same number of fields loaded than what requested. Therefore, it is possible that more fields that requested will be loaded.

    2.	Suppose querying the cache did not yield any viable results. Then the SQL database must be queried. Since SQL is a query-based language, the record’s state is used to create the query. Here FieldLoadInfo determines which fields are in the SELECT clause and which table extensions to include in the FROM block.


## JIT loads:
In the original design, JIT loads wasn’t required or planned, rather accessing an unloaded field was an error. At some point it became apparent that it would be far too hard to always select the right fields. Furthermore, if the platform wanted to automatically apply Partial Records, it was never going to work.

The mechanics of JIT loads aren’t tremendously complex, as under the hood they are just a GET on the record, followed by some validation logic to ensure that one cannot have diverging values. For example, it is not allowed to have read field3 and then have field3 change in-between the initial load and secondary load. It is however allowed that an unread field has changed between initial and subsequent load.

JIT loads should be avoided when possible! They require another access to the database and will likely undo all the performance gains achieved by using Partial Records in the first place. Furthermore, because the record may have been deleted or diverged enough it may fail, causing a runtime error. The server will attempt to automatically correct the FieldLoadInfo adding the fields that caused a JIT load to happen, so upon next iteration on the same record a JIT load would not be needed for the same field.

### Implicit platform JIT loads
Certain operations in the runtime will force a JIT load, either to protect or ensure integrity. In general, it will be found that Partial Records should mainly be applied in read-only scenarios. Scenarios that cause an implicit JIT load are as follows:

1.	Copying to a temporary record:
Necessary since temporary does not support JIT loads.

2. `Rec.Delete()`:
With the record being deleted in the database JIT loads would be guaranteed to fail. 

3.	`Rec.Insert()`:
Based on feedback from AL developers the expected semantics was that a subsequent Insert should have all the values, therefore a JIT load was added.

4.	`Rec.Rename()`:
Renames would change the primary key in the database, making GET requests fail. Theoretically it would be possible to use the row SystemId instead, that might be attempted in the future.

# End
Partial Records are for the platform teams slowly wrapping up, most of the work has been done and leaving just a few extras goodies in the pipeline, it is stable and being used 10s of millions of times a day. At the same time AL developers have begun to embrace it and the number of usages in Base App is going up.

The name “Partial Records” comes from and off-hand remark from [Esben](https://twitter.com/esbennk). A reggae/dub record label is also using the same name, but it was decided that there wasn’t any overlap, so the name could be used. Based on Google and Bing’s search results the record label is still more popular 😊.
