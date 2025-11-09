---
layout: post
title:  "Writing Clean Network Parsing"
date:   2025-03-02 19:23:11 +1100
categories: design patterns
author: JF
tags:
  - C#
  - networking
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

Whilst this isn't the most difficult code in the world to write it can easily descend into a mess of passing a byte counter back and forth between functions or generating unwanted heap allocations. We don't really want to pass raw buffer data right into a parser as it may mess with our data, but copying it creates an allocation right on the spot.

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
    value = ""; // Equivalent to string.Empty. No allocation!

    if (data.Length < length + 2) return false;
    if (data[length] != '\r' || data[length + 1] != '\n') throw new FormatException("Invalid string format");
    value = Encoding.UTF8.GetString(data[..length]);

    consumed += length + 2;
    return true;
}
```
Here we are attempting to parse a string within our parser. Because we were given a *ref int* we can modify consumed directly to tell the caller how many bytes we used. Because we take in data as a span we can start parsing at index 0 without dealing with indices over a larger array.

Here is an even simpler component of the parser:

```csharp
private static bool TryParseStringStart(ReadOnlySpan<byte> data, ref int consumed)
{
    if (data.Length == 0) return false;
    if (data[0] != '$') throw new FormatException("Expected bulk string type");
    consumed++;
    return true;
}
```

As we chain these commands together the count of the consumed bytes moves forward automatically. Here is section of code parsing some data written in the redis protocol, which is formatted like this -> **"$4/r/nsing/r/n"**:

```csharp
for (var i = 0; i < wordCount; i++)
{
    // Consume the $ before the string length
    if (!TryParseStringStart(data[consumed..], ref consumed)) return false;
    // Get the string length
    if (!TryParseInt(data[consumed..], ref consumed, out var length)) return false;
    // Get the string itself
    if (!TryParseString(data[consumed..], length, ref consumed, out var str)) return false;
    result.Add(str);
}
```
The way spans and consumed interact makes this very elegant to read and write relative to other approaches i've tried.

We can accomplish something similar in go with references. This is code that encodes a http response header whilst avoiding generating too many unwanted allocations

```golang
// Encode - turn the response into a byte array
// Designed to not generate unwanted allocations
func (response *Response) Encode() []byte {
	buffer := response.sizedBuffer()
	index := 0

	// Status line
	loadString(buffer, "HTTP/", &index)
	loadString(buffer, response.Version, &index)
	loadString(buffer, " ", &index)
	loadInt(buffer, response.Status, &index)
	loadString(buffer, " ", &index)
	loadString(buffer, response.Reason, &index)
	loadString(buffer, "\r\n", &index)

	// Headers
	for k, v := range response.Headers {
		loadString(buffer, k, &index)
		loadString(buffer, ": ", &index)
		loadString(buffer, v, &index)
		loadString(buffer, "\r\n", &index)
	}
	loadString(buffer, "\r\n", &index)

	// Body
	if len(response.Body) > 0 {
		copy(buffer[index:], response.Body)
	}

	return buffer
}

func (response *Response) sizedBuffer() []byte {
	size := 0
	// Status line
	size += 5 + len(response.Version) + 1 // "HTTP/" + Version " " "
	size += digits(response.Status) + 1   // Status + " "
	size += len(response.Reason) + 2      // Reason + "\r\n"

	// Headers
	for k, v := range response.Headers {
		size += len(k) + 2 + len(v) + 2
	}

	// Blank line "\r\n"
	size += 2

	size += len(response.Body)
	return make([]byte, size)
}

func loadString(buffer []byte, str string, index *int) {
	copy(buffer[*index:], str)
	*index += len(str)
}

func loadInt(buffer []byte, num int, index *int) {
	if num == 0 {
		buffer[*index] = '0'
		*index++
		return
	}

	digitsCount := digits(num)
	for i := digitsCount - 1; i >= 0; i-- {
		buffer[*index+i] = byte(num%10) + '0'
		num /= 10
	}
	*index += digitsCount
}

// Count digits in an int
func digits(n int) int {
	if n == 0 {
		return 1
	}
	d := 0
	for n > 0 {
		n /= 10
		d++
	}
	return d
}
```