---
title: "Zig's New Io"
date: 2026-04-25T18:38:33+03:00
tags: ["Zig", "Io", "Concurrency"]
categories: ["Ziglang"]
keywords: ["Concurrency", "Threads", "Io"]
draft: false
---

![I think I get it now](I-do.jpg)

In version 0.15.1, Zig [removed](https://ziglang.org/download/0.15.1/release-notes.html#async-and-await-keywords-removed)
the `async` and `await` keywords, anticipating to rework and bring them back
as part of a new Io interface. That was pretty exciting, but I didn't get how
it would fully work until I got to try it out in version 0.16.0.
<!--more-->

The context of this blog is set around the workings of a simple port scanner.
I will not go deep into how a port scanner works, but basically it tries to connect
to a list of ports on a target machine, and if it does successfully, it means the
port is open.

Now, we can do this sequentially, or we can do it concurrently. Pretty good excuse
to test the new Io concurrency primitives.

```zig
const std = @import("std");

const Allocator = std.mem.Allocator;
const Io = std.Io;

pub fn main(init: std.process.Init) !void {
    var gpa = std.heap.DebugAllocator(.{}){};
    const allocator = gpa.allocator();

    var io = init.io;

    const host = "192.168.0.102";
    const ports = try parsePorts(allocator, "1-1000");
    defer allocator.free(ports);

    try benchmark(&io, allocator, host, ports);
}
```

Main just sets up allocator and Io using [juicy main](https://ziglang.org/download/0.16.0/release-notes.html#Juicy-Main),
takes host and port range to scan, and then calls the benchmark function.

```zig
fn benchmark(io: *Io, allocator: Allocator, host: []const u8, ports: []const u16) !void {
    const seq_start = std.Io.Clock.now(.real, io.*);

    var seq_results = std.ArrayList(u16).empty;
    defer seq_results.deinit(allocator);

    try scanSequential(io.*, allocator, host, ports, &seq_results);
    
    const seq_end = std.Io.Clock.now(.real, io.*);
    const seq_time = seq_end.toNanoseconds() - seq_start.toNanoseconds();

    const par_start = std.Io.Clock.now(.real, io.*);

    var par_results = std.ArrayList(u16).empty;
    defer par_results.deinit(allocator);

    try scanParallel(io, allocator, host, ports, &par_results);
    
    const par_end = std.Io.Clock.now(.real, io.*);
    const par_time = par_end.toNanoseconds() - par_start.toNanoseconds();

    std.debug.print(
        \\Sequential: {d} ns ({d} open ports)
        \\Parallel:   {d} ns ({d} open ports)
        \\
    , .{
        seq_time, seq_results.items.len,
        par_time, par_results.items.len,
    });
}
```

The benchmark function is pretty straightforward, it just calls the sequential
and parallel scan functions, and measures the time it takes for each of them to
execute, then prints the results.

```zig
fn tcpProbe(io: Io, host: []const u8, port: u16) bool {
    const ip4 = std.Io.net.Ip4Address.parse(host, port) catch return false;
    const addr = std.Io.net.IpAddress{ .ip4 = ip4 };

    const stream = addr.connect(io, .{
        .mode = .stream,
        .protocol = .tcp,
    }) catch return false;

    stream.close(io);
    return true;
}
```

The `tcpProbe` function is the one that actually tries to connect to the target
port, and returns true if it succeeds, and false otherwise.


```zig
fn scanSequential(io: Io, allocator: Allocator, host: []const u8, ports: []const u16, results: *std.ArrayList(u16),) !void {
    for (ports) |port| {
        if (tcpProbe(io, host, port)) {
            try results.append(allocator, port);
        }
    }
}
```

Now we get to our first paradigm, the sequential one. It just iterates over the ports
and calls `tcpProbe` for each of them, and if it returns true, it appends the port
to the results list.

```zig
fn scanParallel(io: *Io, allocator: Allocator, host: []const u8, ports: []const u16, results: *std.ArrayList(u16),) !void {
    var group = Io.Group.init;
    defer group.cancel(io.*);

    const result_slots = try allocator.alloc(?u16, ports.len);
    defer allocator.free(result_slots);
    @memset(result_slots, null);

    for (ports, 0..) |port, i| {
        try group.concurrent(io.*, tcpProbeTask, .{
            io.*,
            host,
            port,
            &result_slots[i],
        });
    }

    try group.await(io.*);

    for (result_slots) |slot| {
        if (slot) |open_port| {
            try results.append(allocator, open_port);
        }
    }
}
```
Now we get to the parallel version, which is a bit more complex. We first initialize
an Io group, which is used to manage the concurrent tasks. Then we pre-allocate a 
list of optional u16 to store the results if each task finds an open port.
We spawn a task for each port, passing it the Io, host, port, and a pointer to
the corresponding result slot. Then we wait for all tasks to complete, and finally we collect the non-null results and append them to the results list.

`tcpProbeTask` is just a task wrapper that calls `tcpProbe` and stores the result in the corresponding result slot.

Comparing the two paradigms, we get this:
```bash
m3lk0r@parrot$ ./main 
Sequential: 10480450421 ns (3 open ports)
Parallel:   1083334239 ns (3 open ports)
```

I modified the code to print the thread id for both execution models, I got this:

```bash
m3lk0r@parrot$ ./main
From sequential: port 1 -> thread 46735
From sequential: port 2 -> thread 46735
From sequential: port 3 -> thread 46735
From sequential: port 4 -> thread 46735
From sequential: port 5 -> thread 46735
From sequential: port 6 -> thread 46735
From sequential: port 7 -> thread 46735
From sequential: port 8 -> thread 46735
From sequential: port 9 -> thread 46735
From sequential: port 10 -> thread 46735
........................................
From concurrent: port 1 -> thread 46748
From concurrent: port 4 -> thread 46751
From concurrent: port 5 -> thread 46748
From concurrent: port 7 -> thread 46753
From concurrent: port 6 -> thread 46752
From concurrent: port 10 -> thread 46756
From concurrent: port 9 -> thread 46754
From concurrent: port 8 -> thread 46751
From concurrent: port 12 -> thread 46757
From concurrent: port 11 -> thread 46749
```
> Notice how the sequential version runs on a single thread, while concurrent version runs on multiple threads.

It's around 8-10x faster, not accounting for network bottlenecks and other factors.

Here is what's interesting though, [function coloring](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
seems to be gone. Let me explain by contrasting it to javascript.

In js, once a function is async, everything that calls it must be async as well,
and everything that calls those functions must be async, all the way up to main.

In zig though, `tcpProbe` is just a normal function that is called by both sequential and
parallel versions, it doesn't need to be async because that is handled by Io. So, there is
no pollution of async-ness, and we can write normal functions that can be called from both paradigms without
worrying whether they are async or not. It is the caller that decides which paradigm
to use.

If Zig was like js, `tcpProbe` would be async, meaning everything else would also
have to be async. Some people argue Zig doesn't really solve this problem, as instead of
async-ness, we have to deal with Io-ness, but regardless, it is a much better design
in my opinion.

It is also noteworthy that the Io chosen here is `Io.Threaded` not `Io.evented`.
This means that each task is running on a separate thread when concurrency is spun, and the performance gain is
coming from the fact that we are able to run multiple tasks in parallel.

It would also be easy to swap to `Io.Evented`, which is the whole point of Zig's design.

