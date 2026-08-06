---
title: "My Favorite Redpill"
date: 2026-08-02T21:21:57+03:00
tags: ["Cybersecurity", "Redpill", "Anti-vm"]
categories: ["Cybersecurity"]
keywords: ["Cybersecurity", "Redpill", "Anti-vm"]
draft: false
---

> No! Not the silly [manospohere](https://en.wikipedia.org/wiki/Manosphere) stuff.

Redpills; techniques in which software (specifically malware) can detect whether
they are running in some sort of virtualized environment, sandbox or debugger.
<!--more-->
It's mostly used by malware authors to make life for us analysts a little
bit harder. The name is an ode to the movie *The Matrix*, and a pretty great one
at that.

There are a few techniques that can be used here including checking for the
presence of certain processes, files, or registry keys that are commonly 
associated with virtual machines or sandboxes. These are relatively uninteresting
to me. Instead I will talk about some of my favorite ones, concluding with probably
my most favorite one.

### Sandbox detection via cpuid
The `cpuid` instruction is an x86 instruction that allows software to determine
the details of the processor. These details may include things like MMX/SSE.

Let's take a look at the following code snippet:

```zig
fn isVm () bool {
    var eax: u32 = 0x40000000;
    var ebx: u32 = 0;
    var ecx: u32 = 0;
    var edx: u32 = 0;

    asm volatile (
        \\cpuid
        : [eax_out] "={eax}" (eax),
          [ebx_out] "={ebx}" (ebx),
          [ecx_out] "={ecx}" (ecx),
          [edx_out] "={edx}" (edx)
        : [eax_in] "{eax}" (@as(u32, 0x40000000)),
          [ecx_in] "{ecx}" (@as(u32, 0))
    );
    std.debug.print("CPUID 0x40000000: eax={x}, ebx={x}, ecx={x}, edx={x}\n", .{eax, ebx, ecx, edx});
    return eax != 0;
}
```
The code is direct. We set eax to  0x40000000 to 
check the maximum hypervisor leaf number and vendor signature. We set the 
remaining registers; ebx, ecx and edx to 0.

If the code runs in a sandbox, it will return vendor signature in the registers,
else they'll just be 0.

Running this in a bare metal environment will return the following:
```powershell
C:\Users\test\>CPUID 0x40000000: eax=0, ebx=o0, ecx=0, edx=0
```

and running it in a sandbox will return something like this:
```powershell
C:\Users\test\>CPUID 0x40000000: eax=0x40000006, ebx=786f4256, ecx=786f4256, edx=786f4256
```

0x40000006 is non-zero and means it supports hypervisor leaves up to 0x40000006.
eax, ebx, ecx and edx show ascii values which decode to VBox (remeber endianness?)

But what if these value can be changed? These cpuid values can be changed at the
VirtualBox configuration level(pretty sure other vendors provide these functionalities too)
So, how do we try to make life harder for the analyst?

### Timing techniques
The idea here is direct; since some instructions have overhead when run in a vm,
we can use timing techniques to determine whether we're in a vm.
The first thing we need to find is a timing source. Luckily the x86 
provides an instruction to do timing; `RDTSC`.

This means we can essentially do something like this:
```c
start = RDTSC
do some work (CPUID + some other instructions) ...
end = RDTSC
if (end - start > threshold)
	hypervisor detected
```

The actual code would look like:

```zig
pub fn main() !void {
    for (0..1000) |_| {
        workload();
    }

    var total: u64 = 0;

    for (0..10000) |_| {
        const start = rdtscStart();
        workload();
        const end = rdtscEnd();
        total += end - start;
    }

    std.debug.print("Average cycles: {}\n", .{total / 10000});
}
```

Where `rdtscStart` is:

```zig
    asm volatile (
        \\lfence
        \\rdtsc
        : [lo] "={eax}" (eax),
          [hi] "={edx}" (edx),
    );
```

and `rdtscEnd`:
```zig
    asm volatile (
        \\rdtscp
        \\lfence
        : [lo] "={eax}" (eax),
          [aux] "={ecx}" (ecx),
          [hi] "={edx}" (edx),
    );
```

and `workload` is:
```zig
    asm volatile (
        \\cpuid
        : [eax] "={eax}" (eax),
          [ebx] "={ebx}" (ebx),
          [ecx] "={ecx}" (ecx),
          [edx] "={edx}" (edx)
        : [leaf] "{eax}" (eax)
    );
```

Running this in bare metal systems gives an average of ~400 cycles, while a vm 
gives ~2700 (at least in my case). These timing differences can be pretty telling
as to whether we're in a vm or not.

### The problem with rdtsc
`rdtsc` can be intercepted and altered when returned causing incorrect readings.

### My favorite redpill
> I learned this technique from one of my favorite [Blackhat talks](https://www.youtube.com/watch?v=_M9SBzSKhsk) by Daniele Cono D'elia

Instead of using rdtsc, we can use a counter thread, making it hard to be trapped
and altered. We can spin up a background thread that just increments an atomic 
counter as fast as possible. Reading that counter acts as a clock. As an addition to this,
we can combine running a computationally "expensive" instruction like cpuid with
a cheap one like `nop` and get the ratios of the two. The idea is that ratios are
harder to fake.

So we'd have something like this:
```zig
pub fn main() !void {
    var clock = CovertClock{};
    try clock.start();
    defer clock.stop();

    const samples: u64 = 10000;

    const rounds = 5;
    var cpuid_total: u64 = 0;
    var nop_total: u64 = 0;
    for (0..rounds) |_| {
        cpuid_total += meanTicks(&clock, cpuidOp, samples / rounds);
        nop_total += meanTicks(&clock, nopOp, samples / rounds);
    }
    const cpuid_avg = cpuid_total / rounds;
    const cpuid_avg_f: f64 = @floatFromInt(cpuid_avg);
    const nop_avg_raw = nop_total / rounds;
    const nop_batch_avg_f: f64 = @floatFromInt(nop_avg_raw);

    const nop_avg = @max(nop_avg_raw / NOP_BATCH, 1);

    const ratio: f64 = (cpuid_avg_f * @as(f64, @floatFromInt(NOP_BATCH))) / nop_batch_avg_f;
    std.debug.print("cpuid: {} ticks (avg)\n", .{cpuid_avg});
    std.debug.print("nop raw (per {}-batch): {}\n", .{ NOP_BATCH, nop_avg_raw });
    std.debug.print("nop:   {} ticks (avg)\n", .{nop_avg});
    std.debug.print("ratio: {d:.2}\n", .{ratio});
}
```

Running this results in a ratio of ~500 on bare metal and ~5000 for vm.

So yeah, my favorite redpill comes from a blackhat talk that I think should get more love.
