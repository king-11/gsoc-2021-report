# Coding Week 3

Created: July 1, 2021 7:23 PM
Tags: Completed, GSOC, Programming
Timeline: June 24, 2021 → June 30, 2021

Probably one of the most fun weeks were I got to learn a lot about chapel's architecture and working. The week as usual starts off with the meeting.

## Qthreads

We have finally figured out how we were to make the Socket Library such that blocking system calls won't block the thread. How ?

The answer lied inside of a third party library that chapel uses `Qthreads` not the `Qt` ones. Qthreads have implemented blocking system calls such that when something like `qt_select` when called they are transferred over to a pthread whose sole purpose is to block while the current thread stays unblocked.

[qt_select(3) manual page](https://cs.sandia.gov/qthreads/man/qt_select.html)

My initial idea was to use `begin_blocking_action` in the #define for `STARTING SLOW SYSTEM CALL`  but Michael suggested instead to use these functions directly and compiling them based on conditional checks.

```c
#ifdef QTHREAD_VERSION
  got = qt_accept(sockfd, (struct sockaddr*) & addr_out->addr, &addr_len);
#else
  got = accept(sockfd, (struct sockaddr*) & addr_out->addr, &addr_len);
#endif
```

This was my first time using `ifdef` finally they make sense to me now thanks to the official documentation on them.

## TCPConn as `file`

Initially I was to create a new record for handling `TCPConn` michael suggested we dont need to do things like as we basically will do IO on it so `file` should suffice with a few additional features. 

I added these additional features using what we call as **tertiary functions** in chapel. The thing which I had to add in for files was how to get their address and the most important thing for a socket its file descriptor.

[File Descriptor Getter for `file` by king-11 · Pull Request #17988 · chapel-lang/chapel](https://github.com/chapel-lang/chapel/pull/17988)

After meeting I was shown my Mike how he codes using Vim it looked like total wizardy to me which I also tried for sometime than back to good ol VSCode. He showed me a similar functions which was fetching something from `qio_file_t` so just had to reuse that which a few checks.

I also updated the Socket Module Design PR to mention TCPConn as a file now.

## Listener Functions

I integrated the Listener Functions which create a `TCPListener` instance for the record itself I had to add in the following:

- An `accept` function which returns the `tcpConn` if successful and waits for new connections internally using `select` call which waits for `read` on socket which translates to that a new connection is pending.
- A `listener` function which creates a socket, binds it and enters into listening mode and then returns the `tcpListener` Instance.
- Getters for `address`, `family` and simple `close` function

## IPAddr Debugging

While we never really talk about `IPAddr` in our meeting, it is one of the things that was polished to its best with good and useful overloads and generic function calls. Internally its based off `sockaddr_storage`. 

I was stuck on a issue with `ipAddr` which mike helped resolve and made functions better too. I was doing several things wrong but he didn't tell me directly he made me battle with `Valgrind` to find it on my own.

- I was trying to get size of a pointer which doesn't provide the size of whole memory it points to so we added in an argument to pass the size.
- I wasn't retrieving the actual filled size of buffer and passing the allocated size to `createStringWithOwnedBuffer` used another `ref` argument for it and `strlen`

While not an issue I wasn't using a generic function and size for retrieving host and port I had those pesky if/else checks so mike suggested a new functions which I hadn't explored.

[GETNAMEINFO flags and returned information examples](https://www.ibm.com/docs/en/zos/2.4.0?topic=reference-getnameinfo-flags-returned-information-examples#getnameinfofe)

When I saw this I was like where were you hiding function. It solved several things for me:

- Buffer can be allocated with new constants that are `NI_MAXHOST` and `NI_NUMERICSERV`
- I dont need to check for type of family it just works.
- code looks neat, clean and is more generic now

```c
int sys_host_sys_sockaddr_t(sys_sockaddr_t* addr, char* host, socklen_t hostlen, int* length){
  err_t err_out = 0;

  err_out = getnameinfo((struct sockaddr *)&addr->addr, addr->len, host, hostlen, NULL, 0, NI_NUMERICHOST);

  *length = strlen(host);
  return err_out;
}
```

## Unit Testing and Docs

Time for testing and due to my issues with `createStringWithOwnedBuffer` I had to change the strings to `c_str` and then compare for equality instead of using `test.assertEqual` which was throwing errors due to wrong allocation.

Eventually I figured it out and added in the test while I also reported to Krishna about issue as he is the creator of `Unit Test` Module

I also added in Documentation for the new functions in Sys Module and the sys_additions PR was finally ready for review.

[[GSoC] Socket Constructs Additions Sys Module by king-11 · Pull Request #17932 · chapel-lang/chapel](https://github.com/chapel-lang/chapel/pull/17932)

## Extras

I created a echo Server using chapel and mentors really liked it. I also felt great it was like if I quote

That's one small step for Socket Library, one giant leap for Chapel.

chapel was finally on web i believe.

One thing Ankush and other mentors discussed was about `QUIC` implementation and can the Socket Library be able to accommodate it in the future. I knew about it but wasn't sure how to implement it at Socket Level.