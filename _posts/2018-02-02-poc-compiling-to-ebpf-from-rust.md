---
layout: post
title: 'PoC: compiling to eBPF from Rust'
date: 2018-02-02 21:33:08.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- General
- Rust
tags: []
meta:
author:
  login: geal
  email: geo.couprie@gmail.com
  display_name: Géal
  first_name: ''
  last_name: ''
---
https://twitter.com/gcouprie/status/957332988462882819

I have been playing with <a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/networking/filter.txt" rel="noopener" target="_blank">eBPF (extended Berkeley Packet Filters)</a>, a neat feature present in recent Linux versions (it evolved from the much older BPF filters). It is a virtual machine running in the kernel, to which you can send code from userland, and that code can be used to filter packets or trace parts of the kernel code.

What makes eBPF really nice is how the kernel handles it. You send a program in bytecode format to the kernel, it then checks it, verifying, for example, that there are no loops, thus guaranteeing that the program will terminate, and it will then apply JIT compilation, making the resulting code quite fast. Even better, that code can be loaded and unloaded at any time through a syscall, and you can set up shared data structures between the eBPF program and your own, to efficiently gather data.

As an example, you can use eBPF (and the <a href="https://www.iovisor.org/technology/xdp" rel="noopener" target="_blank">XDP - eXpress Data Path -</a> feature) to write very efficient firewalls, or employ <a href="http://www.brendangregg.com/ebpf.html" rel="noopener" target="_blank">BCC (BPF Compiler Collection)</a> to trace a process's IO events.

I'm looking at how we could use that to trace applications on our infrastructure at <a href="https://www.clever-cloud.com/" rel="noopener" target="_blank">Clever Cloud</a>. There are a few things we should know about the tooling first.

At the beginning, people wrote their program <a href="https://github.com/systemd/systemd/blob/master/src/core/bpf-firewall.c#L89-L129" rel="noopener" target="_blank">using the bytecode directly</a>:

```c
/* Compare IPv4 with one word instruction (32bit)*/
struct bpf_insn insn[] = {
  /* If skb->protocol != ETH_P_IP, skip this whole block. The offset will be set later. */
  BPF_JMP_IMM(BPF_JNE, BPF_REG_7, htobe16(protocol), 0),
  /*
   * Call into BPF_FUNC_skb_load_bytes to load the dst/src IP address
   *
   * R1: Pointer to the skb
   * R2: Data offset
   * R3: Destination buffer on the stack (r10 - 4)
   * R4: Number of bytes to read (4)
   */
  BPF_MOV64_REG(BPF_REG_1, BPF_REG_6),
  BPF_MOV32_IMM(BPF_REG_2, addr_offset),
  BPF_MOV64_REG(BPF_REG_3, BPF_REG_10),
  BPF_ALU64_IMM(BPF_ADD, BPF_REG_3, -addr_size),
  BPF_MOV32_IMM(BPF_REG_4, addr_size),
  BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_skb_load_bytes),
  /*
   * Call into BPF_FUNC_map_lookup_elem to see if the address matches any entry in the
   * LPM trie map. For this to work, the prefixlen field of 'struct bpf_lpm_trie_key'
   * has to be set to the maximum possible value.<
   *
   * On success, the looked up value is stored in R0. For this application, the actual
   * value doesn't matter, however; we just set the bit in @verdict in R8 if we found any
   * matching value.
   */
  BPF_LD_MAP_FD(BPF_REG_1, map_fd),
  BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),
  BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -addr_size - sizeof(uint32_t)),
  BPF_ST_MEM(BPF_W, BPF_REG_2, 0, addr_size * 8),
  BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),
  BPF_JMP_IMM(BPF_JEQ, BPF_REG_0, 0, 1),
  BPF_ALU32_IMM(BPF_OR, BPF_REG_8, verdict),
};
```

This is a bit raw, and somewhat complex to write, so people worked on C to eBPF compilers, and the feature landed in LLVM: we can use clang to write eBPF programs! It will generate the bytecode, that can then be loaded through the <a href="http://man7.org/linux/man-pages/man2/bpf.2.html" rel="noopener" target="_blank">bpf() syscall</a>.

