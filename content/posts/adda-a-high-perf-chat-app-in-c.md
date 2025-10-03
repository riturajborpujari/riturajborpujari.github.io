---
title       : "Building Adda: A high performance group chat application in C"
description : >
              Learning Network programming and creating a multi-user 
              single-group chat application with C standard library. Uses Posix
              threads and IO Multiplexing to handle thousands of users 
              simulataneously with very minimal resource usage.
date        : 2025-03-14
template    : post.html
---

## Introduction
I don't use C at my work but I do occasionally get back to this fantastic
language. Some time back I developed a Terminal alias to [manage my directory
bookmarks and allow faster
navigation](/posts/terminal-tools-shell-aliases-directory-bookmarks/) using
`fzf`. I was intrigued by the Unix Philosophy and its overall design.So I began
looking at different syscalls Linux provides and understanding some of them. I
even tried out building C programs to replicate behaviours of everyday programs
like `cat`, `cp` etc.

Then I came to a domain which I have never touched in C before: Network
Programming. I was kind of scared of that to be honest. But nevertheless I loved
Network programming. I am, after all, a Web Developer who builds and maintains
Web APIs.

## Building it out
But I wanted to try it out now. I have a copy of [Unix Network
Programming](https://en.wikipedia.org/wiki/UNIX_Network_Programming) and I
referenced that for most parts. But I frequently used Man pages as it helped me
look up things faster once I know the name of the function 

Working my way forward with the book and man pages, I was able to create a
socket, `bind` it to address `0.0.0.0` (any IP on my local machine) with a
specified port and `listen` to it. The **network ordering** of addresses and
ports required some read up and I implemented the order conversion functionality
myself before I actually stumbled upon `htonl`, and `htons` macros.

With that out of the way, I was able to create a server which listens on a port
for TCP connections, `accept`s the first one and echoes whatever data the client
sends to it. For client program, I still relied on NodeJs.

### Concurrency and IO Multiplexing
One thing I really like about the man pages is that every page had a little `SEE
ALSO` section at the end. Without that I wouldn't have found `select` function.
`select` led me to `poll` and soon I enhanced my echo server to use IO
Multiplexing to support multiple clients concurrently, all with a single thread.
I later saw this in the book which explained concepts in detail.

Now I started forwarding messages from one client to all others and I've got a
bare-bones chat server instead of a concurrent echo server. Whenever one client
is ready with some data to be read, (as `poll` tells me this) I can then loop
over all the clients again and forward the message (`write`) it to all

The `poll` method only polls on the valid file descriptors of the client sockets
provided. Any invalid descriptor (_negative value_) would be ignored. This
helped me immensely. Since I could have a list of fds stored on the stack itself
and when a client disconnects, I just simply set the relevant fd in the list as
`-2`. The message writing logic then can simply skip over all the client fds in
the list which have value as `-2`

### Performance and Threading
Now all of this is great, but the performance is a bit slower. Handling 2000
concurrent connections takes a bit more time than expected. Time to employ
threading.

I referred back to the book and found [Posix Threads](https://en.wikipedia.org/wiki/Pthreads). After trying it out, I laid out my design like this
1. main thread to poll for new connections
2. *message_reader* thread to poll for messages from clients
3. *message_writer* thread to write available messages to all clients (except
   the author)

I reused the same list of fds I used to poll in both *message_reader* as well
*message_writer* to let both the threads know the list of clients. main thread
would accept new connections and add it in this list itself.

The remaining part was to find a way to transfer messages from reader to writer.
I used another file to act as a channel for the messages. That way I don't need
to handle locks and can also poll on the channel file in *message_writer*

So the *message_reader* thread would poll on the clients for data to be read,
and once available it would read and write it just to the channel file. A single
write operation per read message while iterating over messages to be read from
all clients meant that the server could now read faster.

On the other hand, *message_writer* would poll only on the channel file, and
once data is available it would read it and write it to all clients in the list. 
The ordering of messages is also preserved this way. The message read first is
the message that is forwarded first to all clients

This is great. The message throughput increased significantly and I was happy.
But I soon saw a peculiar thing. Even for zero clients, the cpu usage was pretty
high. I had not enabled Non-Blocking IO on any file descriptor so `poll` should
block until data is available on the provided fds. And since reader has a list
of zero fds, it is out of the question. It is configured to use poll with a
timeout of 500 ms so as it uses the latest list of fds everytime, but that
should not bring the CPU usage to much higher numbers.

Its obvious that *message_writer* is to blame. And I found out the `poll`
function is returning immediately for the channel file saying it has data to be
read (`POLLIN` event is being raised). But the subsequent `read` returned zero
bytes. So it looped over the poll-read logic continuously. This is causing the
high CPU usage. I looked into man page of `poll` for info and I found out the
thing I skipped earlier. 
> Being "ready" means that the requested operation will not  block;  thus, 
poll()ing regular  files,  block devices, and other files with no reasonable
polling semantic always returns instantly as ready to read and write

Okay so using a channel file is not possible. After reading about Unix Sockets
(sockets for Interprocess communication) I came to this amazing function
`socketpair`. It creates a pair of sockets which are connected to each other.
So, data written to any of them is available on the other. Perfect!

I updated my channel file logic to use channel socket pairs and soon everything
came back to normal. CPU Usage is `Zero` for zero clients and it also increased
predictively now as the number of clients increased

## Benchmarks
The program acheives the following benchmark result on my local machine

- CPU: Ryzen 3700x (8 cores 8 threads)
- RAM: 16GB
- Message length: 211 bytes

Note: CPU usage is divided by the number of cores (`top` - Iris Mode off)

| Number of Clients | CPU Usage | RAM Usage | 1M messages write |
| ---  | --- | ---   | --- |
| 0    | 0   | 1.6m  | NA |
| 500  | 1.3% | 1.6m  | ~4 seconds
| 1000 | 4.9% | 1.6m  | ~1.3 seconds |
| 2000 | 5.6% | 1.6m  | ~0.9 seconds |
| 4000 | 5.3% | 1.7m | ~1.5 seconds |

## Conclusion
The server is working fine now. Obviously this is not ready for production usage
as it would have to be inspected for error scenarios, edge cases and security
considerations. And its feature list is still at the beginning.

I named it **Adda** a word popular in Assamese and many other languages in
India, which roughly translates *to hang out / chat with a group of friends* 

This was built as a fun project and I enjoyed building this and learned a lot
about Network Programming and also C in general.

In future I would try to enhance this to allow command parsing and User
authentication to help better suit the use case. For now, I am happy with what
it is!

## Project repository
The project repository is available on Github here at [Adda](https://github.com/riturajborpujari/adda)

## References
1. [https://en.wikipedia.org/wiki/UNIX_Network_Programming](https://en.wikipedia.org/wiki/UNIX_Network_Programming)
2. [https://en.wikipedia.org/wiki/Pthreads](https://en.wikipedia.org/wiki/Pthreads)
3. [https://www.man7.org/linux/man-pages/man1/top.1.html](https://www.man7.org/linux/man-pages/man1/top.1.html)
