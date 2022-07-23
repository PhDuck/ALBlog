---
title: "Benchmarking AL code"
date: 2022-06-07
draft: false
---

# Introduction
Many a developer has likely wondered the cost of discrete primitives in their language, e.g., what does an assignment cost in C# or AL? With normal programming languages these costs are relatively well defined since they often map directly down to assembly instructions with [‘simple‘ costing models](https://docs.microsoft.com/en-us/previous-versions/dotnet/articles/ms973852(v=msdn.10)#what-things-cost-in-managed-code). Further researchers are doing independent measurements on the timings of assembly instructions, see e.g., Agner Fog’s [4. Instruction tables](https://www.agner.org/optimize/#manuals) which is by many considered the gold standard source on measurements of instructions on various hardware architectures.

In this blog post we attempt to write a simple micro-benchmark to test execution time of a trivial scenario. Throughout the post the benchmarking code will evolve as the common mistakes are presented, explained, and solved. Finally, a relatively good solution will be presented for public usage together with advice on how to run these benchmarks.

# Goals
Much of optimization work focuses tirelessly on bringing down a single important metric: time. The continued focus on this is completely understandable given the amount of [academic literature showing positive effect of lowering of absolute execution time](https://lawsofux.com/doherty-threshold).

However, that leaves a plethora of metrics ignored and forgotten. Earlier times developers had more focus on their application’s memory footprint since it was a scarce resource. Other aspects could be CPU cycles, disk read/writes, wait time, [garbage collection](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)), etc. These are theoretically also scarce[0], but in practice the end user will have experience of these.

However, the developer who wants to investigate why an execution time is high, should investigate based on what the operation does, not just that it takes a long time. Metrics about allocations, garbage collection, disk writes, etc. are crucial for abstractly explaining what a given operation did.

In a world where climate change is a reality and reducing one’s emission footprint, one could imagine optimizing for the lowest energy consumption, which is already an important factor in data center operations.

# Writing a benchmark
Let us imagine writing a simple micro-benchmark testing if a string matches the 24-hour format ([h]h:mm), starting with the recently introduced “Regex” codeunit.
```AL
trigger OnRun()
begin
    TooSimpleBenchmark();
end;

local procedure TooSimpleBenchmark()
var
    tid: Time;
    durr: Duration;
begin
    tid := Time;

    TestIf24HourFormatRegularExpression('23:59');

    durr := Time - tid;
    Message(Format(durr));
end;

local procedure TestIf24HourFormatRegularExpression(input: Text): Boolean
var
    regex: Codeunit "Regex";
begin
    exit(regex.IsMatch(input, '([01]?[0-9]|2[0-3]):[0-5][0-9]'));
end;
```
Listing 1: A too simple micro-benchmark supposed to measure the cost of using regular expressions to check if a string matches a given pattern. Don’t use this code.
```powershell
Invoke-NavCodeUnit -CodeunitId 50102 -CompanyName 'CRONUS International Ltd.' -ServerInstance PlatformCore
[2022-05-06 02:56:03] Invoking codeunit '50102' on tenant '' on server 'PlatformCore'.
WARNING: 206 millisecond
Invoke-NavCodeUnit -CodeunitId 50102 -CompanyName 'CRONUS International Ltd.' -ServerInstance PlatformCore
WARNING:
```
Listing 2: Execution of our simple benchmark, with the second line showing the execution time.

206 milliseconds is an unimaginably long time, in the time 100 000 rows can be queried in SQL, a round-trip internet request over consumer grade network with 17 hops to South Korea responds faster! We have just fallen into the trap of our first run measuring the cost of loading and compiling (one time overhead) the code for our method “TestIf24HourFormatRegularExpression” and we can easily fix that by calling the method before starting out measurement to remove first-run overhead.

Running it a second time returns empty value? The second value isn’t much better, nothing is free. So, unless the code didn’t run, our measurement clearly wrong. The problem we are running into is that the [Time data type]( https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/time/time-data-type) in Business Central is limited to a millisecond resolution and our simple test executes faster than that.

## Attempt 2: Getting it to work
With a few assumptions[1] and under these we can run the same code many times and just take the arithmetic mean to approximate the execution time of the method.

Another positive aspect of running the test a significant number of times is that potential variance that only happens every *n* times will be captured. If every iteration execution time is captured, one can later inspect them and look for potential outliers.

```AL
local procedure ManyIterations()
var
    i: Integer;
    limit: Integer;
    tid: Time;
    dur: Duration;
begin
    // Remove first-run overhead.
    TestIf24HourFormatRegularExpression('23:59');

    limit := 100000;
    tid := Time;

    for i := 0 to limit do begin
        TestIf24HourFormatRegularExpression('23:59');
    end;

    dur := Time - tid;
    Message('Total time: ' + Format(dur) + ' Per op: ' + Format(dur / limit) + ' milliseconds');
end;
```
Listing 3: A slightly improved version of our benchmarking code, removing first-run overhead and working around the limited resolution of Time.

```powershell
Invoke-NavCodeUnit -CodeunitId 50102 -CompanyName 'CRONUS International Ltd.' -ServerInstance PlatformCore
[2022-05-06 02:59:03] Invoking codeunit '50102' on tenant '' on server 'PlatformCore'.
WARNING: Total time: 1 second 912 milliseconds Per op: 0.01912 milliseconds
```
Listing 4: Shows the execution command of the benchmark and corresponding output.

The results of our second attempt at benchmarking show why our first attempt wasn’t successful, the execution time of one operation is ~19 μs. For the average person a microsecond is almost impossible to comprehend since nothing in our frame of reference operates in the microsecond time range.

