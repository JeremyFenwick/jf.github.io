---
layout: default
title: Home
---

# Jeremy Fenwick's Github Nav

## CODECRAFTERS 

Codecrafters offers a variety of challenges where they generate all the tests for you, so it's about incrementally passing each stage until you have a complete solution. These challenges are interesting because they generally require building a replica of some well known technology. 

### REDIS (WIP)

The largest project offered by Codecrafters which is to build a full Redis clone. I restarted this recently as I wanted to rebuild the entire architecture based on my first draft. Real redis is single threaded, and so is my solution at least with respect to the underlying database. 

The central Key-Value data store is built around and event loop, where we process a single message at a time. This prevents data races and such, but we still use async/await for all the networking. C#'s abstractions around Tasks make blocking and timeout requests fairly painless:

```csharp
// Blocking Pop command with optional timeout
public record BlPop(string Key, int TimeoutMs = 0) : Request(), IWithTaskSource, IHasKey
{
    public TaskCompletionSource<string?> TaskSource { get; } = new();

    public void SetException(Exception exception)
    {
        TaskSource.TrySetException(exception);
    }
}

// At the data structure call site we set the timer. Note the use of TrySetResult, we have a race 
// between the timer and the event queue, but TrySetResult makes this perfectly fine. Either the 
// timer or the event queue gets there first, which is a natural expression of the problem.
// 
// Note the timer may stick around for a little while upon completion via the event queue, but the
// cost is so marginal trying to improve this is premature optimization imo
public Task<string?> BlPop(BlPop blPop)
{
    if (_requestQueue.Writer.TryWrite(blPop))
    {
        // Set the timer if it exists
        if (blPop.TimeoutMs > 0)
            _ = Task.Delay(TimeSpan.FromMilliseconds(blPop.TimeoutMs))
                .ContinueWith(_ => blPop.TaskSource.TrySetResult(null));
        return blPop.TaskSource.Task;
    }

    throw new Exception("Failed to add BlPop request to queue");
}
```

* [Reaper - C# (In Progress)](https://github.com/JeremyFenwick?tab=repositories)

### BITTORRENT

Implements the Bittorrent protocol. This involved building a Bencode encoder/decoder (I never use libraries for these things) but the download scheduler was by far the most complex part of the build. It isn't required to pass the tests but I wanted to deal with disconnects, partial downloads, out of order downloads, scalable throughput etc. I used the actor model for safe concurrency.

*   [Broadstone - Kotlin](https://github.com/JeremyFenwick/Broadstone)

### SHELL

A very straightforward project - build a Shell clone! Where things get complicated is when we start chaining commands together and piping outputs. There were a lot of footguns around interfacing with the OS for this from the JVM. Used a good old fashioned Word Trie for text prediction.

* [Terminus - Kotlin](https://github.com/JeremyFenwick/Terminus)

### GREP

Another straightforward project - create a Grep clone. The trick here is to use an Abstract Syntax Tree and Tokenizer together to keep the design clean. Each node of the AST is encapsulated as it knows how to evaluate itself - where "evaluate" returns a list of valid *next* positions. Building the AST requires recursive descent, which I also used for the compiler in the Hack challenge.

```csharp
public abstract record AstNode
{
    // Returns the next valid positions (if any)
    public abstract List<int> Evaluate(string input, int currentPosition, CaptureContext context);
    public abstract void Print(int depth = 0);
}
```
Without a strong architecture the result is a descent into a madness, as you can see in some other completed solutions online =P

* [Repper - C#](https://github.com/JeremyFenwick/Repper)

## OTHER CHALLENGES

### PROTOHACKERS

Implement a variety of networking protocols for both UDP/TCP of increasing complexity. The later challenges have heavy performance restrictions and are fairly diabolical with their edge cases. I completed this in both Golang and C# with differing architectural approaches. C# uses an inheritance model with a base UDP/TCP server, where in Golang each solution is independent with no shared code. 

Golangs built in profiler was very nice here as it's a one liner and my solution was not running locally:

```go
import (
    // Profiler deps
    "log"
    "net/http"
    _ "net/http/pprof"
)

go func() {
    log.Println(http.ListenAndServe(":8080", nil)) // used for the pprof profiler
}()

```

Then profile with:

```shell
go tool pprof -http=:9090 http://<IP>:8080/debug/pprof/profile?seconds=30
```

To be fair dotnet-monitor does something similar (in theory!). Go has very weak data structure support relative to C#, I had to roll my own [thread-safe Max-Heap](https://github.com/JeremyFenwick/Firewatch/blob/main/internal/jobcenter/maxheap.go), for example.

*   [Firewatch - Golang](https://github.com/JeremyFenwick/Firewatch)
*   [Firewatch-C - C#](https://github.com/JeremyFenwick/Firewatch-C)

### HACK

The largest challenge i've attempted overall - build an entire computer up to the operating system from nothing but NAND gates. A very fun project. The bulk of the project is written in either HDL or the JACK language I wrote the compiler for, but for *that* I used good 'ol C#.

*  [Hack - C#/HDL/Jack](https://github.com/JeremyFenwick/Hack)

### APOLLO

The ray tracer challenge but I approached this with Test Driven Design to see how I liked it (I didn't actually like it that much... =( ). I implemented the math library myself which is not ideal for performance but forced me to relearn linear algebra.

*   [Apollo - C#](https://github.com/JeremyFenwick/Apollo)

### INFINIS

A maze generator and solver. An exploration of graph theory and various related algoritmms such as Djikstra's, Sidewinder, Aldous Broder, etc.

*   [Inifis - C#](https://github.com/JeremyFenwick/Infinis)

