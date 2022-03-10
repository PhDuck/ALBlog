---
title: "Partial Records: Interlude"
date: 2022-03-10
draft: true
---

# Introduction
Frequent feedback that we, as platform developers, receive is to include more code examples in our documentation. I wonder if it comes from the old academic adage of reading papers by [reading the abstract; skimming the introduction, figures, and results](https://www.science.org/content/article/how-seriously-read-scientific-paper) or maybe the fancy code listing box catches the eye first. Nevertheless, I will abuse it in this post, having many code listings containing code examples.

This post was originally meant to be a “down and dirty” look in the platform’s engine room, exploring how Partial Records makes data fetching faster. But to satisfy the AL developers who are always asking for more examples, we will first have to take a quick detour in our series 😊.

All the examples in this post will however be anti-patterns, examples of what NOT to do when using Partial Records, which should be considered as a non-exhaustive list of things not to do. While I am generally a fervent opponent of disabling Copy & Paste on websites, this is one of the situations where it would make sense since all examples in the post will either lead to poor performance or runtime errors!
If you are wondering what Partial Records is, a good starting point is [the official documentation](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-partial-records) followed by my previous [post](https://bcinternals.com/posts/partial-records-part-1) on its internals.

# Examples
Since all the examples have to be anti-patterns, we will need to explore the dark side of Partial Records: JIT loads. They are a necessary evil needed for the cases where a developer (or the platform) guessed wrong, needing more fields that originally asked for. In the common case this will lead to a performance hit, but in some cases a runtime error may occur. The following examples will clarify which and when.

## Obvious JIT load
Starting with the simplest example imaginable, the JIT load is trivial and could be avoided by just adding the “Country” field in the SetLoadFields call. This is an example of a JIT load coming from accessing an unloaded field, which is likely the most common type.
```AL
local procedure GetCustomerLocation(): Text[30]
var
    Cust: Record Customer;
begin
    Cust.SetLoadFields(Cust.City, Cust.Address);
    Cust.Find('-');
    exit(Cust.County); // Triggers JIT since County field isn't loaded.
end;
```
Listing 1: Triggering an obvious JIT load by accessing a field that isn’t selected for load.

## Platform implicit JIT load
A less common but a bit harder to understand in the platform required implicit JIT loads. These comes when the platform (server) requires that all fields are loaded. Currently the list of things that trigger this is:
1.	Calling Delete, DeleteAll, Insert, Rename, or TransferFields on the record that is partially loaded.
2.	Calling Copy onto a record that is temporary.

```AL
local procedure GetCustomersLocationAdvanced(): Text[30]
var
    Cust: Record Customer;
begin
    Cust.SetLoadFields(Cust.City, Cust.Address);
    Cust.Find('-');
    Cust.Delete(); // Triggers a platform implicit JIT load.
    exit(Cust.County); // No JIT since we already did above.
end;
```
Listing 2: Platform implicit JIT load triggered via calling the Delete method.

## ResultSet enumerator updating on JIT loading
As previously mentioned, JIT loads predominantly happen when accessing an unloaded field so one might expect the worst case is to read the same field in a loop over a large table. However, when designing Partial Records we stumbled upon this problem and decided to fix it.
Upon once hitting a JIT load that field will automatically be added to the list of fields to load when calling Next on the record. So, to produce the worst (performance) case one must be far more creative.
Here one must trigger a JIT load on one field at a time while iterating. That being said, one could also just reset the ResultSet enumerator in each iteration of the loop, but that shouldn’t be done [either]( https://bcinternals.com/posts/on-records/#resultset-enumerator).
```AL
local procedure GetCustomersLocationExpert(): Text[30]
var
    CustRecRef: RecordRef;
    i: Integer;
begin
    CustRecRef.Open(Database::Customer);
    CustRecRef.SetLoadFields(1);

    i := 2;
    if (CustRecRef.Find('-')) then
        repeat

            Message(Format(CustRecRef.Field(i).Value));
            repeat
                i := (i + 1) mod CustRecRef.FieldCount;
            until CustRecRef.FieldExist(i);
        until CustRecRef.Next() = 0;

    exit('Somewhere'); // Don’t even care about the goal anymore.
end;
```
Listing 3: Triggering a JIT load on every iteration by loading out an unloaded field per iteration.

# JIT load errors
The above examples are mistakes that leads to JIT loads causing worse performance than if Partial Records had been applied correctly. The remaining will focus on the potentially worse when JIT loads cause runtime errors to occur.

When using Partial Records and triggering a JIT load the database must be queried twice, once for the initial load and once for the JIT load. In-between the row isn’t guaranteed to stay the same, f.e. the row might be deleted or altered. Therefore, JIT loads aren’t guaranteed to succeed.

Two types of errors are provided by the platform: inconsistent read error and the more generic JIT load error.
## Inconsistent read
Inconsistent read errors mainly cover two scenarios, concurrency and changes that aren’t reflected in the record currently operating on. For the AL developer these are two distinct cases, that needs different solutions, but for the platform they are the same.
### Un-reflected changes
An un-reflected change consists of a change in the database that hasn’t been reflected in the local version of the record. In the below example procedure `rec0` has the initial view of the row in memory after calling FindFirst, after which the rows address is updated, since the update happened on another record, `rec0` still has the previous value. Upon accessing the “County” field on `rec0` a JIT load is triggered, which then compares the initial row’s values and the ones just loaded from the database.
```AL
local procedure TriggerInconsistentUnreflectedError()
var
    rec0: Record Customer;
    rec1: Record Customer;
begin
    rec0.SetLoadFields(rec0.Address);
    // Fetch initial view of row, fetching at least the Address field.
    rec0.FindFirst();

    rec1.FindFirst();
    rec1.Address := 'Cala Montjoi';
    rec1.Modify(); // Change the row in the DB.

    // Trigger a JIT load, which again loads out the row.
    // field Address is determined to have changed.
    Message(Format(rec0.County));
end;
```
Listing 4: Triggers an “Inconsistent Read” error by reading a field that was modified in-between initial and JIT load on another record.

The choice to disallow an inconsistent read between the initial and JIT load, is one of carefulness, the platform could easily ignore this inconsistency, either by updating the records value or simply ignoring it. Currently we do not allow this to be overwritten, but if enough demand exists, we could provide it as an option on the [LoadFields](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-loadfields-method) method.

The curious reader might question what happens if an unloaded field changed in-between initial and JIT load? It was decided that since that value could not have been observed on the record, it is allowed that an unread value is updated. Thereby, after a JIT load, the record can have values from a previous view of the row and new view.

Concurrency changes are essentially the same as above, just done by another session. They will happen on tables that are often changed. Here locking will solve the problem of concurrency issues but will lead to waiting for other sessions.

## JIT load error
The other type of runtime error thrown by JIT loads is the more “generic” JIT load error. While it technically covers over all sorts of transient errors, in regard to the function of Partial Records only the permanent error matter and only it will be elucidated in this section.

The permanent error that will be observed when a JIT load error occurs is that the underlying row in the database has been deleted (or renamed) in-between the initial and JIT load. This is the simplest presentation of why JIT loads cannot be guaranteed to succeed that I have found and is therefor the crutch to lean on.

To limit the occurrence of it the platform has inserted a platform implicit JIT load before calling Delete, DeleteAll, or Rename, so that after deleting the row the reader doesn’t cause a JIT load that would be guaranteed to fail.
```AL
local procedure TriggerJITLoadErrorViaDelete()
var
    rec0: Record Customer;
    rec1: Record Customer;
begin
    rec0.SetLoadFields(rec0.Address);
    rec0.FindFirst();
    rec0.Delete(); // Triggers platform implicit JIT load.
    Message(rec0.County); // Doesn't trigger JIT load because of above platform implicit JIT.

    rec0.Reset();
    rec0.SetLoadFields(rec0.Address);
    rec0.FindLast();

    rec1.FindLast();
    // Triggers platform implicit JIT load on rec1, but rec0 still has the old record view.
    rec1.Delete();

    Message(rec0.County); // Triggers JIT load which fails cause record has been deleted.
end;
```
Listing 5: Shows the behavior of platform implicit JIT loads and JIT load failures from a deleted row in the database.
# End
To all the AL developers who have asked for more examples, I hope this twist still satisfies your request.
With the majority of bad ways to use Partial Records shown here, I expect to see great applications of Partial Records in the wild.

A side note on language design, we the language developers aren’t infinitely creative (or wise), we cannot foresee all possible usages of our features. It is up to our users to be truly creative, testing the limits and finding interesting combinations. So, if you find yourself being overly creative with Partial Records, please tweet at [me](https://twitter.com/PhDuck2) 😊
