---
title: "Zig as a Drop in C Replacement"
date: 2025-10-21T09:28:15+03:00
tags: ["zig", "build systems", "zig toolchain", "C"]
categories: ["zig", "build systems", "C"]
keywords: ["zig", "build systems", "zig toolchain"]
draft: false
---

A while back I wrote a small neofetch-like program in Zig as a way to learn the
language. I always knew Zig could interoperate with C, as I had earlier written
a yara rules parser using treesitter and Zig. 

Now I wanted to test my neofetch-like program in various environments, and one 
of those was st. St is a simple terminal emulator, part of the suckless
tools. It is written in C, minimalistic and I generally liked using it.

So, st uses a Makefile for building. In this case, I wanted to see if I could
entirely replace the Makefile with a Zig build file. I'm kinda not used to Makefiles
and it can get complicated super quickly. Why not use Zig's build, something I'm
more familiar with?

So this is how I did it.

The first step was to create a `build.zig` file in the st source directory.
I started by defining our build function with the necessary parameters:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe_module = b.createModule(.{
        .target = target,
        .optimize = optimize,
    });
    const exe = b.addExecutable(.{
	.name = "st",
	.root_module = exe_module,
    });
```

Nothing special so far. Next, I needed to add the source files for st.

```zig
    exe.root_module.addCSourceFiles(.{
        .files = &.{
            "st.c",
            "x.c",
        },
        .flags = &.{
            "-DVERSION=\"0.9.3\"",
            "-D_XOPEN_SOURCE=600",
            "-I/usr/X11R6/include",
            "-L/usr/X11R6/lib",
        },
    });
```

This stage is important as it provides the C source files and the necessary flags
for compilation. The `-DVERSION` flag defines the version of st, while the
`-I` and `-L` specify the include and library paths for X11.

The next part is to bundle them together as below

```zig
    exe.root_module.addCSourceFiles(.{
        .files = &.{},
        .flags = &.{},
    });
```

The next step is to link against the X11 libraries that st depends on.

```zig
    exe.linkLibC();
    exe.linkSystemLibrary("X11");
    exe.linkSystemLibrary("rt");
    exe.linkSystemLibrary("m");
    exe.linkSystemLibrary("util");
    exe.linkSystemLibrary("xft");
    exe.linkSystemLibrary("fontconfig");
    exe.linkSystemLibrary("freetype2");
    exe.addIncludePath(.{ 
        .cwd_relative = "/usr/X11R6/include" 
    });
```

The `.cwd_relative` field is crucial since we are using absolute paths for the
include directories. Zig does complain about it though.

Next, `config.def.h` needs to be copied as it is specified in the Makefile.

```zig
    const config_copy = b.addSystemCommand(&[_][]const u8{ "cp", 
        "config.def.h", "config.h" });
    exe.step.dependOn(&config_copy.step);
```
Finally we can add a step for running the program after building it.

```zig
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(&install_exe.step);
    if (b.args) |args| {
        run_cmd.addArgs(args);
    }
    const run_step = b.step("run", "run the st terminal");
    run_step.dependOn(&run_cmd.step);
}
```

With all these pieces in place, the complete zig build file can be found in this
[Github gist](https://gist.github.com/pop-ecx/1b52d05cf2b38c075b0fbcce9f7157d9).

To build st using Zig, you would run the following command:

```bash
zig build
```
The resulting st binary should be created in the `zig-out/` directory.

Remember that this is a basic example and does not include all the features such
as install and uninstall in the original Makefile. That is left as an exercise 
for the reader. You could try to build other C projects using Zig like Picom for example.

Anyway I kinda like the fact that I can build C projects with ~~Makefile~~, 
~~C compiler~~, just Zig.
