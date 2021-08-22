# Coding Week 6

Created: July 23, 2021 9:39 PM
Tags: Completed, GSOC, Programming
Timeline: July 15, 2021 → July 21, 2021

## Update Qthreads PR

As was decided in we should remove the overhead caused by `qt_accept` and `qt_connect` as eventually they are supposed to either be called when everything's ready or simply supposed to tell that `EINPROGRESS` respectively.

So I removed them in favor of simple overhead free `accept` and `connect` whereas left `sys_select` with `qt_select` inside. But this was also additional overhead which we want to handle with `select` (blocking) first for a short burst time and then switch over to `qt_select` . I forget about this 😅 and thought my task was done.

Also I had removed the busy wait select with was implemented earlier as wasn't working so we had planned on using blocking `select` for time being.

So mike reminded me that I need to make those change as well which I did. As I also have to test whether they are working or not so I have to test firstly on `socket_library` branch and then copy them over to `qt_threads`.

This is a trouble like real pain in the arse BOLLOCKS 😾. A the thing was happening again `connect` stopped working which I thought had something to do with my implementation of `sys_select` or it was `qt_select` as everything was working if I didn't call `select` prior to `qt_select`.

I planned on asking mentors for help.

## GetAddrinfo

[[GSoC] update addrinfo constructs by king-11 · Pull Request #18072 · chapel-lang/chapel](https://github.com/chapel-lang/chapel/pull/18072)

This was a simple addition as `getaddrinfo` returns the errno that happens if any which isn't the case with other sys_calls we have to check `errno` separately. The changes included using newer interoperability things like suing `c_ptr` as we can't compare a `nil` to opaque but can do it for `c_ptr` which is required when moving through the linked list returned by `getaddrinfo`. Other thing included using `extern "struct addrinfo"` which was earlier just a  record and also updated `freeaddrinfo` which was mistakenly taking a double pointer to `getaddrinfo` instead of one.

This don't end easily do they. So the issue was I wasn't using `getaddrinfo` properly so it was returning error which I was passing to `fromSysError` and it was doing some allocation which was causing it to go out of memory an infinite loop. Paul helped we through this by making notice of issue and mike made a solution PR till the time I was relieved that I wasn't doing out of memory allocations. I just needed to correct by function call.

[SysError.SystemError.fromSyserr() halts on negative arguments · Issue #18103 · chapel-lang/chapel](https://github.com/chapel-lang/chapel/issues/18103)

## Timeout

This was a simple design decision. A negative value for `tv_sec` will signify a `NULL` value for timeout in `select` call which is equivalent to `nil` in chapel so it was just a if else check I had to do at places I was using `select` no need of an overload for the same.

## DNS Resolved Connect

After addition of `getaddrinfo` I was able to create the a DNS resolver connect which can first call getaddrinfo with given `host` and `service` and then call the usual connect after creating an `ipAddr` from the returned `sockaddr_storage`.

## TCP Optimizations

I finally got the hang of two TCP Optimizations `naggle` and `delayAck` . Julia had named one based on optimization while other on the flag it changes:

- Naggle - TCP_NODELAY false
- Delay Ack - TCP_QUICKACK false

So they turn flag to false to enable themselves. I changed the overloads too such that they are only available for `TCP` as there is no use for them in `UDP`