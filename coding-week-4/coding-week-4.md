# Coding Week 4

Created: July 12, 2021 11:02 PM
Tags: Completed, GSOC, Programming
Timeline: July 1, 2021 → July 7, 2021

One of the most stale and error friendly weeks where I didn't progress much in terms of coding and was loaded with bugs. Also had to make some design decisions about UDP which is entirely difference game from TCP.

## Update Design

I forget about updating the design as it was decided in the last meeting which I did. So `tcpConn` was a `file` now. I added in more details about each of the functions to help out other people looking on design.

Also created a Gist just so in case someone wants to see a rendered Markdown ( can be done in my PR too tho 😛 )

[API Design for socket library](https://gist.github.com/king-11/9e36868660170765688610fdaa12baba)

## Qthreads PR

As Discussed I created PR for Qthreads integration based on conditional compilation.

[Qthread Sys Calls by king-11 · Pull Request #18019 · chapel-lang/chapel](https://github.com/chapel-lang/chapel/pull/18019)

## Sys Additions Bugs, Adds and Review

### Bugs

There was one slight issue with the PR I was not converting host in ipv4 to network order bytes which is done using `htonl` and is required for ipv4 standard addresses like `LOOPBACK` , `ANY`, `BROADCAST`, etc. only when using standard addresses.

This came out while I performed testing.

### Adds

Also their were a few constants which I missed out on adding namely the `setsockopt` ones which I forced push as usual 👉🏻👈🏻 guilty I stand but it makes everything clean right so why not.

### Review

Mike asked Greg to look at the PR he is the runtime expert. The told me that they were planning on making a switch from `Sys` module and making it `POSIX` module with just extern procs and as such no runtime support.

> I'm inclined to think the Chapel-style parts of this interface such as the set() procedures should move into a new module, perhaps called something like SysNetwork, leaving the C-style additions matching the POSIX interface in Sys

I didn't document any of the new functions so that we won't have to go through the additional set of deprecating them when the time comes.

> I'm okay with merging this as-is if there's something like a promise that any Chapel-style parts of the new interface will be moved from Sys to Sockets when the latter is added. (This assumes that Sockets will be added before we embark on the actual Sys module rework, but that seems pretty likely based on comparing recent progress on the two, heh.)

### Merged

After full local testing and reviews the PR was merged

[[GSoC] Socket Constructs Additions Sys Module by king-11 · Pull Request #17932 · chapel-lang/chapel](https://github.com/chapel-lang/chapel/pull/17932)

## Connect Function

Non blocking connects are a pain in code-base. You have to handle so many edges and have to go through so many procedures for such a simple task. I found a good article which I used for the same then.

[](https://www.masterraghu.com/subjects/np/introduction/unix_network_programming_v1.3/ch16lev1sec4.html)

### Internals

If I cite a few of it's procedures they are as follows:

- create a non blocking socket
- call in `qtconnect` using `sockaddr_storage`
    - if it works i.e. returns `0` they we just open the file and return it to user
    - else we call in `qt_select` with `fd` set in read and write sets.
- on timeout or error we handle as supposed
- then we check is `rset` or `wset` has our `fd` set or not
    - if was set we call in `getpeername` to see if it's connected to a remote address. if `getpeername` fails with `ENOTCONN`, the connection failed and we must then call `getsockopt` with `SO_ERROR` ( added in Sys Module Later ) to fetch the pending error for the socket.
    - if socket fd was not set then we return with error
- finally we open the file and return in ... phew.

So I created chapel's connect and got it working with the echo server. Although am inexperienced on Chapel IO so wasn't able to perform much operations.

### QtConnect Issue

While `qt_syscalls` was amazing a hurdle came when then `qt_connect` kept on returning success even when there was no service listening to it on the other end at the port. I had to revert back to normal `connect` because of this.

## UDP Architecture

UDP is a whole new game compared to TCP. I was very much unsure about how would I go about the design. The issues/doubts I had with respect to UDP were:

- Should the functions be blocking or not?
- Can we use chapel IO instead I don't want to get my hands dirty 😓
- How should buffer sizes be configured
- How to handle Generics in Chapel

I told the mentors and Krishna told called me for a meet where we went through Chapel's codebase together to look at ways to make our UDP Architecture.

[recvfrom(3p) - Linux manual page](https://man7.org/linux/man-pages/man3/recvfrom.3p.html)

First we found out that we had `sendto` , `recv` and `recvfrom` in the runtime just need to extern proc them. Then when I told him about buffer issue he told me to use the maximum possible on `recv` and truncate on `send` as per the max. I was reluctant but that seemed like the only solution at moment. We looked at chapel IO's send and recv way too complex for me to understand totally not gonna try. 

### Python to rescue

He also asked me about how python does it? That's when things became easy. Python uses `bytes` for both its `send` and `recv` so user has to encode the data and decode it respectively. And in `recv` they have to specify how many bytes the wanna read. What this solved:

- Generics are handled by bytes now.
- user has to give us buffer size we don't need to go for max possible one.