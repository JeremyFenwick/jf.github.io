---
layout: post
title:  "List Allocation Rabbit Hole"
date:   2024-12-10 14:13:55 +1100
categories: weirdness
author: JF
tags:
  - C#
  - data structurs
  - but why?
---
### Lists, how do they *** work!?

I was joking the other day that C# is so complicated creating a list requires a deep knowledge of interfaces. It turns out, that's true! Here is a basic problem - we need to return an empty list from a function and we are concerned about the allocation cost because I am surrounded by crazy people. Here is some innocent looking code:

```csharp
 private void HandleLPop(LPop pop)
{
    if (_dataStore.TryGetValue(pop.Key, out var value) && value is RedisList list)
    {
        // Other stuff
        pop.TaskSource.TrySetResult(result);
    }
    else
    {
        pop.TaskSource.TrySetResult(new List<string>()); // Heap Allocation!
    }
}
```

Poking around one solution is to use an IEnumerable instead. Why? Because we can do this:

```csharp
pop.TaskSource.TrySetResult(Enumerable.Empty<string>()); // No heap allocation!
```
Apparently the compiler optimises this for you not actually allocating any memory - interesting! The problem is, IEnumerable is a limited interface. To get **count** for example we have to iterate over the entire data structure which is O(N) for what would be O(1) with a list. 

So another idea i've seen used is to use a statically allocated List over and over again.

```csharp
private static List<string> _thisIsEvil = []; // One time allocation!
```

This is a one time allocation but the problem is someone can start adding to the list somewhere down the line which will weack havok on your codebase, its just terrible design. 

So apparently there is a solution! There is another interface *IReadOnlyList* which is self explanatory I think. However, if we do this:

```csharp
pop.TaskSource.TrySetResult(Array.Empty<string>()); // No allocation!
```
Since its read only it does make sense that it satisfies the interface. Also in modern C# you can just use the shorthand so you don't have to remember the Array.Empty thing. 

```csharp
pop.TaskSource.TrySetResult([]); // No allocation!
```

Using immutable data structures is something i'm fond of anyway, I just never remember to specify the read only interface. Its an area where Kotlin's immutable by default approach makes this stuff much simpler.

Anyway, Interesting!

