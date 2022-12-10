---
title: "Directions EMEA 2022: Questions"
date: 2022-12-10
draft: true
---

# Introduction

Recently I had the pleasure of attending [Directions EMEA 2022](https://directions4partners.com/events/directions-emea-2022/) in Hamburg, likely the largest conference in the Business Central sphere.
Having only been at Days of Knowledge previously, the ~3000 attendees shows how impressive our community is.
At Microsoft, we have internal usage numbers which show impressive usage, however the gravity of our product can sometimes be forgotten.
So, seeing the keynote fill a 3000-person room does an amazing job conveying the importance of what we work on.

# Questions

At the event I had the opportunity to present at three different sessions.
Unfortunately, time didn't allow us to take question in _plenum_, so many questions were asked on a one-on-one basis.
Some of the questions, and their accompanying answers, would likely provide value for others, so I will present them here.

## RecordRef

The first question was how Record and RecordRef are different in regards to a concrete scenario, the precise question isn't important since my answer was that Record and RecordRef, from a runtime perspective, is fundamentally the same.
The "from a runtime perspective" is crucial part of that statement, since RecordRef from a compiler or AL developers perspective are very different.

The Record API is strong and statically typed, allowing only for the definition and instantiation of Record variables prescribed by a Table object in the current extension or a declared dependency, ergo the type is checked at compile time.
The RecordRef API is however dynamically typed, leaving the type check to happen at runtime rather.

```AL
table 424242 Cake
{
    ...
}

codeunit 424243 Foo
{
    local procedure ValidCode()
    var
        c: Record Cake;
    begin
        Message(Format(c.Count));
    end;

    local procedure InvalidCode()
    var
        b: Record BlaBlaBla; // Table 'BlaBlaBla' is missing AL(AL0185)
    begin
        Message(Format(b.Count));
    end;

    local procedure ValidDynamicCode()
    var
        rref: RecordRef;
    begin
        rref.Open(123456789);
    end;
}
```
#### Listing 1: AL code with compiler errors annotated as comments.

In [listing 1](#listing-1-al-code-with-compiler-errors-annotated-as-comments) the three methods are defined, all three are grammatically and syntactically valid code.
The compiler however denotes an error for the declaration of the variable `b` in the function `InvalidCode` based on a table definition named `BlaBlaBla` is missing (unknown).
The function `ValidDynamicCode` uses the RecordRef API rather which is dynamically typed, trusting the developer to have entered a table id that will be valid at runtime. Whether that is correct, depends on which extensions are installed at runtime.

Assuming we have removed the function `InvalidCode` and run the code, the runtime will instantiate the variable `c` with a record instance of type `Cake`, this implicitly includes a type check, so practically the Record API is both statically and dynamically typed.

Running the method `ValidDynamicCode` the instantiation of the RecordRef variable `rref` does not perform a type check since the final type has yet to decided.
First upon calling the `Open` method the type decided and checked. The type check comes from the attempt to instantiate a record instance of a type with the id `123456789`.

And there is the answer to the question, a RecordRef is just a dynamically typed Record with a thin wrapper over it and a corresponding wrapper for its fields (FieldRef).
Most of the methods will just check that the RecordRef is "opened" and then forward the API call to the underlying record.

## Bulk Inserts

The second question was about bulk inserts, both on its limitations and functionality.

Bulk inserts is a performance oriented feature that buffers up individual inserts and first inserts them upon reaching a pre-specified limit or when the platform determines an early flushing is necessary.
The feature is documented [here](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/administration/optimize-sql-bulk-inserts), so as always this post will focus on explaining the internals.

```AL
local procedure BufferedInsert()
var
    curr: Record Currency;
    i: Integer;
begin
    curr.FindFirst();

    for i := 0 to 100 do begin
        curr.Code := Format(i);
        curr.Insert();
    end;
end;
    ```
#### Listing 2: AL code with bulk inserts automatically applied by the runtime.

As shown in [listing 2](#listing-2-al-code-with-bulk-inserts-automatically-applied-by-the-runtime) nothing special is required from the AL developer to enable bulk inserts, rather there are certain which will disable bulk inserts, see [the below list](#disables-bulk-inserts).

### Disables bulk inserts
- Record is based on a non-SQL table.
- Table (or table extension) has Auto increment field.
- Table (or table extension) has blob fields.
- Return value of function call is used.

In the runtime the inserts are delayed by buffering the insert requests and first "flushing" them upon a limited is reached, or the runtime determines it must flush it to ensure correct results.

[Listing 3](#listing-3-sql-code-for-bulk-inserts-as-generated-by-the-runtime) shows the SQL code that is generated when flushing and writing to SQL, writing multiple rows in one SQL statement.
The main performance gains here come from minimizing the number of round-trips to the SQL Server instance, so if one's latency to the SQL Server instance is high, the gains will be significant.

```SQL
INSERT INTO Currency ("Code", ..., "$systemModifiedBy")
VALUES
(@0, ..., @36),
(@37, ..., @73),
(@74, ..., @110),
(@111, ..., @147),
(@148, ..., @184)
```
#### Listing 3: SQL code for bulk inserts as generated by the runtime.

The runtime will currently flush the buffered inserts in the cases described below (please remember this is an implementation detail which changes):
- Reaching the buffers limit.
- Calling Modify/ModifyAll/Delete/DeleteAll on any record of same type
- Calling Find/FinedSet/FindLast/FindFirst/Get/GetBySystemId/IsEmpty/Count/CalcFields on any record of the same type.
- Calling above methods on other record or query that reference the original table from flowfields.
- Calling CopyFields or CopyRows on the DataTransfer object with the table as source or destination table.
- Committing the transaction.
- Accessing a MediaSet value on a record of same type.
- Copying a record of the same type to a temporary record.
- Using TransferFields to move the fields on a record of same type.
- Accessing the timestamp or SystemId fields.
- Calling Modify or Delete on a company record.
- Exiting the running of a Codeunit.Run.

An interesting implication of this is that the errors which may happen during insert are also delayed until flushing, [listing 4](#listing-4-code-showcasing-delayed-insert-errors-due-to-bulk-insert) gives an example of this where the duplicate key error is first realized when a copying to a temporary table triggers the flushing.
In this case the flush is happening inside another method, but imagine the flush happening due to flowfields, where the error would happen on another record type, I can only imagine how horrible it would be to debug.

```AL
trigger OnAction()
var
    curr: Record Currency;
begin
    curr."ISO Code" := 'XBT';
    curr.Description := 'Bitcoin';
    curr.Symbol := '฿';
    curr.Code := 'BTC';
    curr.Insert();

    // Oops inserting the same Currency again?!
    curr.Insert();
    // But why not error? Bulk inserts!
    // The currency hasn't actually be inserted in SQL yet.

    // ...
    // Imagine this is happening many hundreds lines later
    // ...

    TriggerFlush(curr);
end;

local procedure TriggerFlush(var curr: Record Currency)
var
    tCurr: Record Currency temporary;
begin
    // Copy to temp which triggers a flush leading to a duplicate key error.
    tCurr.Copy(curr);

    // Imagine this code did something useful.
end;
```
#### Listing 4: Code showcasing delayed insert errors due to bulk insert.

Another interesting question regarding bulk inserts was posed by my editor [Natalie Karolak](https://twitter.com/KarolakNatalie) when she was reviewing this blog post:

What if there is code in the OnInsert trigger (directly, or indirectly by events)? Can this influence/stop bulk inserts?

The short answer is: no, the medium long answer is: They behave the same as without, but they can also trigger a flush.

```AL
[EventSubscriber(ObjectType::Table, Database::Currency, 'OnAfterInsertEvent', '', true, true)]
local procedure OnAfterInsertCurrency(var Rec: Record Currency)
begin
    Message('Inserted: ' + Rec.Code);
end;
```
#### Listing 5: Event subscriber on OnAfterInsert on currency showing the same Currency inserted twice.

However, more broadly how triggers and events interact with bulk inserts is interesting and worth exploring.
If we add an event subscriber as done in [listing 5](#listing-5-event-subscriber-on-onafterinsert-on-currency-showing-the-same-currency-inserted-twice) and call the action presented in [listing 4](#listing-4-code-showcasing-delayed-insert-errors-due-to-bulk-insert), the OnAfterInsert event subscriber will be triggered twice, both time suggesting a Currency with the primary key `BTC` has been inserted.
This really shows the importance of remembering that operations in AL are transaction and first upon commit we can be sure that something has happened.