This is still a bit complex, since the eBPF program might need access to some internal data structures of the kernel, and those change depending on kernel versions and configuration options. And we still need to set up the shared data structures with the userland program that will gather data.

That's why the <a href="https://github.com/iovisor/bcc" rel="noopener" target="_blank">BCC project</a> provides an easy to use interface to compile and load eBPF programs. They made it so simple that you can write a python script to compile, load and interact with your program:

```python
from bcc import BPF
BPF(text='int kprobe__sys_clone(void *ctx) { bpf_trace_printk("Hello, World!\n"); return 0; }').trace_print()
```

They provide a lot of <a href="https://github.com/iovisor/bcc/tree/master/examples" rel="noopener" target="_blank">useful examples</a> and <a href="https://github.com/iovisor/bcc/blob/master/docs/tutorial_bcc_python_developer.md" rel="noopener" target="_blank">a nice tutorial</a> to get started writing eBPF tracers.

Unfortunately, those tools make a tradeoff that's slightly annoying for me: they require installing BCC, which requires Python, LLVM and the complete Linux sources, on the target machines. It might be possible to precompile the programs though, but it does not look like it's a common use case with BCC.

So, maybe there's a nice way to precompile those programs, store them as bytecode, then load them with a small agent that does not need LLVM and the kernel sources to work? It turns out it is possible, thanks to the <a href="https://github.com/iovisor/gobpf/" rel="noopener" target="_blank">gobpf project</a>, who <a href="https://github.com/iovisor/gobpf/pull/6/commits/869e637f483f499254d57d443e7aaadad50dce24" rel="noopener" target="_blank">split their ELF loading code</a> from the BCC part a year ago.

And, now, you'll see where I am going with this. Being one of those annoying Rust developers who want to rewrite everything in their favorite language, I thought "hey, maybe I can Rust that thing too!"

