---
title: "On Records"
author: PhDuck
date: 2021-12-31
draft: false
---

Warning: Most  of the article discusses implementation details  for which we reserve the right to alter at any given point. Simplifications has been made for the sake of brevity and finally no guarantees are established by this article.

## Introduction
The record type is arguably the most important data type in AL development, one can hardly imagine AL development without it. Furthermore, with its inherent use in reports and pages it leaves no doubt about its fundamental status.

Directions sessions have been dedicated to describing mere features supported by the record type. Therefore, this blog post cannot be expected to fully elucidate the matter, but rather it focuses on what has previously been shrouded in mystic: the internals of the record type.

My hope is that reading this post leaves you with a with a better understand of **tables** and their corresponding **record**. To achieve this, we will explore the connection between the table and the runtime's record type. We will also touch on the records runtime in-memory representation both its state and data. Finally, we will look at how records iterate over data in a performant manner, and how not to screw that up. 

## Records and their relationship to table
Before we dive into the structures necessary to represent the record at runtime, we must take a detour to define its relationship with its respective table definition.

## Table metadata
To support that the record type can represent both a Customer and a Currency, we need a great deal of flexibility. That main aspect comes through metadata which is derived from the table definition in e.g., Currency.Table.al (and potentially multiple table extensions CurrExt.tableextension.al).

For tables, metadata is made up of a structure called “MetaTable”, which the compiler emits based on the table’s schema, and the server subsequently consumes at runtime to create its own internal augmented representation.

```xml
<?xml version="1.0" encoding="utf-8"?>
<MetaTable MetadataVersion="130000" ID="4" Name="Currency" CaptionML="ENU=Currency" LookupFormID="5" TableType="Normal" CompressionType="Unspecified" Access="Public" PasteIsValid="1" LinkedObject="0" Extensible="1" ReplicateData="1" DataClassification="CustomerContent" DataPerCompany="1" SourceAppId="437dbf0e-84ff-417a-965d-ed2bb9650972" SourceExtensionType="2">
  <Fields>
    <Field Name="Code" ID="1" Datatype="Code" DataLength="10" OnValidate="1" SourceAppId="437dbf0e-84ff-417a-965d-ed2bb9650972" SourceExtensionType="2" DataClassification="CustomerContent" CaptionML="ENU=Code" NotBlank="1" FieldClass="Normal" DateFormula="0" Editable="1" Access="Public" Numeric="0" ExternalAccess="Full" ValidateTableRelation="1" />
    <Field Name="Last Date Modified" ID="2" Datatype="Date" SourceAppId="437dbf0e-84ff-417a-965d-ed2bb9650972" SourceExtensionType="2" DataClassification="CustomerContent" CaptionML="ENU=Last Date Modified" Editable="0" FieldClass="Normal" Access="Public" ExternalAccess="Full" ValidateTableRelation="1" SignDisplacement="0" />
    <Field Name="Last Date Adjusted" ID="3" Datatype="Date" SourceAppId="437dbf0e-84ff-417a-965d-ed2bb9650972" SourceExtensionType="2" DataClassification="CustomerContent" CaptionML="ENU=Last Date Adjusted" Editable="0" FieldClass="Normal" Access="Public" ExternalAccess="Full" ValidateTableRelation="1" SignDisplacement="0" />
    <Field Name="ISO Code" ID="4" Datatype="Code" DataLength="3" OnValidate="1" SourceAppId="437dbf0e-84ff-417a-965d-ed2bb9650972" SourceExtensionType="2" DataClassification="CustomerContent" CaptionML="ENU=ISO Code" FieldClass="Normal" DateFormula="0" Editable="1" Access="Public" Numeric="0" ExternalAccess="Full" ValidateTableRelation="1" />
    <Field Name="ISO Numeric Code" ID="5" Datatype="Code" DataLength="3" OnValidate="1" SourceAppId="437dbf0e-84ff-417a-965d-ed2bb9650972" SourceExtensionType="2" DataClassification="CustomerContent" CaptionML="ENU=ISO Numeric Code" FieldClass="Normal" DateFormula="0" Editable="1" Access="Public" Numeric="0" ExternalAccess="Full" ValidateTableRelation="1" />
    ...
```
Figure 1: Subset of the MetaTable for the Currency table emitted as an XML file from the compiler.

While the MetaTable is a huge structure with too many aspects and usages to cover here, a few common usages are:
1.	Creation and maintenance of the corresponding SQL table, columns, keys, indices, constraints. See e.g., the <company name>$Currency$437dbf0e-84ff-417a-965d-ed2bb9650972 table and all its properties in SQL.

2.	Type information on fields, which is e.g., used to answer if a given assignment or filtering on a record’s field is valid? Both the compiler and the server verify this. The compiler in VS Code when assigning a string to a decimal will not compile. If one tries to be too smart and avoids the compiler’s analysis using RecRef and FieldRef the server will throw an exception at runtime. See listing 1.

3.	How much memory should be allocated to hold the values read from SQL and how should they be read?
F.e. Option values are stored as an integral value in SQL, so at runtime the integer value is translated to the proper option.

