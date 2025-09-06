---
editPost:
  URL: "https://github.com/buraksekili/buraksekili.github.io/blob/main/content/articles/inodes-filedescriptors-sockets.md"
  Text: "Edit this page on "
author: "Burak Sekili"
title: "Notes on i-nodes, File Descriptors, and Sockets"
date: "2025-09-06"
description: "A Developer's Notes on Inodes, File Descriptors, and Sockets"
tags: ["linux", "file", "networking"]
TocOpen: true
---

I've always found the best way to learn something is to try and write it down. This post is the result of that process, a collection of my personal notes (zettelkasten) aimed at connecting the dots between three fundamental concepts in Linux: inodes, file descriptors, and sockets. So, this post is just a cleaned-up version of my personal notes, explaining how inodes (representing files on disk), file descriptors (used by programs), and sockets (for network communication) all fit together.

## i-nodes

Every file and directory in a filesystem is represented by a data structure called an inode. The inode stores metadata about the object, such as its permissions, attributes, and the disk block locations of its data. In essence, an inode contains pointers to the actual data blocks on the disk, along with the file attributes.

![inode-table](/images/inodes-filedescriptors-sockets-1#center)

This is an overly simplified reference of the inode table and data blocks. In real scenarios, since a file can be larger than a single disk block (typically 4KB or 8KB), the inode stores a series of pointers to all the data blocks that constitute the file. This allows the system to assemble the complete file from its scattered blocks.

An inode also includes a reference count, which tracks the number of links pointing to it. When you create a link to a file, the system does not create a new inode or copy the data. Instead, it creates a new name entry in a directory and increments the reference count of the existing inode. The file's data blocks are only deallocated when this reference count drops to zero. Removing a file link, an operation known as "unlinking", decrements this count.

The following example demonstrates this concept. First, let's inspect a new file:

```
$ ls -il   
total 0  
3558217 -rw-rw-r-- 1 burak burak 0 Sep  6 17:54 document.md  
```

The first column in the output is the inode number (`3558217`), and the third column is the reference count (`1`). Now, let's create a hard link to the file:

``` 
$ ln document.md document-linked.md  
$ ls -il   
total 0  
3558217 -rw-rw-r-- 2 burak burak 0 Sep  6 17:54 document-linked.md  
3558217 -rw-rw-r-- 2 burak burak 0 Sep  6 17:54 document.md  
```

After creating a hard link named `document-linked.md`, the output shows that both filenames point to the same inode number. The reference count for that inode has now increased to `2`.

Finally, if one of the links is removed, the reference count is decremented.

```
$ rm document-linked.md && ls -il  
total 0  
3558217 -rw-rw-r-- 1 burak burak 0 Sep  6 17:54 document.md  
```

The inode and its data persist as long as the reference count is greater than zero.

## File Descriptors

A file descriptor is a small, non-negative integer that acts as a handle for an open file. When your process opens or creates a file, the kernel returns this number, which your program then uses to perform all further operations, like reading from or writing to that file.  
There are special file descriptors for processes (except daemons) in Linux: 0 (stdin), 1 (stdout), and 2 (stderr), called stdin, stdout, and stderr. By convention, the shell automatically opens these three standard file descriptors for every new process it starts:

Because these first three numbers are reserved, the first file you open in a program will typically be assigned the next available file descriptor, which is `3`.

The `open()` system call manual provides a clear definition of a file descriptor:  

> “The return value of open() is a file descriptor, a small, integer that is an index to an entry in the process's table of open file descriptors. The file descriptor is used in subsequent system calls to refer to the open file. The file descriptor returned by a successful call will be the lowest-numbered file descriptor not currently open for the process.” [^1](https://man7.org/linux/man-pages/man2/openat.2.html#NAME)

As this description states, a file descriptor is an integer that serves as an index in a per-process table. The kernel maintains a separate file descriptor table for each process, and the number returned by open() is simply the lowest available index in that table. If the kernel cannot open a file, it will typically return -1 to indicate an error. Because each process has its own table, a file descriptor in one process is independent of those in another. 

This per-process table, however, is only the first part of the structure. Each entry in the file descriptor table points to a second data structure: the system-wide open file table. An entry in this table, often called an "open file description," records information like the current file offset and status flags (e.g., read-only, append-only). A file descriptor's reference to an open file description is unaffected even if the original file path is later removed or changed.

Finally, each entry in the system-wide open file table points to the file's inode, which contains the metadata and pointers to the actual data on disk.

With that being said, we can extend our simplified drawing above as follows:

![fd-table](/images/inodes-filedescriptors-sockets-2#center)

The relationship between these tables is important when a process is forked. When fork() is called, the child process receives a copy of the parent's file descriptor table. Crucially, the corresponding file descriptors in both the parent and child tables point to the same entry in the system-wide open file table. This means that they share a file offset; if the child reads from the file, the offset advances for the parent as well.

In contrast, processes that are completely independent of each other have separate file descriptor tables that do not share these underlying open file descriptions.

## Sockets

Sockets provide a standard way for processes to communicate with one another, a mechanism known as Inter-Process Communication (IPC). This communication can occur between processes on the same host using Unix Domain Sockets (UDS), or between processes on different hosts across a network. Sockets are the fundamental application programming interface (API) for network protocols like TCP and UDP within the TCP/IP suite.

The three most common types of sockets are Stream, Datagram, and Unix Domain sockets. Working with them involves a set of key system calls, such as socket(), bind(), listen(), accept(), and connect().

The process begins with the socket() system call, which creates a communication endpoint and returns a socket descriptor. This descriptor is implemented as a file descriptor, allowing programs to use many standard file-related calls like read() and write() to send and receive data. However, not all file operations apply to sockets; for example, a syscall lseek() has no meaning for a socket stream and cannot be used.

## The Server (Passive Socket) Lifecycle

A server follows a specific sequence of system calls to establish a listening socket capable of accepting client connections.

- **`socket()`**: This call creates an endpoint for communication and returns a file descriptor that references it. This is the initial step for both clients and servers.  
    
- **`bind()`**: This associates the socket file descriptor with a specific network address, which consists of an IP address and a port number. The kernel now knows where to send incoming packets destined for this socket.  
    
- **`listen()`**: This call marks the socket as a passive one, indicating that it will be used to accept incoming connection requests. The `listen()` call also takes a `backlog` parameter, which defines the maximum length of the queue for pending connections.  
    
- **`accept()`**: This call extracts the first connection request from the queue of the listening socket, creates a new connected socket dedicated to that specific client, and returns a new file descriptor for it. The original listening socket remains open and continues to listen for other clients.  
    
- **`read()` / `write()`**: All subsequent communication with the client, such as sending and receiving data, occurs on the new file descriptor returned by `accept()`.  
    
- **`close()`**: When communication is finished, the socket file descriptor is closed.

## The Client (Active Socket) Lifecycle

The client-side setup is more straightforward.

- **`socket()`**: Just like the server, the client first creates a socket endpoint.  
    
- **`connect()`**: This call establishes a connection from the client's socket to the server's listening address. Once this call succeeds, the two-way communication channel is established.

A common best practice for handling connection errors is to completely close the socket file descriptor with `close()` and then create a new one before attempting to `connect()` again. Reusing the same file descriptor in subsequent `connect()` calls after an error can lead to unpredictable behavior across different operating systems. For example, some BSD-derived implementations may fail, so creating a new socket is the most portable approach.
