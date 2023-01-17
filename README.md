# Socket Library - Chapel

<p align="center">
<a href="https://summerofcode.withgoogle.com/"><img style="padding: 20px;" alt="drawing" src="GSoC_logo.png" height="200"></a>
<a href="https://www.chapel-lang.org/"><img style="padding: 20px;" alt="drawing" src="chapel.png" height="200"></a>
</p>

## Abstract

[Chapel](https://chapel-lang.org/) is a programming language designed for productive parallel computing at scale.  It simplifies parallel programming through elegant parallel programming constructs. As Chapel can utilize a system to its fullest potential, it seems like one of the ideal solutions to handle millions of network requests.

Currently, we can do network programming by using [C Interoperability](https://chapel-lang.org/docs/language/spec/interoperability.html), but those constructs are not intuitive and good enough for users to easily do their tasks.

## My Project

My Google Summer of Code project with Chapel was focused on creating a standard "Socket Library". This project aimed to ensure concurrent, parallel safe, and easy to use constructs for networking. The project was built on top of C System Calls, saving users from their complexity to perform even the most trivial tasks.

This blog post is for my final submission of the GSoC Project.

- Project: [Socket Library](https://summerofcode.withgoogle.com/projects/#5343790958641152)
- Mentor: [Michael Ferguson](https://github.com/mppf), [Krishna Dey](https://github.com/krishnadey30) and [Ankush Bhardwaj](https://github.com/ankingcodes).
- Organisation: [Chapel](https://github.com/chapel-lang)

Under this project, the following three major parts needed to be managed:

1. Getting review over design for the socket library from other developers.
2. Adding and updating runtime functions that can then be used in Socket Module.
3. Nonblocking but synchronous procedures that allow other tasks to run freely.
4. Creating network constructs for UDP and TCP that align themselves except for the I/O differences.

Pull requests created in the period of GSoC:

| PR Title | Review Status | Merge Status |
|:---:|:---:|:---:|
| [Design: Socket Library](https://github.com/chapel-lang/chapel/pull/17899) | Approved | Draft |
| [Socket Construct Additions Sys Module](https://github.com/chapel-lang/chapel/pull/17932) | Approved | Merged |
| [File Descriptor Getter for `file`](https://github.com/chapel-lang/chapel/pull/17988) | Approved | Merged |
| [Qthread Select](https://github.com/chapel-lang/chapel/pull/18019) | Approved | Merged |
| [Update Addrinfo Constructs](https://github.com/chapel-lang/chapel/pull/18072) | Approved | Merged |
| [Socket Library](https://github.com/chapel-lang/chapel/pull/17960) | Approved | Merged |

> I'll update the status of PRs as they get changed.

## Work Done

### Design

Even though my GSoC [Proposal](https://docs.google.com/document/d/1e95RBpy5eSPHiJUQAJmlPDWjjAaXG0LOm8GnH9J6Lfc/edit?usp=sharing) talked about how the Socket Module should look like its design was entirely based on Python and C System Calls which aren't the best ones and don't align well with how we wanted it to look for Chapel.

My initial task was to look over how other modern-day programming languages implement their Socket Modules. I was recommended to look over at Julia, Go, Python, D, etc. I made did my analysis and brought together a simple [gist](https://gist.github.com/king-11/dc50ba6e3f97b8949a4ec78bbe5cb098) as suggested to me by my mentors.

Finally, after analyzing implementations of other languages and multiple design discussion sessions with Chapel developers, I made a design pull-request describing the API.

<a href="https://github.com/chapel-lang/chapel/pull/17899"><img src="./17899.png"></a>

### Changes to Runtime

The module that is storing most of the extern procs, records, and constants is the [Sys](https://chapel-lang.org/docs/modules/standard/Sys.html) standard module. There were several changes made to the module which were as follows:

- Add in new constants to be used as flags in C System Calls e.g. `F_GETFL`, `SOL_SOCKET` to be used in [fcntl](https://linux.die.net/man/2/fcntl) and [setsockopt](https://www.ibm.com/docs/en/zos/2.3.0?topic=functions-setsockopt-set-options-associated-socket) or [getsockopt](https://man7.org/linux/man-pages/man2/getsockopt.2.html).
- New opaque types `sys_in_addr_t` and `sys_in6_addr_t` which are for standard network addresses like loopback, any, and broadcast.
- Standard Sizes for address and service buffer length to be used in [getnameinfo](https://man7.org/linux/man-pages/man3/getnameinfo.3.html).
- Add in generic getters and setters for `sockaddr_storage` record.

All this was done as a part of the below PR and soon will be moved to Socket Module when it gets added in.

<a href="https://github.com/chapel-lang/chapel/pull/17932"><img src="./17932.png"></a>

To treat socket connections like chapel `file` we needed access to their file descriptor for all the operations for this a minor addition was made in runtime to fetch the file descriptor. This allowed us to use Chapel I/O in the Socket Module which handles buffered I/O and keeps the I/O methods consistent across chapel modules.

<a href="https://github.com/chapel-lang/chapel/pull/17988"><img src="./17988.png"></a>

Finally, to resolve Domain Name Service from user inputs like _"https://summerofcode.withgoogle.com/"_ I made some fixes to `getaddrinfo` and constructs related to it whose old implementation had a few issues:
- Some of the function definitions were not correct which were resulting in segmentation fault and memory leak.
- `addrinfo` wasn't using the new capabilities of chapel runtime and was not interacting with C code it was defined separately.
- As `sys_addrinfo_ptr` was an opaque type we can't compare it with `nil` which was necessary for checking end of linked list returned by `getaddrinfo`.

<a href="https://github.com/chapel-lang/chapel/pull/18072"><img src="./18072.png"></a>

### NonBlocking Procedures

The architecture for the socket library was decided to be synchronous but nonblocking as that's how most chapel procedures work ( async is a possibility for the future ). If they need to block they will yield the main thread so that other tasks of the process can run without being blocked.

For this, I decided to use I/O multiplexing using `select` system calls where the procedure waits until it becomes ready which can mean it becomes readable, writable or a timeout occurred.


![blocking.png](./blocking.png)

The issue was that calling in normal select will cause the pthread on which chapel qthreads are scheduled to block. Hence, to counter this [qthreads library](https://cs.sandia.gov/qthreads/) provides its own set of system calls that will block but not on the current pthread instead are transferred to another helper pthread where they do blocking.

The outcome was making use of `qt_select` inside of our `sys_select` call.

![non_blocking.png](./non_blocking.png)

<a href="https://github.com/chapel-lang/chapel/pull/18019"><img src="./18019.png"></a>

There were certain hurdles and PR remained stale for some time because we faced hurdles due to bugs in error informing inside of qthread's library procedures, which were resolved by collaborating with qthread's core team. The following is the issue that describes the bugs in detail.

<a href="https://github.com/Qthreads/qthreads/issues/88"><img src="./88.png"></a>

### Socket Module

The socket module was designed as stated in the design PR and had a lot of modifications being made to the exposed API as reviews were coming in.

>The main intent was to hide the complexity associated with the C System Calls under custom procedures and records.

The major records in Socket Module that do all the heavy lifting include
- ipAddr
- tcpConnection
- tcpListener
- udpSocket

Along the lines of [Julia](https://docs.julialang.org/en/v1/stdlib/Sockets/#Sockets.IPAddr) utility procedures were provided in the module to user which included `bind`, `naggle`, `delayAck`, `getsockname`, `getpeername`, `setSockopt` and `getSockOpt`.

>The PR has been rebased and changes are being made to it since the finalization of the design for the Module. It consists of all the tests, documentation, and the module itself as well.

Currently, all the tests are passing and PR is under review before a major change to the internal architecture while the APIs exposed to users will remain the same.

<a href="https://github.com/chapel-lang/chapel/pull/17960"><img src="./17960.png"></a>

### Libevent Integration

[Libevent](https://libevent.org/) exposes APIs that are more performant and handle more open file descriptors or connections than `select` call ever could. The reason being that libevent uses kernel methods that have a time complexity of `O(1)` unlike select which is `O(N)`.

Libevent works using callbacks but we have to ensure to make it work in a non-blocking and synchronous manner. Integrating libevent was something that was never done in the chapel so it was a completely new experience for me.

The architecture can be best explained with diagrams, so here goes.

![connect with event loop.png](./event_connect.png)

The events are handled by the event loop which runs a callback whenever an event ( read/ write/ timeout ) occurs on the file. It writes back the event's value to a sync variable (passed as an argument to the callback function) transitioning to full state and the procedure waiting on the sync variable is finally woken up.

![event loop.png](./event_loop.png)

## Usage of Socket Module

A TCP Client in Chapel is as follows:
```chapel
use Socket;
use Time;

proc main() {
  config const host:string = "chapel-lang.org", port:uint(16) = 80;
  const x = connect(host, port);
  const writer = x.writer();
  writer.writeln("hello world!");
  writer.close();
  x.close();
}
```

To serve a basic TCP Server using Socket Module will look as follows:
```chapel
use Socket;

proc main() {
  var address = ipAddr.create("127.0.0.1", 8000, IPFamily.IPv4);
  var server = listen(address, backlog = 6000);
  writeln("Listening for connections on ", server.addr.host, ":", server.addr.port);
  defer server.close();

  while (true) {
    var conn = server.accept();
    const address = getpeername(conn);
    const writer = conn.writer();
    writer.writeln(address, " connected ");
    writer.close();
    conn.close();
  }
}
````

We can utilize chapel `begin` to demonstrate UDP connections in a single program. `begin` creates a new task or user-level thread the terminology might differ but it has a separate execution context from the main task/thread providing us capabilities for task parallelization.

```chapel
use Socket;

proc main() throws {
  var address = ipAddr.ipv4(IPv4Localhost, 8111);
  var receiver = new udpSocket();
  bind(receiver, address);

  var sender = new udpSocket();

  var sent = b"hello world!";
  begin {
    var sentBytes = sender.send(sent, address);
  }
  var received:bytes;
  received = receiver.recv(sent.size);
  writeln(received);
}
```

## Weekly Updates

>Updates for all the weekly meetings that took place can be found on this [notion site](https://curse-okapi-b2c.notion.site/ef73f6e5c0b840b1a2e0b1ec977e0c24?v=e62ff72d91f84d728d9594714e228efd).

## Future Work

- Integrating `libevent` with the chapel channels through an I/O plugin.
- Running performance tests to analyze the network calls and I/O latency.
- Supporting opaque types with, the [LLVM](https://llvm.org/) backend.
- Providing users with libevent as a dependency if they don't have it on their system.

## Learning

- The major learning outcome from this project was good coding practice. I learned how to write DRY, reusable code following SOLID principles. I also learned how to write good tests/specs covering all possible scenarios.
- When writing the Design for the project, I had long discussions with my mentors trying to decide on exposed APIs. Coming up with good names for methods, variables, and hiding the complexity from end-users.
- Through this journey, I learned a lot about how open source projects work, getting involved with the community, and contributing to the projects.
- The major tech takeaway for me was how familiar I got in working with C System Calls, Pointer Exceptions, and Chapel itself.
- I am still exploring Asynchronous I/O through [libevent](https://libevent.org/doc/event_8h.html) for the project which is something that powers fast servers like NodeJS, Julia, and Go.

## Special Thanks

It was wonderful working with the Chapel community throughout the GSoC project. I would like to thank Michael, Krishna, and Ankush for solving my queries, debugging issues with me, giving constructive suggestions and guiding me through the program. Thanks to Brad, Lydia, Engin, Greg, Paul, and the members of the Chapel Community for helping with their suggestions.

I would like to thank [Shivansh Saini](https://github.com/shivanshs9), @[Nishant Mittal](@nishantwrp), and @[Ashish Kumar](@krashish8) for motivating and guiding me in applying and getting through the whole program.

Final thanks to Google for organizing this amazing program. I feel GSoC made it easier for me to get started with open source contributions. It was a great lesson on how projects are built and maintained with great quality. I am excited to continue contributing to the open-source community

Thanks for reading :) and please share your thoughts. I always like to meet more enthusiastic people, so do reach out to me on [Twitter](https://twitter.com/1108King) or [Linkedin](https://www.linkedin.com/in/lakshyasingh11/) to share your feedback or for any queries. I'd be more than happy to help and connect.
