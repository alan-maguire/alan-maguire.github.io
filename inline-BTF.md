# Adding inlined functions to the BPF Type Format

## Introduction

One of the common pain points for kernel tracing is when we attempt
to attach tracing to a function and attachment fails.  This failure
can happen for a number of reasons, and once we have understood these
we will be in a better position to see the value of having inline
function information.

## The problem of missing functions

Here we discuss some of the reasons you may not be able to trace
a function.

### 1. "notrace"/"noinstr" functions

#### Problem

The function itself is marked "notrace" or "noinstr" ; this is the case
for functions that are themselves invoked as part of the tracing
infrastructure.  As you can imagine, tracing a function which
then fires again because you are tracing, leading to a further
firing and so on is a big problem.

#### Diagnosis

A function is in `/proc/kallsyms` but not in `/sys/kernel/tracing/available_filter_functions`.

#### Solution

Instead of relying on `/proc/kallsyms` to find traceable functions,
use `/sys/kernel/tracing/available_filter_functions`, or even
better `/sys/kernel/tracing/available_filter_functions_addrs` which
has both function address, name and optional module:

```
$ sudo cat cat /sys/kernel/tracing/available_filter_functions_addrs
ffffffff84000014 trace_initcall_level_cb
ffffffff84000044 trace_initcall_start_cb
ffffffff84000100 run_init_process
ffffffff84000210 try_to_run_init_process
```

It is still not possible to trace these functions.

### 2. Optimized functions with changed function signatures

#### Problem

To attach BPF tracing using fentry/fexit, it is vital that the BPF
Type Format (BTF) information contains the function signature
as a `BTF_KIND_FUNC/FUNC_PROTO`; without this the verifier cannot
safely attach to the function.  For optimized functions which omit
parameters or have their parameters in unexpected locations rather
than the expected registers from the ABI, BTF is often omitted.

#### Diagnosis

Determine if the function made it into BTF:
```
$ bpftool btf dump file /sys/kernel/btf/vmlinux |grep myfunc
...
```

If the kernel was built with gcc, optimized functions will have
a `.isra` or `.constprop` suffix, for example `crypt_iv_tcw_whitening.isra.0`.

#### Solution

Instead of fentry/fexit, use kprobe tracing instead, and be aware
that the parameters may not be in the registers you expect.

### 3. Inlined functions

#### Problem

Compilers make all sorts of choices to optimize the objects they
generate.  One big win is inlining, where a function - potentially
called in many different places - has its body placed into the
body of the caller(s).  This saves on function call overhead and
inlining is extensive in modern environments.

To give a sense of this, consider the numbers for just the kernel
vmlinux image itself.

On a representative distro kernel, I see around 60,000 un-inlined and
available to trace functions, while there are about 100,000 functions
that were inlined to about 600,000 sites.

That is we see 10x inline sites vs available-to-trace functions.

You can see this by examining the kernel source; the "inline" hint
is used extensively (especially in headers containing small functions)
and `__always_inline` - which tries to ensure the function is inlined
rather than hinting to the compiler to do so - is used frequently too.

#### Diagnosis

The function is not in `/proc/kallsyms`.

#### Solution

None, aside from changing the code to avoid inlining.  Tricks include
making a function non-static, though even this breaks with a Link-Time
Optimized (LTO) kernel.

## Some good news

In fact it is possible to trace at an inline site using a kprobe.
kprobes can trace at most instructions (unlike fentry/fexit), but the
problem is that the only place we have information about inlining is
in the DWARF debuginfo which is hundreds of megabytes and often not
available on a system like BTF is.  BTF is found in /sys/kernel/btf
and is baked into the kernel image rather than being a separate
installable package like DWARF debuginfo.

So if we had a way to compactly represent the 600,000 inline sites
we could trace them.  However with 600,000 of anything the representation
needs to be as compact as possible - how to proceed?

## Steps towards a solution


## Inline support in libbpf, pahole and the kernel

## Using inline information to trace inlined functions

## Wrapping up