Since it is possible to compile to eBPF bytecode from C, it is possible to compile LLVM IR (the kind of bytecode LLVM generates from the code before compiling it to the target CPU's assembly) to eBPF. Look for "LLVM IR debugging" in <a href="https://cilium.readthedocs.io/en/latest/bpf/#llvm" rel="noopener" target="_blank">this link</a> for an example. And I know I can compile Rust to that LLVM IR, and everything should work out, as long as Rust's LLVM version is the same as the system's version.

So I created a small Rust project ([code available here](https://github.com/Geal/rust-ebpf-demo)), and wrote the following build script:

```sh
#!/bin/sh
cargo rustc --release -- --emit=llvm-ir
cp target/release/deps/hello-*.ll hello.ll
cargo rustc --release -- --emit=llvm-bc
cp target/release/deps/hello-*.bc hello.bc
llc-4.0 hello.bc -march=bpf -filetype=obj -o hello.o
```

I generated the ll file to take a look at the IR in text form)

And now, the code!
I used the example program from <a href="https://kinvolk.io/blog/2017/09/an-update-on-gobpf---elf-loading-uprobes-more-program-types/" rel="noopener" target="_blank">this blog post</a> as inspiration, and came up with this:

```rust
use std::mem::transmute;
use std::ffi::CStr;

#[no_mangle]
#[link_section = "license"]
pub static _license: [u8; 4] = [71u8, 80, 76, 0]; //b"GPL\0"

#[no_mangle]
#[link_section = "version"]
pub static _version: u32 = 0xFFFFFFFE;

#[no_mangle]
#[link_section = "kprobe/SyS_clone"]
pub extern "C" fn kprobe__sys_clone(ctx: *mut u8) -> i32 {
  let BPF_FUNC_trace_printk = unsafe {
    transmute::<i32>(6)
  };

let msg: [u8; 17] = [104u8, 101, 108, 108, 111, 32, 102, 114, 111, 109, 32, 114, 117, 115, 116, 10, 0]; //b"hello from Rust\0"
  BPF_FUNC_trace_printk((&msg).as_ptr(), 17);
return 0;
}
```

First, the constants are in their own ELF section, this is expected by gobpf's elf loader. Apparently, I cannot write `pub static _license: &'static [u8] = b"GPL\0"`, because the `_license` symbol would then be a relocation of the actual string.

```rust
#[no_mangle]
#[link_section = "license"]
pub static _license: [u8; 4] = [71u8, 80, 76, 0];//b"GPL\0";

#[no_mangle]
#[link_section = "version"]
pub static _version: u32 = 0xFFFFFFFE;
```

Now, the function: gobpf expects a section with the name of the function we will try to hook later.

```rust
#[no_mangle]
#[link_section = "kprobe/SyS_clone"]
pub extern "C" fn kprobe__sys_clone(ctx: *mut u8) -> i32 {
```

Now we might need to import functions. They are <a href="http://elixir.free-electrons.com/linux/v4.7/source/include/uapi/linux/bpf.h#L153" rel="noopener" target="_blank">defined in a C enum</a>, but interpreted as function pointers. So calling the printk function from there amounts writing the instruction `call 6`.
So we transmute the number `6` into a function. I know, ewww, but it works :D

```rust
let BPF_FUNC_trace_printk = unsafe {
  transmute::<i32>(6)
};
```

And now, the last part, actually printing something. To avoid the previous issue of a constant string appearing as a relocated symbol on which gobpf will throw an error, I defined it as a local constant, then called `BPF_FUNC_trace_printk` on it. That should be harmless, right?

```rust
let msg: [u8; 17] = [104u8, 101, 108, 108, 111, 32, 102, 114, 111, 109, 32, 114, 117, 115, 116, 10, 0]; //b"hello from Rust\0"
BPF_FUNC_trace_printk((&msg).as_ptr(), 17);
```

So now, let's take a look at the generated LLVM IR (the ll file):

```text
; ModuleID = 'hello0-b4990b5a434d0f01306c6e79c17f427.rs'
source_filename = "hello0-b4990b5a434d0f01306c6e79c17f427.rs"
target datalayout = "e-m:e-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"
@_license = local_unnamed_addr constant [4 x i8] c"GPL\00", section "license", align 1
@_version = local_unnamed_addr constant i32 -2, section "version", align 4

; Function Attrs: nounwind uwtable
define i32 @kprobe__sys_clone(i8* nocapture readnone %ctx) unnamed_addr #0 section "kprobe/SyS_clone" {
start:
  %msg = alloca [17 x i8], align 16
  %0 = getelementptr inbounds [17 x i8], [17 x i8]* %msg, i64 0, i64 0
  call void @llvm.lifetime.start(i64 17, i8* nonnull %0)
  %1 = bitcast [17 x i8]* %msg to *
  store  , * %1, align 16
  %2 = getelementptr inbounds [17 x i8], [17 x i8]* %msg, i64 0, i64 16
  store i8 0, i8* %2, align 16
  %3 = call i32 (i8*, i64, ...) inttoptr (i64 6 to i32 (i8*, i64, ...)*)(i8* nonnull %0, i64 17) #2
  call void @llvm.lifetime.end(i64 17, i8* nonnull %0)
  ret i32 0
}

; Function Attrs: argmemonly nounwind
declare void @llvm.lifetime.start(i64, i8* nocapture) #1

; Function Attrs: argmemonly nounwind
declare void @llvm.lifetime.end(i64, i8* nocapture) #1

attributes #0 = { nounwind uwtable "probe-stack"="__rust_probestack" }
attributes #1 = { argmemonly nounwind }
attributes #2 = { nounwind }
```

So, it generates the correct symbols and sections for `_license` and `_version`. It apparently generates a big store instruction for the string we'll print. And the function call looks like this `call i32 (i8*, i64, ...) inttoptr (i64 6 to i32 (i8*, i64, ...)*)(i8* nonnull %0, i64 17)` where we cast 6 to a function pointer: `inttoptr (i64 6 to i32 (i8*, i64, ...)*)`.

So that should generate correct BPF bytecode, right? Let's check that with the command `llvm-objdump-4.0 -S hello.o`:

```text
hello.o:    file format ELF64-BPF

Disassembly of section kprobe/SyS_clone:
kprobe__sys_clone:
       0:   b7 01 00 00 0a 00 00 00     r1 = 10
       1:   73 1a ef ff 00 00 00 00     *(u8 *)(r10 - 17) = r1
       2:   b7 01 00 00 74 00 00 00     r1 = 116
       3:   73 1a ee ff 00 00 00 00     *(u8 *)(r10 - 18) = r1
       4:   b7 01 00 00 73 00 00 00     r1 = 115
       5:   73 1a ed ff 00 00 00 00     *(u8 *)(r10 - 19) = r1
       6:   b7 01 00 00 75 00 00 00     r1 = 117
       7:   73 1a ec ff 00 00 00 00     *(u8 *)(r10 - 20) = r1
       8:   b7 01 00 00 6d 00 00 00     r1 = 109
       9:   73 1a e9 ff 00 00 00 00     *(u8 *)(r10 - 23) = r1
      10:   b7 01 00 00 72 00 00 00     r1 = 114
      11:   73 1a eb ff 00 00 00 00     *(u8 *)(r10 - 21) = r1
      12:   73 1a e7 ff 00 00 00 00     *(u8 *)(r10 - 25) = r1
      13:   b7 01 00 00 66 00 00 00     r1 = 102
      14:   73 1a e6 ff 00 00 00 00     *(u8 *)(r10 - 26) = r1
      15:   b7 01 00 00 20 00 00 00     r1 = 32
      16:   73 1a ea ff 00 00 00 00     *(u8 *)(r10 - 22) = r1
      17:   73 1a e5 ff 00 00 00 00     *(u8 *)(r10 - 27) = r1
      18:   b7 01 00 00 6f 00 00 00     r1 = 111
      19:   73 1a e8 ff 00 00 00 00     *(u8 *)(r10 - 24) = r1
      20:   73 1a e4 ff 00 00 00 00     *(u8 *)(r10 - 28) = r1
      21:   b7 01 00 00 6c 00 00 00     r1 = 108
      22:   73 1a e3 ff 00 00 00 00     *(u8 *)(r10 - 29) = r1
      23:   73 1a e2 ff 00 00 00 00     *(u8 *)(r10 - 30) = r1
      24:   b7 01 00 00 65 00 00 00     r1 = 101
      25:   73 1a e1 ff 00 00 00 00     *(u8 *)(r10 - 31) = r1
      26:   b7 01 00 00 68 00 00 00     r1 = 104
      27:   73 1a e0 ff 00 00 00 00     *(u8 *)(r10 - 32) = r1
      28:   b7 01 00 00 00 00 00 00     r1 = 0
      29:   73 1a f0 ff 00 00 00 00     *(u8 *)(r10 - 16) = r1
      30:   bf a1 00 00 00 00 00 00     r1 = r10
      31:   07 01 00 00 e0 ff ff ff     r1 += -32
      32:   b7 02 00 00 11 00 00 00     r2 = 17
      33:   85 00 00 00 06 00 00 00     call 6
      34:   b7 00 00 00 00 00 00 00     r0 = 0
      35:   95 00 00 00 00 00 00 00     exit
```

So, the lines 0 to 29 are there to load our string (instead of pointing to a constant somewhere). The expected `call 6` instruction is on line 33!

To load that program, we can now use <a href="https://kinvolk.io/blog/2017/09/an-update-on-gobpf---elf-loading-uprobes-more-program-types/" rel="noopener" target="_blank">the example Go program from the blog post</a> (slightly modified to accept a program name as argument): `sudo ./bpf-load hello.o`. The eBPF program will the hook the `sys_clone` function and print a hello every time it is called. You can see the trace with the command `sudo cat /sys/kernel/debug/tracing/trace_pipe`.

And now you can be a happy Rust developer like me because again, you put some Rust where you were not supposed to!
https://twitter.com/gcouprie/status/951526092279607297

Now, this is a small hack. To make it more useful, here is what we would need:

* a small library to import BPF functions instead of transmuting the number every time
* that small library should also have a nice way to interact with BPF maps to transmit data to userland
* a userland library (in Rust, of course) that can set up maps and load eBPF programs. I should mention here that Julia Evans is currently working on a port of gobpf's BCC part in Rust <a href="https://github.com/jvns/ruby-mem-watcher-demo/blob/master/src/bin/bpf.rs" rel="noopener" target="_blank">for a Ruby profiling tool</a>! The ELF part might not be too far :)

That's all, I'll post more once I get more useful code working!
PS/ if you want to learn more about BPF, <a href="https://qmonnet.github.io/whirl-offload/2016/09/01/dive-into-bpf/" rel="noopener" target="_blank">read this great list</a>!
