---
layout: post
title:  "Grokking the Actor Model"
date:   2025-07-03 18:43:14 +1100
categories: design patterns
---
### Intro

Recently I was working on a programming challenge implementing the Bittorrent protocol and found myself constantly rewriting the download scheduler. It's a fairly simple technical problem to state but can be genuinely nasty to implement as it lends itself to deadlocks, data races and shared mutable state all while being terrible to debug.

The problem is we want to download a torrent:
- Each torrent has N pieces, and may represent a single or many files to download
    - Pieces can *traverse* files, so a single piece may contain the end of one file and the start of another
- To download a piece we break it into *chunks*, and download each chunk by requesting it from a peer
    - Once we have all the chunks for a piece it is complete - we need to validate its hash against the reference hash provided by the torrent file for correctness
    - Once we have all the pieces we have all the data. If we have multiple files we can break the contiguous data we downloaded into distinct files

That sounds relatively straightforward but the requirements for this to work well are not trivial:

* We need to maintain optimal throughput. [This paper](https://bittorrent.org/bittorrentecon.pdf) recommends we have five requests in our pipeline at a time currently pending (each request is typically 16kb)
* Peers may disconnect, send invalid data or be slower than others to respond
* Handle the tangled mess of shared state. We have chunks pending, pieces pending, peers handling different chunks and disk write ordering
* Handle backpressure as we may run into IO limitations on either the network or writes to disk. If the disk slows down for example we need to propogate this through our solution

Overall this is a problem that requires *a lot* of concurrency and a fairly complex synchonization mechanism to work without errors. We also want an approach that scales naturally.

### The Actor Model

I first learned about the actor model from working with Elixir/Erlang distributed systems in my spare time. Conceptually each "actor" handles processing and state management on its own. Actors then pass messages between eachother via their inboxes. This approach is extremely robust and offers fantastic horizontal scaling without the use of mutexes, locks or semaphores - all of which are very tricky to work with and reason about at scale.

But I think its best to see it in action. So lets solve our scheduling problem. We only need four actors for the simplest solution to this problem, each with its own responsibility:


| Actor        | Responsibility    |
|:-------------|:------------------|
| Piece Actor | Assembling and validating a piece |
| Peer Actor | Requesting chunks from a peer |
| File Actor | Writing a file to disk |
| Manager Actor | Orchestrating overall torrent download |

These actors communicate with eachother by passing messages. We can start with the messages we think we need:

```kotlin
    
data class RequestChunk(val pieceIndex: Int, val offset: Int, val length: Int) : DMMessage()

class BeginPieceDownload() : DMMessage()

data class ChunkData(val pieceIndex: Int, val offset: Int, val data: ByteArray) : DMMessage()

data class PieceCompleted(
    val pieceIndex: Int,
    val absOffset: Long,
    val data: ByteArray,
    val piece: TorrentPiece,
) : DMMessage()

data class FileCompleted(val pieces: List<TorrentPiece>) : DMMessage()

data class BadPiece(val pieceIndex: Int) : DMMessage()
  
```
We have six messages to make the requests needed to download a torrent

1. Request Chunk - request a chunk from a peer TCP connection
2. Begin Piece Download - begin downloading a piece, used for orchestration
3. Chunk Data - a downloaded chunk with the data
4. Piece Completed - a completed piece with the validated data
5. File Completed - a message that we successfully downloaded a file. Note that "TorrentPiece" is just metadata, not the data itself
6. Bad Piece: a message that a piece failed validation and needs to be redownloaded

Now lets begin designing the actors. The "manager" actor feels complex so lets start with the others first.

#### The Peer Actor

In the Bittorrent protocol we establish TCP connections with our peers via handshake. We then request chunks from those peers, who then respond with the data we requested. All actors must have an inbox - in our case a Channel that contains messages. Conceptually this actor does two things - request chunks from peers, then output those chunks to be used elsewhere. But *where* exactly? Who knows! At this stage we don't actually care. 

```kotlin
 class PeerActor(val peer: PeerConnection, val outbox: Channel<DMMessage>) {
    val inbox: Channel<DMMessage> = Channel(Channel.BUFFERED)
    // Inbox -> RequestChunk, request the chunk from peer
    // PeerConnection -> Recieve data from the peer, ChunkData -> Outbox
 }
```
This actor does two things. Upon recieving a RequestChunk message in its inbox, it requests the specified chunk from the peer over TCP. Upon recieving the data from its peer connection, it then writes the ChunkData message to the outbox. Thats it! 

For each peer we want to download from, we create a new PeerActor to manage that connection.

#### The File Actor

Once we download and validate pieces we need to write them to disk. The file actor recieves pieces and writes them to disk. Once it has downloaded all of the pieces the file needs, it reports this to its outbox. Note that since each file is controlled by a single actor we never have to worry about locks or mutexes here.

```kotlin
 class FileActor(val pieces: List<TorrentPiece>, val outbox: Channel<DMMessage>, val fileDir: String) {
    val inbox: Channel<DMMessage> = Channel(Channel.BUFFERED)
    val tracker: MutableMap<Int, Boolean> = mutableMapOf() // Tracks which pieces have been downloaded
    // Inbox -> PieceCompleted, writes the piece data to disk
    // If all pieces have been recieved & written to disk, FileCompleted -> Outbox.
  }
```

#### The Piece Actor

Each torrent piece contains many chunks we need to download - so this actor is responsible for generating the requests for the chunks we need whilst also assembling the downloaded piece and validating it.

```kotlin
  class PieceActor(private val piece: TorrentPiece, val outbox: Channel<DMMessage>) {
    val inbox: Channel<DMMessage> = Channel(Channel.BUFFERED)
    val data: ByteArray = ByteArray(piece.length)
    // Inbox -> BeginDownload, generate all the requests we need. RequestChunks -> Outbox
    // Inbox -> ChunkData, use the chunk data to assemble the piece. 
    // If we have all the chunks, validate the assembled piece and PieceCompleted -> Outbox.
    // If the piece validation failed BadPiece -> Outbox
```
Note this actor is only concerned with its own state, as are all the others. So we can manage downloading multiple pieces at a time quite easily by creating actors for each different piece we want to download.

#### The Manager Actor

This is the most complex actor of the four in that it creates as many of the other types of actors as it needs to download the torrent. The outboxes of the three actors above is this actors inbox. So the orchestration of the download is managed here.

```kotlin
  class ManagerActor(
      val torrent: Torrent, val peers: List<PeerConnection>, val downloadDir: String,
  ) {
    val inbox: Channel<DMMessage> = Channel(Channel.BUFFERED)
    suspend fun start() {
    // Create the file actors we need based on the torrent
    // Create the peer actors we need based on the peers list
    // Generate the list of pieces contained in the torrent to track completions
    // Create at least one piece actor
    // Instruct each piece actor to begin downloading. BeginDownload -> [Piece Actor]Inbox
    }
    
    // Inbox -> RequestChunk. Choose a peer at random and forward the message. RequestChunk -> [Peer Actor]Inbox
    // Inbox -> ChunkData. Forward to the relevant piece actor. ChunkData -> [Piece Actor]Inbox
    // Inbox -> PieceCompleted. Track this internally and forward to the file actor. Piece Completed -> [File Actor]Inbox. Generate a new piece actor and send it a begin download message. BeginDownload -> [Piece Actor]Inbox
    // Inbox -> File Completed. Check this off internally and shut down the file actor.
    // Inbox -> Bad Piece. Tell the relevant piece actor to re request all chunks. BeginDownload -> [Piece Actor]
  }
```
The start function kicks things off by creating the initial batch of actors we need. From there, we just loop on the inbox as we always do. I haven't specified here how we deal with disconnected peers or unresponsive peers here but the conceptual model doesn't change at all. We can scale this to as many peers, pieces and files as we like.

Note the use of channels creates a natural ability to throttle the system. With our file actor our inbox was specified this way:

```kotlin
    val inbox: Channel<DMMessage> = Channel(Channel.BUFFERED)
```
But we can change it to:

```kotlin
    val inbox: Channel<DMMessage> = Channel(1)
```
This would force the writes to be serialized. If the disk IO was slow, it would create natural backpressure:

* PieceActor finishes a piece → tries PieceCompleted to FileActor's inbox
* FileActor is busy writing previous piece → channel is full (capacity 1) → the send suspends
* Because the send suspends, the ManagerActor is also suspended when forwarding messages, which in turn slows down new chunk requests
* This slows PieceActors from requesting more chunks → automatically throttling download rate to what the disk can handle

The channels themselves allow for the system to throttle itself appropriately over time.