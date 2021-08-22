# Coding Week 5

Created: July 15, 2021 9:22 AM
Tags: Completed, GSOC, Programming
Timeline: July 8, 2021 → July 14, 2021

The Mid Term Evaluations are going this week thankfully two of my PRs got merged after I told Mike i was waiting on his review for them.

## Qthreads Isssue

Integrating `qt_connect` didn't seem possible so I was asked to create a `C` reproduction for the success I was getting where I expected an error.

### Installing Qthreads

While the `INSTALLATION` document was pretty brief and clear about installing qthreads. It certainly didn't cover how should I use it in C programs after installing.

[https://github.com/Qthreads/qthreads](https://github.com/Qthreads/qthreads)

So asked about this on Slack as well as Chapel Gitter. Paul helped me out in compiling the program finally and stackoverflow too. But then the global install was segfaulting which the qthreads contributor told me required a `qthread_initialize()` call when I created the issue with reproducible code.

Michael Suggested me to use the qthreads install inside [chapel/third-party](https://github.com/chapel-lang/chapel/tree/main/third-party/qthread). But that was giving a whole new error stream on runtime. So we gave up and created the iss

### Creating the issue in qthreads

[qt_connect sucess when expected to fail · Issue #88 · Qthreads/qthreads](https://github.com/Qthreads/qthreads/issues/88)

## File FD Merged

Not so quick

I again introduced noise in Chapel testing 😓 this time it was because it declared some functions with a void type in runtime whereas where `extern`ing them I used a `c_int` return. What I introduced earlier was more worse though 😂 i basically broke down everything I belive.

[Fix long path error to only check length of base filename by bradcray · Pull Request #17698 · chapel-lang/chapel](https://github.com/chapel-lang/chapel/pull/17698)

Although current one just needed a quick fix, I was online when Mike told me about this, so I added a commit into this PR only 😅 he was hoping for a new PR but he told it works too. So after final review he merged the PR 🥳

[File Descriptor Getter for `file` by king-11 · Pull Request #17988 · chapel-lang/chapel](https://github.com/chapel-lang/chapel/pull/17988)

## UDP IO

UDP IO became far more simpler after we decided upon that user has to specify number of bytes they wanna receive and using bytes as  our generic data type for sending and receiving. I needed to extern proc three functions `recv` , `recvfrom` and `sendto`.

### Receive Data

This has two procedures one for getting the address along with data received and one that just returns the data.

```css
extern proc sys_recvfrom(sockfd:fd_t, buffer:c_void_ptr, len:size_t, flags:c_int, ref address:sys_sockaddr_t,  ref recvd_out:ssize_t):err_t;
```

The task is simple all I have to do was ask user how much bytes they wanna read and how much time should we wait ( can be infinite too ). Internally I first call in a select with socket in reading set. On signal we call in receive and then return the bytes. timeout and other errors are thrown along the way.

```css
proc udpSocket.recvfrom(buffer_len:int, in timeout = new timeval(0,0), flags:c_int = 0) throws {
    var rset: fd_set;
    sys_fd_zero(rset);
    sys_fd_set(this.socketFd, rset);
    var nready:int(32);
    var err_out:err_t;

    err_out = sys_select(socketFd + 1, c_ptrTo(rset), nil, nil, c_ptrTo(timeout), nready);

    if nready == 0 {
      throw SystemError.fromSyserr(ETIMEDOUT, "recv timed out");
    }
    if err_out != 0 {
      throw SystemError.fromSyserr(err_out);
    }

    var buffer = c_calloc(c_uchar, buffer_len);
    var length:c_int;
    var addressStorage = new sys_sockaddr_t();
    err_out = sys_recvfrom(this.socketFd, buffer, buffer_len, 0, addressStorage, length);
    if err_out != 0 {
      throw SystemError.fromSyserr(err_out);
    }

    return (createBytesWithOwnedBuffer(buffer, length, buffer_len), new ipAddr(addressStorage));
  }
```

### Send Data

It was more simpler but buggy at first. So we can in destination for the data and the data. Internally we have to call in `sendto` which need a `void` pointer. Initially mike told be convert the bytes into a pointer and then use it directly in the call.

That wasn't working the data size I was sending was alright but at the receive end things were buggy where I wasn't able to see the data sent. 

So I changed the `data` to `c_str` and then created a pointer for `c_str` to use in the `sendto` call. This was still sending it the correct number of bytes but data was malformed. Huh ... so I went to **Ankush** asking for help he told me to use buffers.

Buffers ankush initial used the older documentation and the buffer module has very less support as of now. He wanted to use `buffers` so that we can handle sending in large data streams too. As we were talking I found the actual issue.

So you see `c_str` is actually a pointer kids so don't forget `C` i was sending in pointer to a pointer obviously it wouldn't work. So got it done and finally in meeting we talked about since buffers don't have a good support we should drop the idea.