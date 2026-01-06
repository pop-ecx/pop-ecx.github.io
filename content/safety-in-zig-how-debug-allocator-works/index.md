---
title: "Safety in Zig: How Debug Allocator Works"
date: 2026-01-06T10:55:00+03:00
tags: ["zig", "allocators", "memory-safety"]
categories: ["zig", "allocators", "memory-safety"]
keywords: ["zig", "allocators", "memory-safety"]
draft: false
---

Zig is not a memory-safe language. It, however, provides some tools to help avoid
some memory safety issues. One of the key tools that help achieve this is the debug allocator.
The debug allocator is a special memory allocator that tracks memory allocations and deallocations,
helping identify memory leaks and other memory-related issues during development.

Since zig isn't a GC language and does not have a borrow checker, partly due to its design goals,
the debug allocator plays a crucial role in helping avoid some memory safety issues.

#### What the debug allocator checks

1. **Memory Leaks**: The debug allocator detects memory leaks with stack traces, making it easier to identify where memory was allocated but not freed.
2. **Double Free**: Double frees are also detected, and prints all three traces (allocation, first free, second free).

#### How Debug allocator detects memory leaks
Before we dive into how zig does this, let's first get a TL;DR of Zig's detection strategy.

The debug allocator splits memory into small and large allocations.Small ones
are rounded up to the next power-of-two size class and packed into fixed-size pages (“buckets”),
where each page tracks used slots with a simple bitset and hands out the next free
slot fast and easy. When a bucket fills up, it links to a new one, forming a list
that’s handy for leak checks.
Frees find their metadata by math and alignment tricks. Resizing only works if 
you stay in the same size class.
Big allocations skip all this and go straight to the backing allocator, with their metadata tracked separately in a hash map.

Now that that's out of the way, let's see how zig implements leak detection.

> The following code snippet is taken from the debug allocator implementation in Zig's standard library (0.15.x).

```zig
pub fn detectLeaks(self: *Self) bool {
    var leaks = false;

    for (self.buckets, 0..) |init_optional_bucket, size_class_index| {
        var optional_bucket = init_optional_bucket;
        const slot_count = slot_counts[size_class_index];
        const used_bits_count = usedBitsCount(slot_count);
        while (optional_bucket) |bucket| {
            leaks = detectLeaksInBucket(bucket, size_class_index, used_bits_count) or leaks;
            optional_bucket = bucket.prev;
        }
    }

    var it = self.large_allocations.valueIterator();
    while (it.next()) |large_alloc| {
        if (config.retain_metadata and large_alloc.freed) continue;
        const stack_trace = large_alloc.getStackTrace(.alloc);
        log.err("memory address 0x{x} leaked: {f}", .{
            @intFromPtr(large_alloc.bytes.ptr), stack_trace,
        });
        leaks = true;
    }
    return leaks;
}
```

In this code snippet, the `detectLeaks` function iterates though all the small 
allocation buckets. It walks through each bucket and calls `detectLeaksInBucket` to
check which memory slots were allocated but not freed. The `detectLeaksInBucket`
reports the specific memory address that leaked along with its stack trace.

It then moves on to large allocations, doing the same check. If any leaks are found,
it logs the memory address and stack trace. After all is done, it returns whether
there were leaks found or not.

> Cursory reading of the same code in master branch (future 0.16.x) shows improvements
> such as reporting the number of leaks instead of just a boolean. The core logic however
> remains the same.

Let's put this to the test with a simple example;

