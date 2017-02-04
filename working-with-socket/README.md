### Buffering

- 一次该读（写）多少数据？
- 是多次进行少量数据的读取还是一次性读取大量数据？

#### Write buffer.

What happens when write data on a tcp socket ?

> Buffering allows calls to write to return almost immediately.
> Then, behind the scenes, the kernel can collect all the pending writes,
> group them and optimize when they're sent for maximum performance to avoid flooding the network.
> At the network level, sending many small packets incurs a lot overhead,
> so the kernel batches small writes together into larger ones.

#### Read buffer.

> When calling `read`, Ruby may actually be able to receive more data than your limit allows.

So buffering the pending data.

16kb is a reasonable minimum length.


### Socket Options

- Socket#setsockopt -> setsockopt(2)
- Socket#getsockopt -> getsockopt(2)


reuse_addr.


### Non-blocking IO

- read_nonblock
- write_nonblock
- connect_nonblock
- accept_nonblock



> Given more than one process trying to accept a connection on different copies of the same socket,
> the kernel balances the load and ensures that one, and only one, copy of the socket will be able to accept any particular connection.

### Unix Process

- pid
- ppid
