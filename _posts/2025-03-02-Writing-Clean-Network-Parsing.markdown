---
layout: post
title:  "Writing Clean Network Parsing"
date:   2025-03-02 19:23:11 +1100
categories: design patterns
---
### Intro

For whatever reason half my projects always seem to involve dealing with the same engineering problem. We recieve data from a client over a TCP/UDP connection that we then have to parse. Many network protocols don't have unambiguous delimiters so we may be looking at a partial message. So we need to store the data we recieve into a buffer and then attempt at least a single parse on recieved data.

So we can implement the first half this easily enough:

```csharp
    private async Task RespondAsync(TcpClient client)
    {
        // Setup the buffer and the accumulated request data
        var stream = client.GetStream();
        var buffer = new byte[8192];
        var bufferLen = 0;

        // Now keep reading in the socket data
        while (true) {
            var bytesRead = await stream.ReadAsync(buffer.AsMemory(bufferLen, buffer.Length - bufferLen));
            if (bytesRead == 0) break;
            bufferLen += bytesRead;
            // PARSING HERE
        }
    }
```

The parsing section of the code is where things get messy though. We need to call a function that tells us the following:

* Whether the buffer contains a valid message
* What that message is if it exists
* The number of bytes used to parse that message

Whilst this isn't the most difficult code in the world to write it can easily descend into a mess of passing a byte counter back and forth between funcions or unwanted allocations. We don't really want to pass raw buffer data right into a parser as it may mess with our data, but copying it creates an allocation right on the spot.

### Spans

For better or worse C# has a solution for literally everything, with Spans being fantastic for network programming. A span is a window over an array. It gives a standard interface for working with raw data and has support for immutability (which Go slices, for example, doesn't).

```csharp
    ReadOnlySpan<byte> mySpan = buffer.AsSpan(0, bufferLen);
```
A span is essentially just this: 

```csharp
    public ref struct Span<T>
    {
        internal readonly ref T _reference;  // Reference to first element
        private readonly int _length;       // Number of elements
        // ...
    }
```
For those not marinated in this stuff a *ref struct* is a reference to a struct (value type). The *ref* keyword forces the reference to stay on the stack. Moving Span onto the heap is unsafe in various situations (working with stack allocated memory, for example) so Spans are not compatable with all C# features (such as boxing).

For our purposes this is fantastic as we can pass a window of our buffer into our parser with mutability enforcement to boot.

### TryParse

My favourite pattern for writing parsers is TryParse. The function itself returns a boolean as to whether it parsed successfully or not, whereas the out variables give the consumed count and request.

```csharp
    public static bool TryParse(ReadOnlySpan<byte> data, out int consumed, out Request request)
    {
        // ...
    }
```
The reason this is so much nicer to work with than tuple return types is at the call site we can just do this:

```csharp
    if (!Parser.TryParse(span, out var consumed, out var request))
        break; // Need more data
```
It naturally captures the idea that we don't care about the return values consumed or request on failure. But I also love this pattern within the parser logic itself:

```csharp
    private static bool TryParseString(ReadOnlySpan<byte> data, int length, ref int consumed, out string value)
    {
        value = "";

        if (data.Length < length + 2) return false;
        if (data[length] != '\r' || data[length + 1] != '\n') throw new FormatException("Invalid string format");
        value = Encoding.UTF8.GetString(data[..length]);

        consumed += length + 2;
        return true;
    }
```
Here we are attempting to parse a string within our parser. Because we were given a *ref int* we can modify consumed directly to tell the caller how many bytes we used. Because we take in data as a span we can start parsing at index 0 without dealing with indices over a larger array.

As we chain these commands together the count of the consumed bytes moves forward automatically. Here is section of code parsing some data written in the redis protocol, which looks like this "$4/r/nsing/r/n":

```csharp
    for (var i = 0; i < arrayLength; i++)
    {
        // Consume the $ before the integer
        var span = data[consumed..];
        if (span.Length == 0 || span[0] != (byte)'$')
            throw new FormatException("Expected bulk string type");
        consumed++;
        // Get the data itself
        if (!TryParseInt(data[consumed..], ref consumed, out var length)) return false;
        if (!TryParseString(data[consumed..], length, ref consumed, out var str)) return false;
        result.Add(str);
    }
```
The way spans and consumed interact makes this elegant to write relative to other approaches i've tried.