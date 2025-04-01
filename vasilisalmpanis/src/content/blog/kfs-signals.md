---
title: 'Operating system from scratch: Signals'
description: 'Implementing POSIX signals'
pubDate: 'Mar 31 2025'
heroImage: '/signals.png'
---


#### Introduction
**Signals** are standardized messages sent to a executing program meant to trigger specific behavior

In this article we will be talking about Standard POSIX Signals and not Linux Real-Time signals.
In order to implement signals there are some prerequisites that need to be fullfilled. The kernel
should be  ready to run **userspace** code and also there should be a **syscall** interface for userspace
to be able to communicate with the kernel.

Each signal has a current disposition, which determines how the process behaves when it is delivered
the signal. The default dispositions for a process can be:
- **Term**   Default action is to terminate the process.
- **Ign**   Default action is to ignore the signal.
- **Core**   Default action is to terminate the process and dump core
- **Stop**   Default action is to stop the process.
- **Cont**   Default action is to continue the process if it is
              currently stopped.

Signal handling breaks the single-control-flow structure of a C-like program. In order to undestand what happens lets check at typical signaling scenario.

* Program **x** sends `SIGHUP` to program with pid **y** via syscall.
* The kernel stores the sent signal to **y**'s **PCB**.
* Some time later the kernel will schedule task y. Now it needs to execute the signal handlers before it continues execution of the task.
* After executing the handlers and according to the disposition of the signal the kernel could:
    - terminate the task and optionally `core dump`. 
    - `stop/continue` the task
    - ignore the signal if it is set as ignored.

To implement this we will need to modify the definition of our **PCB**
```zig
const Task = struct {
    // Previous fields
    sigpending: u32,
    action: kSigaction
}
```

There are 31 standard POSIX signals, ranging from 1 to 31 (inclusive) so we will use each bit in this 32 bit
unsigned integer to signify that a signal has be scheduled for this task.We will leave bit 0 unused for now.
This implementation will not queue signals like linux does.  
According to the manual 
***If multiple standard signals are pending for a process, the order
in which the signals are delivered is unspecified.***

**kSigaction** will be the struct that will hold the 31 signal handlers.
The handlers according to the specification are simple function that take
the signal number as argument and return **void**.

```zig
const Signal = enum(u8) {
    EMPTY = 0,
    SIGHUP = 1,
    SIGINT,
    // ...
    SIGUNUSED = 31,
};

const handler = fn (sig: u8) void;

fn stubHandler(sig: u8) void {
    // this is just a stub
    // each handler should implement its own logic.
}

const kSigaction = struct {
    sig_handlers: std.EnumArray(Signal, *const handler) =
            std.EnumArray(Signal, *const handler).init(.{
                .EMPTY      = &handler,
                .SIGHUP     = &handler,
                .SIGINT     = &handler,
                // ...
                .SIGUNUSED  = &handler,
            }),
}
```

Of course sigaction as defined in the C standard library has other members but
the full implementation of it is left to the reader as exercise.

#### Kill

Now of course there are multiple ways that a process could be signaled. For example 
when a process tries to write to a pipe of which the other end (reader) is already closed
it would receive the `SIGPIPE` signal. Or if the user pressed **Ctrl + c** on the keyboard, the
process that is attached to the current **TTY** will receive a `SIGINT` signal. Your kernel could
probably be missing this functionality so we will go for something simpler, Syscalls.

Looking at the Linux syscall table we can see that number **62** is used for the kill syscall.
In case you are not familiar with the **kill** shell command, you can basically send any signal you want
to a process by **PID**. 

![Alt text](/kill.png)

Of course the kernel can accept or reject the signal for multiple reasons.
For our implementation we will create a stub for kill. To keep things simple, we will take 
the pid passed from userspace and try to find the task that it belongs too. If it exists we will set the corresponding
bit in sigpending and return zero for success. Otherwise we will return `-ESRCH` so in the future **Lib C** can set **errno** accordingly.

```zig
pub fn kill(pid: u32, sig: u32) i32 {
    if (sig < 1 or sig > 31)
        return -EPERM;
    const task: ?*Task = getTaskByPid(pid);
    if (task) |obj| {
        const s: u8 = @intFromEnum(sig);
        const mask: u32 = @as(u32, 1) << @as(u5, @truncate(s));
        obj.sigpending |= mask; // Set the bit for this signal
        return 0;               // Success
    }
    return -ESRCH;              // Not found
}
```

Now there could be bugs hiding in this code in case this task is preempted
in between getting the task and setting the bit, and in this case locking
and refcounting would help us solve these problems, but our goal for this
blog post is to get the general idea of how to implement signals and not
present a production ready solution.


#### Scheduling

The scheduling part is perhaps the most complex aspect of signal implementation. 
When a task is about to be scheduled for execution, we need to check if there
are any pending signals that need to be handled before the normal execution flow continues.


Reference:
- **PCB** "Process control block"
- [man signal](https://man7.org/linux/man-pages/man7/signal.7.html)