```AL
trigger OnAction()
var
    fref: FieldRef;
    rref: RecordRef;
begin	
    rref.Open(Database::Currency);
    fref := rref.Field(10); // 10; "Invoice Rounding Precision"; Decimal
    fref.Value := 'Cake'; // Nonsensical! Server throws runtime error here!
end;
```
Listing 1: The error here is lucky since the underlying record buffer could hold an incompatible typed value, which is guaranteed to give infuriating errors down the line.

## Runtime representation:
Moving into the exclusive domain of the server/runtime/NST: execution of AL code and the creation of records in-memory. 

In our codebase the record code is abstractly split up into three parts: The state, its metadata, and the record buffer (data). In the previous sections we got acquainted with metadata, the following part will discuss its state and data.

## State
A record’s state pertains to its imperative nature, where AL code alters the state of the record which subsequent affects accesses to its data provider (normally SQL).
The state in question is filters, marks, fields to load, fields to calc, etc. and are applied via their appropriate record APIs.

SetFilter and SetRange are the most used API for modifying a record’s state, so further exploration is warranted. 

Below is a step-by-step look at what happens upon calling Item.SetRange("Unit Price", 0, 42); 
1.	The compiler emits the field number of the field ”Unit Price” (18) and the type, which the runtime then verifies and loads out the metadata for the field.
2.	The input arguments are mangled to their appropriate type. E.g., integral literals are cast to decimal in the given example.
3.	If the input values are string based their length is compared to the maximum length of the field to be filtered on.
4.	The filter is created and assigned to the current filter group.
5.	Update the list of fields need be loaded if using Partial Records.
6.	Result set enumerator is invalidated.

## Data
Following the usual program order for AL code, data operations come after state changes. So first, after the filters have been applied and the fields selected, finally we are ready to invoke our data provider via Get, Find, Delete, etc.

After calling the data provider the record has its data buffer populated. So next is to explore how data is stored and manipulated in-memory before and after calling the data provider.

The record’s data is stored in a structure called a RecordBuffer which is split into two parts:
```AL
trigger OnAction()
var
    Curr: Record Currency;
begin
    curr.Reset();
    // Read-only: <EMPTY>
    // Mutable  : <EMPTY>
    Curr.FindFirst();              
    // Read-only: 97432 | AED | AED | ... | United Arab Emirates dirham | ...
    // Mutable  : <EMPTY>
    Curr.Code := 'BTC';            
    // Read-only: 97432 | AED | AED | ... | United Arab Emirates dirham | ...
    // Mutable  : 97432 | BTC | AED | ... | United Arab Emirates dirham | ...
    Curr."ISO Code" := 'XBT';      
    // Read-only: 97432 | AED | AED | ... | United Arab Emirates dirham | ...
    // Mutable  : 97432 | BTC | XBT | ... | United Arab Emirates dirham | ...
    Curr.Description := 'Bitcoin';
    // Read-only: 97432 | AED | AED | ... | United Arab Emirates dirham | ...
    // Mutable  : 97432 | BTC | XBT | ... | Bitcoin                     | ...
    if Curr.Insert then;
    // Read-only: 97433 | BTC | XBT | ... | Bitcoin                     | ...
    // Mutable  : <EMPTY>
end;
```
Listing 2: Example AL code, with comments showing the values of the read-only and mutable part of the record’s buffer after each operation.

Calls to Modify and Insert uses the combination of values from the read-only and mutable part, mutable part takes precedence, to create the result to alter or insert with.

## ResultSet Enumerator
Previously I mentioned that SetRange invalidates the ResultSet enumerator, while it might have seemed as an off-hand remark for the sake of completeness, it is crucial for ensuring correctness.

The ResultSet enumerator is a [generator](https://en.wikipedia.org/wiki/Generator_(computer_programming)) that gets created based on the records state when executing a find type data request. For the SQL based data provider a SQL query will be generated based on the records state (filters), which supports forward iterations (calling Next). Since iteration merely reads the next results (row) it is far more performant than to re-create the SQL query and read one result.

Invalidation of the ResultSet enumerator happens if the record’s state changes, in one of the following ways. It is done since the previous generator is based on the previous record’s state and therefore it cannot be guaranteed to still be producing the correct results. Avoiding invalidations when looping over tables is crucial, especially for the SQL data provider.

State changes that lead to the ResultSet enumerator being invalidated.
1.	SetRange/SetFilter/CopyFilter/SetSelectionFilter/ security filter changes
2.	SetCurrentKey
3.	TransferFields/CopyRecords (some)
4.	Marks changes
5.	SetAutoCalcFields
6.	SetLoadFields/AddLoadFields
7.	Changing the value of a primary key field
8.	Calling Reset() or Clear(rec)

```AL
trigger OnRun()
var
    Item: Record Item;
    tb: TextBuilder;
begin
    tb.Append('<>');
    Item.SetRange(Item."Unit Cost", 0, 42);
    if Item.FindSet() then begin
        repeat
            if Item."Unit Volume" = 0 then begin
                tb.Append(Item."No.");
                Item.SetFilter(Item."No.", tb.ToText()); // <-- Invalidates the result set.
                tb.Append('&<>');
            end;
        until Item.Next = 0;
    end;
end;
```
Listing 3: Example code that invalidates the Item records ResultSet enumerator by changing filters while iterating.

## End
As Richard Feynman once said: [I gotta stop somewhere, leave you something to imagine.](https://youtu.be/DZGINaRUEkU?t=197)