```zig
const std = @import("std");

pub fn main() void {
    var debug_alloc = std.heap.DebugAllocator(.{ .verbose_log = true, .safety = true }){};
    const allocator = debug_alloc.allocator();

    defer {
        const result = debug_alloc.deinit();
        switch (result) {
            .ok => {},
            .leak => std.debug.print("Memory leak detected!\n", .{}),
        }
    }

    const leak = allocator.alloc(u8, 16) catch return;
    leak[0] = 99;
    std.debug.print("Leaked buffer first byte: {}\n", .{leak[0]});
}
```
> The debug allocator takes a struct config parameter, where you can set options like safety, stack trace frames etc.
> You can check the full list of options in [the std lib documentation](https://ziglang.org/documentation/0.15.1/std/#std.heap.debug_allocator.Config).

In this example, we are deliberately leaking memory by not freeing an allocated buffer.
When this is run, the leak will be detected and reported.

Running the code brings the following output:

```bash
m3lk0r@parrot:~$ zig build run
anyzig: .minimum_zig_version '0.15.1' pulled from '/home/m3lk0r/allocdemo/build.zig.zon'
anyzig: appdata '/home/m3lk0r/.local/share/anyzig'
anyzig: zig '0.15.1' already exists at '/home/m3lk0r/.cache/zig/p/N-V-__8AAN5NhBR0oTsvnwjPdeNiiDLtEsfXRHd1fv-R3TOv'
info(gpa): small alloc 16 bytes at 0x7f859a260000
Byte: 99
error(gpa): memory address 0x7f859a260000 leaked: 
/home/m3lk0r/allocdemo/src/main.zig:15:33: 0x113e8a7 in main (main.zig)
    const leak = allocator.alloc(u8, 16) catch return;
                                ^
/home/m3lk0r/.cache/zig/p/N-V-__8AAN5NhBR0oTsvnwjPdeNiiDLtEsfXRHd1fv-R3TOv/lib/std/start.zig:618:22: 0x113d9dd in posixCallMainAndExit (std.zig)
            root.main();
                     ^
/home/m3lk0r/.cache/zig/p/N-V-__8AAN5NhBR0oTsvnwjPdeNiiDLtEsfXRHd1fv-R3TOv/lib/std/start.zig:232:5: 0x113d271 in _start (std.zig)
    asm volatile (switch (native_arch) {
    ^

Memory leak detected!
```

#### How debug allocator detects double frees
This is the snippet from Zig's source code:

```zig
fn reportDoubleFree(ret_addr: usize, alloc_stack_trace: StackTrace, free_stack_trace: StackTrace) void {
    var addresses: [stack_n]usize = @splat(0);
    var second_free_stack_trace: StackTrace = .{
        .instruction_addresses = &addresses,
        .index = 0,
    };
    std.debug.captureStackTrace(ret_addr, &second_free_stack_trace);
    log.err("Double free detected. Allocation: {f} First free: {f} Second free: {f}", .{
        alloc_stack_trace, free_stack_trace, second_free_stack_trace,
    });
}
```
This function basically captures the stack traces for the allocation, first free, and second free events.
It then logs an error message that includes all three stack traces.

> Master branch (future 0.16.x) shows a similar function with some improvements
> such as using `@BranchHint(.cold)` to express that this code path is unlikely to be executed
> so optimizations to other branches can be prioritized,
> and some better logging formatting. The core logic remains the same.

To see this in action, let's modify our previous example to include a double free:

```zig
pub fn main() void {
    var debug_alloc = std.heap.DebugAllocator(.{ .verbose_log = true, .safety = true }){};
    const allocator = debug_alloc.allocator();

    defer {
        const result = debug_alloc.deinit();
        switch (result) {
            .ok => {},
            .leak => std.debug.print("Memory leak detected!\n", .{}),
        }
    }

    const leak = allocator.alloc(u8, 16) catch return;
    leak[0] = 99;
    std.debug.print("Byte: {}\n", .{leak[0]});
    
    //Note: We are freeing to patch the memory leak from previous example
    defer allocator.free(leak);

    const df = allocator.alloc(u8, 16) catch return;
    df[0] = 42;
    std.debug.print("byte: {}\n", .{df[0]});
    //Now we do a double free
    allocator.free(df);
    allocator.free(df);
}
```

When you run this code, a double free will be detected and reported:
```bash
m3lk0r@parrot:~$ zig build run
anyzig: .minimum_zig_version '0.15.1' pulled from '/home/m3lk0r/allocdemo/build.zig.zon'
anyzig: appdata '/home/m3lk0r/.local/share/anyzig'
anyzig: zig '0.15.1' already exists at '/home/m3lk0r/.cache/zig/p/N-V-__8AAN5NhBR0oTsvnwjPdeNiiDLtEsfXRHd1fv-R3TOv'
info(gpa): small alloc 16 bytes at 0x7f8a52ea0000
Byte: 99
info(gpa): small alloc 16 bytes at 0x7f8a52ea0010
Double free buffer first byte: 42
info(gpa): small free 16 bytes at u8@7f8a52ea0010
error(gpa): Double free detected. Allocation: 
/home/m3lk0r/allocdemo/src/main.zig:20:31: 0x113eb15 in main (main.zig)
    const df = allocator.alloc(u8, 16) catch return;
                              ^
/home/m3lk0r/.cache/zig/p/N-V-__8AAN5NhBR0oTsvnwjPdeNiiDLtEsfXRHd1fv-R3TOv/lib/std/start.zig:618:22: 0x113d9dd in posixCallMainAndExit (std.zig)
            root.main();
                     ^
/home/m3lk0r/.cache/zig/p/N-V-__8AAN5NhBR0oTsvnwjPdeNiiDLtEsfXRHd1fv-R3TOv/lib/std/start.zig:232:5: 0x113d271 in _start (std.zig)
    asm volatile (switch (native_arch) {
    ^
 First free: 
/home/m3lk0r/allocdemo/src/main.zig:24:19: 0x113edef in main (main.zig)
    allocator.free(df); // first free 
                  ^
/home/m3lk0r/.cache/zig/p/N-V-__8AAN5NhBR0oTsvnwjPdeNiiDLtEsfXRHd1fv-R3TOv/lib/std/start.zig:618:22: 0x113d9dd in posixCallMainAndExit (std.zig)
            root.main();
                     ^
/home/m3lk0r/.cache/zig/p/N-V-__8AAN5NhBR0oTsvnwjPdeNiiDLtEsfXRHd1fv-R3TOv/lib/std/start.zig:232:5: 0x113d271 in _start (std.zig)
    asm volatile (switch (native_arch) {
    ^
 Second free: 
/home/m3lk0r/allocdemo/src/main.zig:25:19: 0x113ee58 in main (main.zig)
    allocator.free(df); // second free
                  ^
/home/m3lk0r/.cache/zig/p/N-V-__8AAN5NhBR0oTsvnwjPdeNiiDLtEsfXRHd1fv-R3TOv/lib/std/start.zig:618:22: 0x113d9dd in posixCallMainAndExit (std.zig)
            root.main();
                     ^
/home/m3lk0r/.cache/zig/p/N-V-__8AAN5NhBR0oTsvnwjPdeNiiDLtEsfXRHd1fv-R3TOv/lib/std/start.zig:232:5: 0x113d271 in _start (std.zig)
    asm volatile (switch (native_arch) {
    ^

info(gpa): small free 16 bytes at u8@7f8a52ea0000
```

The output is a bit long, but that's the point; detailed stacktraces for all three events are provided.

### Conclusion
The debug allocator in zig is a powerful tool for ensuring you dodge some common memory pitfalls.
It is however noteworthy that the allocator is meant to be used only in development. Using it in
production can lead to performance costs due to its overhead. Once you
are confident your code is solid, switch to a more efficient allocator for production builds.

There are other classes of memory issues and safety in general that the debug allocator does not cover.
Some are covered by release mode safety checks, while others can be caught by some plugins/tools.
Those will be covered in future blogs 🤞.