## Attempt 3: Measuring what matters 
A seasoned performance developer will recognize that 19 μs is relatively slow and will wonder where the time is spent? I therefore pose a question for the reader to ponder on, which of the following costs the most?
- The act of calling the method TestIf24HourFormatRegularExpression (not executing its content).
- The creation of the codeunit “Regex” and codeunit “Regex Impl.”.
- Executing the code within TestIf24HourFormatRegularExpression which entails the creation of the Regex dotnet object and executing the corresponding [dotnet method]( https://referencesource.microsoft.com/#System/regex/system/text/regularexpressions/Regex.cs,539fad5e1b3e233a,references).
Quick testing shows that removing the method call overhead is ~1 μs. While removing the re-creation of the codeunits saves about ~3 μs.
## Attempt 4: Measure your assumptions
The documentation states that AL’s RegEx library provides an option for compiling the regular expression exists, which should make the execution of the regex faster. Unfortunately, this will make the initialization slower, so we cannot use it as a proxy for estimating if the execution or initialization is the bottleneck. So, the overall execution time might get faster, right?

The output shows the execution time jumps to ~1.7 milliseconds, 100 times slower than our previous best. This behavior is documented and the initialization times for a compiled regex is far larger than the matching times of such simple regular expressions as ours.
```AL
    regexOptions.Compiled := true;
    for i := 0 to limit do begin
        b := gRegexComp.IsMatch('23:59', '([01]?[0-9]|2[0-3]):[0-5][0-9]', regexOptions);
    end;
```
Listing 5: The updated code instrumenting the regex to compile the pattern into code, read more [here](https://docs.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-options#compiled-regular-expressions).
## Reuse of precomputed versions:
 With the understanding that the expensive part isn’t the method call or codeunit creation we are stuck with it either being the regex matching algorithm or the re-creation of the underlying regex object and its pattern. The AL implementation allows for creating a regex instance once and reusing it, see listing 6. The output of running this gives an average execution of ~2 μs which means we can finally deduce the creation of the regex object and its pattern was the most expensive part.
```AL
    gRegex.Regex('([01]?[0-9]|2[0-3]):[0-5][0-9]');
    for i := 0 to limit do begin
        b := gRegex.IsMatch('23:59');
    end;
```
Listing 6: The updated code which creates the underlying regex instance and pattern once. 

## Caveats
The following is a non-exhaustive list of common mistakes one should consider when benchmarking in AL, or in any language.

### Consider your input
In our previous examples we have only used a nice and short matching string, however regular expressions are quite known for being inherently complex and leading to denial-of-service attacks ([ReDos](https://en.wikipedia.org/wiki/ReDoS)), but their execution time also increase with the texts input length. Running the same input on a string with 13000 characters yields an execution time of ~620 μs. Furthmore, certain special cases might exist for certain inputs so testing with inputs that reflect real-life application is prefered.

### Caching of results
In the NST the server automatically caches the results of record-based database operations. So, testing the performance of calling GET on a record in a loop will not yield useful measurements about the cost of getting the row from SQL. The NST does provide an API for requiring results to be the latest : [SelectLatestVersion]( https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/database/database-selectlatestversion-method).

### Timings
Throughout this blog post I have given absolute execution timings, but unless you can replicate my machine, your execution will show different numbers. In practice, benchmarks will pit multiple possible solutions against each other, and there the ratio between them is more useful than the absolute timings.

### Execution environment
When running benchmarks, one will quickly find that the absolute numbers provided between runs will vary, anywhere from a few ns to μs depending on what else is currently executing on the machine. Therefore, it isn’t recommended to run this sort of benchmarking in SaaS since other users’ operations may interfere making it impossible to get consistent results. As an example, I ran the famous [Prime95](https://www.mersenne.org/download/) on my computer while running the benchmark which lead to the execution time per operation doubling.

## Final code:
A final version of the benchmarking code can be found [here](https://gist.github.com/PhDuck/a8e8a6d7cd2f68255ed454777cbafea6). It uses a dynamic approach to solving the time resolution problem and saves executing timings allowing for inspection of the distribution. It also includes a solution which does not rely on regular expressions but rather tedious handwritten parsing, which executes in sub-microseconds (~650 ns) but isn’t very maintainable nor have it been tested, so please don’t use in production.

## Exiting tooling
Fortunately, the problems and solutions above are a well-known and have been realized by many developers before, so therefore tools to alleviate these problems have been developed for major languages, see [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) for .NET and [jhm](https://github.com/openjdk/jmh) for Java.

# Conclusion
I hope this blog post illuminated the problem space of benchmarking code, hopefully the reader understands that this isn’t a trivial problem nor does trivial solutions exist. I encourage the curious reader to consider reading Andrey Akinshin’s excellent book [Pro .NET Benchmarking](https://aakinshin.net/prodotnetbenchmarking/) which will further illustrate  complexities and show how our solution isn’t even close to perfect.

[0]: Blocks on an SSD has endurance rating for [how many writes](https://en.wikipedia.org/wiki/Flash_memory#Write_endurance) can be done, after which the degrade.

[1]: We make the semi-reasonable assumption that multiple executions of TestIf24HourFormatRegularExpression(…) will lead to a unimodal distribution. Semi-reasonable since the code is the same for each execution, however elements like garbage collection are beyond our control and may lead to executions that take substantially longer, leading to bimodal or multimodal distributions. Better tools should present the distribution f.e. with a histogram allowing for inspection of the results.

[2]: We could use a .NET profiler and thereby measure the execution time of the individual elements. I will leave that for another time in another post.
