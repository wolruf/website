+++
title = "eBPF, part 2: Deep Dive [DRAFT]"
date = "2017-04-26"
tags = ["eBPF", "BPF", "Networking", "Linux"] 
draft = true

summary = '''
TODO
'''
+++

# LWT NOTES

The goal is to allow for stateful and stateless tunnel/encap to be performed without requiring every single state to be represented with a net_device

There are two variations, MPLS without a net_device at all and VXLAN with a master device to derive config but the majority of state being configured through metdata, either in a route/dst or via BPF/OVS

-Thomas Graf

# eBPF, part 2

This article is the second in my series on eBPF.
This one builds on the previous, ["eBPF, part 1: Past, Present, and Future"]({{< ref "ebpf_past_present_future.md" >}}), by diving into the depths of eBPF's current implementation.

TODO


# Intro

The difficulty of explaining and using eBPF stems from how different it is from anything the general methods of network operations.

It is a new, fundamentally different abstraction of networking.
It is built into the Linux kernel and subsequently approaches networking the way one might imagine a kernel developer would.


The path to modern day eBPF is not a simple one.
**TODO: add more here.**


# Walking through the details of eBPF's implementation

A great summary of eBPF, when it was initially released, and its extensions over BPF can be found in the release notes for version 3.18 of the Linux kernel.[[^linux_3.18_notes]]

[^linux_3.18_notes]: Linux 3.18 release notes related to eBPF: [https://kernelnewbies.org/Linux_3.18#head-ead251efb6bbdbe2922e7c6bd0c7b46342e03dad](https://kernelnewbies.org/Linux_3.18#head-ead251efb6bbdbe2922e7c6bd0c7b46342e03dad)

> bpf() syscall is a multiplexor for a range of different operations on eBPF which can be characterized as "universal in-kernel virtual machine". eBPF is similar to original Berkeley Packet Filter used to filter network packets. eBPF "extends" classic BPF in multiple ways including ability to call in-kernel helper functions and access shared data structures like eBPF maps. The programs can be written in a restricted C that is compiled into eBPF bytecode and executed on the eBPF virtual machine or JITed into native instruction set.
>
>eBPF programs are similar to kernel modules. They are loaded by the user process and automatically unloaded when process exits. Each eBPF program is a safe run-to-completion set of instructions. eBPF verifier statically determines that the program terminates and is safe to execute. The programs are attached to different events. These events can be packets, tracepoint events and other types in the future. Beyond storing data the programs may call into in-kernel helper functions which may, for example, dump stack, do trace_printk or other forms of live kernel debugging. 

This is a relatively dense explanation of eBPF so it's worth walking through.
Since this explanation encapsulates most all of eBPF, getting a base level understanding of it will make understanding uses of eBPF relatively easy to follow.

## The Linux bpf syscall

The Linux `bpf` syscall, not to be confused with the FreeBSD syscall of the same name, does have [a man page](http://man7.org/linux/man-pages/man2/bpf.2.html).
Sadly, however, it's vastly out of date for the time being.
As of writing, building the most recent commit for the Linux man pages still has dated and missing information on the `bpf` syscall.
From the git logs it appears that the last notable contribution of new information to the man page was on **July 30, 2015**, over a year and eight months ago.[[^last_bpf_man_addition]]

[^last_bpf_man_addition]: This addition was done by Daniel Borkmann in commit `9a818dddcffce642126c4d8389ad679554617fc4` to the Linux [man-pages repository](https://git.kernel.org/pub/scm/docs/man-pages/man-pages.git/).

The information from the man page does provide a decent idea of the basic functionality of BPF.
The kernel code can be compared with this to gleam added features which haven't been documented yet.
For reference, this article will use the code of the most recent Linux kernel release, version 4.10.
As of 4.10 there are four files directly related to the operation of eBPF.
Each path listed below refers to a file in the kernel repository and links to the file it references.


* [`kernel/bpf/syscall.c`](https://github.com/torvalds/linux/blob/v4.10/kernel/bpf/syscall.c):
The Linux kernel code related to the `bpf` syscall.
* [`include/uapi/linux/bpf.h`](https://github.com/torvalds/linux/blob/v4.10/include/uapi/linux/bpf.h):
The `bpf` header file for assisting in using the `bpf` syscall.
* [`net/core/filter.c`](https://github.com/torvalds/linux/blob/v4.10/net/core/filter.c):
The eBPF kernel side code that runs the virtual machine.
* [`net/core/sock.c`](https://github.com/torvalds/linux/blob/v4.10/net/core/sock.c):
Includes the `setsockopt` code used to manipulate eBPF filters.


## eBPF commands

To begin with let's look at the ten commands for the `bpf` Linux syscall.
Of those ten, six are listed in the man page: `BPF_PROG_LOAD`, `BPF_MAP_CREATE`, `BPF_MAP_LOOKUP_ELEM`, `BPF_MAP_UPDATE_ELEM`, `BPF_MAP_DELETE_ELEM`, and `BPF_MAP_GET_NEXT_KEY`.
Of these commands there are really only two types of actions the Linux `bpf` syscall can do: loading an eBPF program and various manipulations of maps in the eBPF system.

A quick look at the `bpf_enum` in `include/uapi/linux/bpf.h` ([link here](https://github.com/torvalds/linux/blob/v4.10/include/uapi/linux/bpf.h#L67)) shows the four other commands: `BPF_OBJ_PIN`, `BPF_OBJ_GET`, `BPF_PROG_ATTACH`, `BPF_PROG_DETACH`.
All together this gives us the following 10 commands.

```c
// from https://github.com/torvalds/linux/blob/v4.10/include/uapi/linux/bpf.h#L67
enum bpf_cmd {
	BPF_MAP_CREATE,
	BPF_MAP_LOOKUP_ELEM,
	BPF_MAP_UPDATE_ELEM,
	BPF_MAP_DELETE_ELEM,
	BPF_MAP_GET_NEXT_KEY,
	BPF_PROG_LOAD,
	BPF_OBJ_PIN,
	BPF_OBJ_GET,
	BPF_PROG_ATTACH,
	BPF_PROG_DETACH,
};
```

These calls for the basics of what it possible 
To give some context to these, we'll quickly walk through the different commands for the `bpf` Linux syscall.

To begin with, `BPF_PROG_LOAD` is used for the loading of an eBPF program and is straightforward.
It requires the type of eBPF program (more on that in a [below]???), the array of eBPF virtual machine instructions, the license associated with the filter, a log buffer for messages from validating the filter code, and the log level for those messages.
To see an example of how this is done in C refer to the [man page](http://man7.org/linux/man-pages/man2/bpf.2.html) mentioned above.

Next, there are the five map operations.
Here we encounter another term which has been outgrown by its evolution.
As of writing eBPF has ten types of "maps", which will be expanded upon in the ??? section.
For the moment simply know that eBPF offers different data structures that are either hash maps or arrays.
A number of specialized variants of these exist for more specialized circumstances.
This article will use the term "eBPF-map" to refer to the all of these different data structures eBPF offers for storing state.

The eBPF-map operations are fairly self descriptive and are used to create eBPF-maps, lookup an element from them, update an element, delete an element, and iterate through an eBPF-map (`BPF_MAP_GET_NEXT_KEY`).
Creating a map only requires the type of eBPF-map desired, the size of the keys, the size of the values, and the maximum number of entries.

After this we have the currently undocumented commands related to pinning objects (`BPF_OBJ_PIN`), getting objects (`BPF_OBJ_GET`), attaching programs (`BPF_PROG_ATTACH`), and detaching programs (`BPF_PROG_DETACH`).


Tracing through the history in the repository, one can find that `BPF_OBJ_PIN` and `BPF_OBJ_GET` were added in  version 4.4 of the Linux kernel.[[^ebpf_pin_commit]]
The release notes for version 4.4 state that two eBPF features were added: "unprivileged eBPF" and "persistent eBPF programs".[[^linux_4.4_release_notes]]
The `BPF_OBJ_PIN` command relates to the persistent eBPF programs feature of version 4.4 of the Linux kernel.
With this the Linux kernel features a new minimum file system located at `/sys/fs/bpf` which can hold eBPF-maps or eBPF programs.
This is useful before, prior to this, all eBPF-maps and eBPF programs were tied to the program that created them.
Thus it wasn't possible to have a tool create an eBPF program and exit because that would cause the filter to be destroyed.
This addition allows for eBPF-maps and eBPF programs to persist after the program that creates them exits.
Because a file system is used for this, eBPF-maps and eBPF programs are pinned to a path in the file system.
This command thus only needs the file descriptor of what to pin and the path to pin it at.

[^ebpf_pin_commit]: They were contributed by Daniel Borkmann and David S. Miller with commit [`	b2197755b2633e164a439682fb05a9b5ea48f706`](https://github.com/torvalds/linux/commit/b2197755b2633e164a439682fb05a9b5ea48f706).

[^linux_4.4_release_notes]: Linux kernel 4.4 release notes: [https://kernelnewbies.org/Linux_4.4#head-20c20e63018e8fb916fd26476eda2512e2d96632](https://kernelnewbies.org/Linux_4.4#head-20c20e63018e8fb916fd26476eda2512e2d96632)

Finally, we have the last two commands `BPF_PROG_ATTACH` and `BPF_PROG_DETACH`.
These commands were actually just added with version 4.10 of the Linux kernel in February of 2017, only two months ago, though they appear to have been written in November of 2016.[[^ebpf_attach_detach_commit]]
The version 4.10 release notes explain that this is used for attaching eBPF programs to cgroups.[[^linux_4.10_release_notes]]
For those not familiar with cgroups, they're a Linux kernel feature used on processes for resource limiting and isolation.
The primary use case for eBPF with cgroups is that filters can be used to accept or drop traffic either to or from processes of a cgroup.

It is worth noting that the eBPF program type `BPF_PROG_TYPE_CGROUP_SOCK` also exists which appears to allow control over which device `AF_INET` and `AF_INET6` sockets are attached to for processes in the designated cgroup.
This is evidence of the possible future of eBPF programs in the Linux kernel, a future where eBPF program are used for injection of non-trivial logic by users into kernel functionality.


[^ebpf_attach_detach_commit]: This was contributed by Daniel Mack with David S. Miller on November 23, 2016 with commit [f4324551489e8781d838f941b7aee4208e52e8bf](https://github.com/torvalds/linux/commit/f4324551489e8781d838f941b7aee4208e52e8bf).
[^linux_4.10_release_notes]: Linux 4.10 release notes: [https://kernelnewbies.org/Linux_4.10](https://kernelnewbies.org/Linux_4.10)


## eBPF program types

There are currently 12 types of eBPF programs, which is a substantial number.
Of these twelve, four are listed on the man page and only one has any real documentation.
Also, before proceeding, it's helpful to clarify that the enumeration of all eBPF program types contains thirteen options because the first one is `BPF_PROG_TYPE_UNSPEC`.
This option for the program type is invalid to use and exists as the first option to ensure that a zero, in its many possible forms with C code, doesn't accidentally slip in for the program type.

```C
// From https://github.com/torvalds/linux/blob/v4.10/include/uapi/linux/bpf.h#L94

enum bpf_prog_type {
	BPF_PROG_TYPE_UNSPEC,
	BPF_PROG_TYPE_SOCKET_FILTER,
	BPF_PROG_TYPE_KPROBE,
	BPF_PROG_TYPE_SCHED_CLS,
	BPF_PROG_TYPE_SCHED_ACT,
	BPF_PROG_TYPE_TRACEPOINT,
	BPF_PROG_TYPE_XDP,
	BPF_PROG_TYPE_PERF_EVENT,
	BPF_PROG_TYPE_CGROUP_SKB,
	BPF_PROG_TYPE_CGROUP_SOCK,
	BPF_PROG_TYPE_LWT_IN,
	BPF_PROG_TYPE_LWT_OUT,
	BPF_PROG_TYPE_LWT_XMIT,
};
```

### Socket filter

The first valid option, `BPF_PROG_TYPE_SOCKET_FILTER`, is the easiest to understand as it is used for programs that do what BPF was originally designed to do: filter packets for user-space processing.
These programs return either `-1`, to indicate the packet matches the filter, or `0`, to indicate the packet does not match the filter.
It is important to remember that there are use cases where a user isn't interested in the raw packets but aggregate information on them.
For such situations a `BPF_PROG_TYPE_SOCKET_FILTER` program can be used that always returns `0` but aggregates state into a eBPF-map which the user can retrieve at their discretion.

### Kprobe

Next there is the `BPF_PROG_TYPE_KPROBE` eBPF program type.
To make sense of this, one first must understand what kprobes are.
Kprobes are another feature of the Linux kernel and were first introduced into version 2.6.11 of the kernel by IBM in March 2005.[[^original_kprobe_commit]]
Using kprobes is a fairly technical process, but their end result is they allow for inspection of the kernel by tying custom instructions to a kernel instruction for debug and tracing purposes.
This allows for inspection of the kernels guts on anything from opening a socket to flushing a disk queue.
Tools like SystemTap are built on top of kprobes and allow for shell scripts to debug kernel internal processes.[[^systemtap_reference]]
For those interested in an example using these tools in the real world, ScyllaDB has an excellent blog post by Glauber Costa titled "Big latencies? It’s open season for kernel bug-hunting!".
It involves using SystemTap with other tools to debug a bizarre latency when using ScyllaDB.
The blog post can be found [at this link here](http://www.scylladb.com/2016/11/14/cfq-kernel-bug/).

[^original_kprobe_commit]: The original commit appears to come from Ananth N Mavinakayanahalli of IBM and the message from it can be found in the kernel 2.6.11 change log: [https://www.kernel.org/pub/linux/kernel/v2.6/ChangeLog-2.6.11](https://www.kernel.org/pub/linux/kernel/v2.6/ChangeLog-2.6.11)

[^systemtap_reference]: Like kprobes, SystemTap was created by members of IBM as well. Source: [https://www.ibm.com/support/knowledgecenter/en/linuxonibm/liaai.systemTap/liaaisystapover.htm](https://www.ibm.com/support/knowledgecenter/en/linuxonibm/liaai.systemTap/liaaisystapover.htm)

In eBPF, the `BPF_PROG_TYPE_KPROBE` program type is used to attach an eBPF function to a kernel event so that it is run every time the specific system event occurs.
An number of examples using eBPF kprobe programs can be found in the BCC toolchain by IO Visor.
This article will cover the BCC tool chain in more depth later on, but for the moment know that it allows for easy use of eBPF from Python and Lua.
In the code for the `fileslower` tool, written in a mere 248 lines and [viewable on Github](https://github.com/iovisor/bcc/blob/78948e4aae6aa0d06806d452d193320936d59dc7/tools/fileslower.py), multiple kprobes are used to monitor calls to `__vfs_read()` and `__vfs_write()`.
An example of using this tool has been reposted from [BCC's documentation](https://github.com/iovisor/bcc/blob/78948e4aae6aa0d06806d452d193320936d59dc7/tools/fileslower_example.txt) and shows the power of eBPF for low level debugging.

```
$ ./fileslower 
Tracing sync read/writes slower than 10 ms
TIME(s)  COMM           PID    D BYTES   LAT(ms) FILENAME
 0.000   randread.pl    4762   R 8192      12.70 data1
 8.850   randread.pl    4762   R 8192      11.26 data1
12.852   randread.pl    4762   R 8192      10.43 data1
```

### Traffic Control Classifiers and Actions

The `BPF_PROG_TYPE_SCHED_CLS` and `BPF_PROG_TYPE_SCHED_ACT` program types are for Linux traffic control (often abbreviated "tc") classifiers and actions respectively.
Linux traffic control encompasses the system by which Linux processes network traffic into and out of the kernel.
Traffic control is often used to for quality of service solutions, a basic example of which could be preferring latency sensitive traffic.
The Linux traffic control system is highly capable and, perhaps as a result of this, complex.
This article will not delve into the depths of Linux traffic control as an entire weekend could be spent simply reading on the subject.

*For the curious and the thorough (with a weekend to kill), the most complete references I've found on the Linux traffic control system is ["Linux Advanced Routing & Traffic Control HOWTO" (sometimes called "LARTC") by Bert Hubert with many additional section authors](http://lartc.org/lartc.html).[[^LARTC_section_authors]]*

[^LARTC_section_authors]: Section authors, listed in same order as in "Linux Advanced Routing & Traffic Control HOWTO", are Thomas Graf, Gregory Maxwell, Remco van Mook, Martijn van Oosterhout, Paul B Schroeder, Jasper Spaans, and Pedro Larroy.

A massively condensed and simplified description of Linux traffic control is that traffic coming in on interfaces and sent out on interfaces must pass through traffic control which processes the traffic through a series of queues.
These queues, their qualities, composition, overall affect on the traffic, and the tools used to interact with them is a large past of the complexity of Linux traffic control.
However, the two concepts from traffic control which concern eBPF are classifiers and actions.

Classifiers are used to match data frames that flow through traffic control and thus are sometimes referred to as "filters".[[^data_frames_vs_packets]]
Actions are used to do things upon a filter match including mangling and redirecting traffic.
This is highly powerful as it can be combined with eBPF-maps to allow traffic to be proxied in a highly efficient manner for whatever uses one can imagine.
Information on the usage of eBPF classifiers and actions can be found in more detail on [traffic controls man page dedicated to eBPF](http://man7.org/linux/man-pages/man8/tc-bpf.8.html).

[^data_frames_vs_packets]: Frames are the correct term for units of data sent in layer two of the OSI network model. However, some writings incorrectly refer to frames as "packets" even though packets are the units of data at layer three.

*The history of Linux traffic control is not one that appears well documented and thus exists in varying, sparse, and often dated pieces.
For those who read through other documentation, the term "policers" is often used instead of actions.
As best I could find at the time of this writing, policers where replaced by actions, but this is not something I could find definitive information on.
This article welcomes feedback and corrections on Linux traffic control.
Such responses can be submitted, with evidence, as issues to the [Github repository for this website](https://github.com/code-ape/website).*


### Tracepoints

The Linux kernel also features tracepoints which can invoke an eBPF program if they are loaded as the `BPF_PROG_TYPE_TRACEPOINT` program type.
Tracepoints are specific points in the Linux kernel which allow for code to be executed when code fires through the point, including visibility of the arguments.
In this way they're very similar to kprobes except tracepoints are generally used for static tracing.
When the kernel is compiled there are certain places which are coded as tracepoints so that probes can easily be attached in a reliable and safe way.
Kprobes, conversely, can be attached to any point in the system call chain.
Both have their uses and with static tracing being simpler, more stable, and more limited and dynamic tracing being more complex, less stable, and more reaching in its tracing ability.

The ability to attach eBPF programs to Linux tracepoints was first introduced in Linux kernel version 4.7 which was released in July of 2016.
The release notes for version 4.7 of the kernel include a [short summary on the addition of eBPF usage for tracepoints](https://kernelnewbies.org/Linux_4.7#head-33cca324387de62dcbbdc7b6a320df2bf4cdcf62) and also has a link to an [article by Jonathan Corbet on the matter](https://lwn.net/Articles/683504/).

On of the most knowledgeable individuals on tracing is [Brendan Gregg](http://www.brendangregg.com), who works at Netflix as of writing.
Gregg has authored [perf-tools](https://github.com/brendangregg/perf-tools) and written extensively on tracing with it, including [using eBPF tracepoint programs](http://www.brendangregg.com/perf.html#eBPF).
Other reference to using eBPF tracepoint programs can be found in BCC's examples.
This includes a [program that traces KVM calls](https://github.com/iovisor/bcc/blob/bd8370e8980c6457bef45cb20ab4752ccad679a5/examples/tracing/kvm_hypercall.py) and a [program that traces calls to `urandomread`](https://github.com/iovisor/bcc/blob/0c8c179fc1283600887efa46fe428022efc4151b/examples/tracing/urandomread.py)(which was contributed by Gregg).


### XDP 

Express Data Path, abbreviated as XDP, is a recent effort to allow eBPF processing of packets as low down in the network stack as possible and thus as fast and efficient as possible.
It was introduced in Linux kernel version 4.8, released in October 2016, and is thus only five months old as of writing.
There doesn't appear to be a well documented history on XDP and thus understanding its intended purpose can be a bit confusing.
Some [presentations on XDP](https://people.netfilter.org/hawk/presentations/xdp2016/xdp_intro_and_use_cases_sep2016.pdf) say that it was created to compete with the Data Plane Development Kit (DPDK).

Whether intentional or not, XDP is designed to achieve a similar result to DPDK and thus it's worth explaining what DPDK is.
DPDK is a Linux Foundation Project designed to do extremely efficient networking from user-space by bypassing the kernel and talking directly to the Network Interface Controller (abbreviated as NIC, this is generally the physical network card).
This has allowed use of Linux for high speed and high throughput network devices.
One example of the capabilities of DPDK is CloudRouter which, as of version 3.0, is able achieve a max throughput of 650Gbps when run on beefy hardware (validity of that number for real world applications is questionable as the benchmark details aren't included).[[^cloud_router_benchmark]]
It is also worth noting that the scope of uses for DPDK are larger than those of XDP.
DPDK can be used in non-networking service cases, such as ScyllaDB, to achieve high throughput and low latency for services.[[^scylladb_dpdk]]

[^cloud_router_benchmark]: Post from CloudRouter's blog on the release and benchmark: [https://cloudrouter.org/cloudrouter/2016/03/29/cloudrouter-3.0-released.html](https://cloudrouter.org/cloudrouter/2016/03/29/cloudrouter-3.0-released.html)

[^scylladb_dpdk]: Benchmarks from ScyllaDB using HTTPD: [https://github.com/scylladb/seastar/wiki/HTTPD-benchmark](https://github.com/scylladb/seastar/wiki/HTTPD-benchmark)


XDP takes a similar approach to DPDK by also processing data directly from the NIC.
However, instead of processing it with a user-space program, XDP uses eBPF programs to process the raw packets.
The main benefits of XDP, as listed in the [initial release notes for it](https://kernelnewbies.org/Linux_4.8#head-fd53c4b82c689a2639ff3092603be428213f8770), are:

1. **XDP is designed for high performance.** It uses known techniques and applies selective constraints to achieve performance goals.
1. **XDP is also designed for programmability.** New functionality can be implemented on the fly without needing kernel modification.
1. **XDP is not kernel bypass.** It is an integrated fast path in the kernel stack.
1. **XDP does not replace the TCP/IP stack.** It augments the stack and works in concert.
1. **XDP does not require any specialized hardware.** It espouses the less is more principle for networking hardware.

Currently XDP allows for one of three actions on the frames it processes: pass it up to the network stack, drop it so that it never makes it to the network stack, or bounce it back to the NIC for transmission onto the network.
XDP can also edit the content of frames, thus allowing it to act as a proxy to either entities on the machine or else where on the network.
As you may have guessed, using XDP to edit frames is why bouncing them back out on the NIC makes any sense.
Currently this has limited usability as there is no way to take a frame received on one interface and send it out another.
However, some use cases do fit inside these constraints such as [using XDP for a decentralized load balancer solution](http://prototype-kernel.readthedocs.io/en/latest/networking/XDP/use-cases/xdp_use_case_load_balancer.html).

The most common use case of XDP, as of writing, is for dropping traffic in an extremely rapid and efficient manner.
This is a critical need for mitigating denial of service attacks (DOS attacks) and XDP has been benchmarked at dropping 20 million packets a second which equates to approximately 28Gbps of traffic.

Finally, it's important to note that XDP can not be used everywhere.
Because XDP sits so low in the network stack the network driver must have XDP implemented.
Currently this includes Mellano's xmlx4 and mlx5 driver, Netronome's nfp driver, QLogic's / Cavium's qed drivers, and the virtio_net driver.
It also appears that Broadcom's bnxt_en driver will be [supported in 4.11](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=c6d30e8391b85e00eb544e6cf047ee0160ee9938).
Finally, work has been done for Intel's [e1000](https://git.kernel.org/pub/scm/linux/kernel/git/ast/bpf.git/commit/?h=xdp&id=0afee87cfc800bf3317f4dc8847e6f36539b820c), [e1000e](https://lists.iovisor.org/pipermail/iovisor-dev/2017-April/000705.html), and [i40e](https://www.spinics.net/lists/netdev/msg409498.html), and [a generic driver](https://www.spinics.net/lists/xdp-newbies/msg00054.html).


[Tom Herbert "We're, in some sense, building a hammer. We'll say how it's used."](https://youtu.be/lpJk_HcCLnQ?t=1m06s)


Hardware offload: https://netdevconf.org/1.2/session.html?jakub-kicinski

https://jvns.ca/blog/2017/04/07/xdp-bpf-tutorial/


### Perf events

Linux has yet another benchmarking system for measuring hardware and it's referred to, by the somewhat generic name of, "performance monitoring".
The eBPF program type `BPF_PROG_TYPE_PERF_EVENT` allows for eBPF to be run as a handler to data from the performance monitoring system.
Performance monitoring works, in short, by either sampling one out of a configurable number of a targeted event or sampling on a set time interval.
Performance monitoring can be used to monitor things like CPU cache miss ratio, page faults, and mis-predicted branch instructions.

A short writeup on using eBPF for performance monitoring including why eBPF is an improvement over old approaches can be found on, perhaps unsurprisingly, [on Brendan Gregg's blog](http://www.brendangregg.com/blog/2016-10-21/linux-efficient-profiler.html).
For something more code oriented and simply in intentions, the [BCC repository has a tool called `llcstat`](https://github.com/iovisor/bcc/blob/0a34d1e6f83a1f25952ab308b2df60d7a2a5447b/tools/llcstat.py) which samples CPU cache references and misses.
And finally, for a more detailed look into performance monitoring, the [man page for `perf_event_open`](http://man7.org/linux/man-pages/man2/perf_event_open.2.html) details all the possible ways that they can be used.



### Cgroups

As mentioned in the [eBPF Commands section](#ebpf-commands) of this article, eBPF can be used to both filter network traffic in and out of processes from a cgroup and allow control over which device sockets are attached to when opened from a cgroup.
The program types `BPF_PROG_TYPE_CGROUP_SKB` and `BPF_PROG_TYPE_CGROUP_SOCK` are used for program that do this traffic filtering and device assignment respectively.

As mentioned previously, these were both added in the most recent version of the Linux kernel, 4.10, only two months ago as of writing.
Thus documentation on them is scarce, the only notable piece of it at the moment is an [article written by Jonathan Corbet in August of 2016](https://lwn.net/Articles/698073/) which talks about the efforts to determine the best solution for filtering network traffic of cgroups.


### Light-Weight Tunnels

This leave the final eBPF program types of `BPF_PROG_TYPE_LWT_IN`, `BPF_PROG_TYPE_LWT_OUT`, and `BPF_PROG_TYPE_LWT_XMIT`.
The acronym "LWT" stands for light-weight tunnels and is a building block for uses like Multi-Protocol Label Switching and Identifier Locator Addressing, or MPLS and ILA for short.
For brevity, and because it is currently the more widely used of the two, this article will only give a brief explanation of MPLS.

MPLS is used to achieve fast switching across large networks by attaching labels to frames instead and switching on them instead of addresses in the frame.
Because MPLS is indifferent to the protocol it encapsulated (hence the "MP" part of MPLS) it has a wide variety of uses.
However, according to some sources "The two most popular implementations of MPLS are layer 3 BGP/MPLS-VPNs (based on RFC 2547) and Layer 2 (or pseudowire) VPNs".[[^mpls_source]]
It's important to not the unusual fact that MPLS can be used for layer 2 and layer 3 solutions.
This is because MPLS doesn't fit well into the standard ISO networking model and as such has been used for encapsulation at both levels. 

[^mpls_source]: Quote from Johna Till Johnson of Network World: [http://www.networkworld.com/article/2297171/network-security/network-security-mpls-explained.html](http://www.networkworld.com/article/2297171/network-security/network-security-mpls-explained.html)

While there are, sadly, no examples of using eBPF for MPLS at the moment, one can imagine the benefits of being able to do the label attachment and switching using an eBPF program.
Security policies and real time network optimization are just a few of the possibilities.

There are two examples of using eBPF with light-weight tunnels.
They are both located in the [`samples/bpf/` directory of the Linux kernel](https://github.com/torvalds/linux/blob/v4.10/samples/bpf/).
One, [`lwt_len_hist.sh`](https://github.com/torvalds/linux/blob/v4.10/samples/bpf/lwt_len_hist.sh) appears to track information of the frames coming through the tunnel.
The other, [`test_tunnel_bpf.sh`](https://github.com/torvalds/linux/blob/v4.10/samples/bpf/test_tunnel_bpf.sh) is a test script that does GRE, VXLAN, GENEVE, and IPIP tunnels with eBPF as part of the tunnel.


# eBPF tool-chains

Currently there are a few tool chains to choose from.

These include

## Linux bindings

Because eBPF is part of the kernel it must be exposed by it.
However, since these are the low level user-space bindings to the kernel, they must be built up with lots of other code before they can serve any use.

*For those who are interested in seeing some examples of the low level usage of the kernel headers, check out Linus Torvalds' example directory [on Github](https://github.com/torvalds/linux/tree/master/samples/bpf).*


## BCC by IO Visor / Plumgrid

The first version of BCC were released in June of 2015 by the Plumgrid team before Plumgrid was acquired by VMWare in December 2016.[[^plumgrid_acquisition]]
It is now branded under IO Visor, fully known as the IO Visor Project, which is a Linux Foundation Collaborative Project and has VMWare as [one of its two platinum members](https://www.iovisor.org/members).


If you venture to [IO Visor's "About" page](https://www.iovisor.org/about) you'll notice that the technology is looks to develop is basically eBPF:

[^plumgrid_acquisition]: Source: [https://www.sdxcentral.com/articles/news/vmware-acquires-employees-assets-plumgrid/2016/12/](https://www.sdxcentral.com/articles/news/vmware-acquires-employees-assets-plumgrid/2016/12/)


> The IO Visor Project is an open source project and a community of developers to accelerate the innovation, development, and sharing of virtualized in-kernel IO services for tracing, analytics, monitoring, security and networking functions.


Indeed, as of writing this article the IO Visor lists three technologies: eBPF, XDP, and it's BCC tool-chain to use them.

The result is this is that BCC, an acronym for BPF Compiler Collection, allows uses to conveniently load C code of eBPF programs, insert them into the kernel, and the get data from them.

## bpftools by CloudFlare

# Projects using eBPF

There are a few projects currently making use of eBPF.

## Cilium

# Further reading

Refer to "eBPF: Past, Present, Future"

# Appendix

## The search for LWT in Linux

*For those looking to get more information on light-weight tunnels in Linux, here is the path I went down and what I was able to find on them.
Hopefully there will be more documentation in the future, but for the time being I hope this will make others search for information on light-weight tunnels easier.*

The non-obvious meanings of "LWT" isn't surprising.
The eBPF program types for LWT were just added in the most recent version of the Linux kernel, 4.10, only two months ago as of writing.
More surprisingly, however, is any information about them dead ends rapidly.
The release notes for Linux 4.10 have all of five words to say about the matter, "BPF for lightweight tunnel encapsulation", followed by links to two commits which where authored by Thomas Graff and committed by David S. Miller.
The first of these commits offers a short description for the new eBPF program types with no real context and second just adds tests for the additions from the first.

The commit message of the new eBPF code, written by Graff, states that these three eBPF program types correspond to the "LWT hooks" of `dst_input()`, `dst_output()`, and `lwtunnel_xmit()`.
It goes on to say that all eBPF "programs receive an skb with L3 headers attached" and that they may either choose to propagate the frame up the routing chain or drop it and return an `EPERM` error (no explanation for what on earth an "operation not permitted" error means in this context).
Finally, eBPF `BPF_PROG_TYPE_LWT_XMIT` programs can "modify packet content as well as prepending an L2 header via a newly introduced helper bpf_skb_change_head()".

While Graff's commit message does explain how the new eBPF program types work by relating them the light-weight tunnels, it does nothing to help us understand what light-weight tunnels are.
Searching for articles or writings on "LWT" or "light-weight tunnels" in Linux yields essentially nothing.
The only thing I was able to find after a half hour of searching was a single presentation by Roopa Prabhu of Cumulus Networks from February 2016 on MPLS with Linux.
This in turn uses the phrase `lwtunnel` for referring to light-weight tunnels and upon searching the Linux kernel I finally got a hit in the `net/core/` directory from [a file called `lwtunnel.c`](https://github.com/torvalds/linux/blob/v4.10/net/core/lwtunnel.c).

Tracing down the original commit for the `lwtunnel.c` file in Linux yields a [commit by Roopa Prabhu and David S. Miller from July of 2015](https://github.com/torvalds/linux/commit/499a24256862714539e902c0499b67da2bb3ab72).
The commit message says it provides infrastructure to parse, dump, and store encapsulation information for light-weight tunnels like MPLS.
Based on the fact that Prabhu has a presentation on MPLS with Linux from February 2016, it appears that light-weight tunnels have been successfully used for MPLS.
Searching in the Linux kernel for MPLS does indeed turn up an [entire directory under networking for it, located at `net/mpls`](https://github.com/torvalds/linux/tree/v4.10/net/mpls).
And finally, after all of this, we find that `lwtunnel` is used by [`net/mpls/mpls_iptunnel.c`](https://github.com/torvalds/linux/blob/v4.10/net/mpls/mpls_iptunnel.c).

For reference, there are a few article on doing MPLS with Linux including a [short tutorial from 2015 by Sam Russell](http://pieknywidok.blogspot.com.co/2015/12/mpls-testbed-on-ubuntu-linux-with.html).

There are also a few other uses of `lwtunnel` in the Linux kernel including for Identifier Locator Addressing (ILA) in [`net/ipv6/ila`](https://github.com/torvalds/linux/tree/v4.10/net/ipv6/ila) ([here's an article explaining ILA](https://lwn.net/Articles/657012/)),
IP Tunneling in [`net/ipv4/ip_tunnel_core.c`](https://github.com/torvalds/linux/blob/v4.10/net/ipv4/ip_tunnel_core.c), and in the [`net/ipv4/fib_semantics.c`](https://github.com/torvalds/linux/blob/v4.10/net/ipv4/fib_semantics.c) file.

Finally, there are two files related to both eBPF and light-weight tunnels.
One is the [`net/core/lwt_bpf.c`](https://github.com/torvalds/linux/blob/v4.10/net/core/lwt_bpf.c) which is the internal kernel functions for using eBPF for light-weight tunnels.
This other is a short script that tests using eBPF for light-weight tunneling via the `tc` and `ip` command line tools from the iproute2 tool-chain.
It is located in [`samples/bpf/test_tunnel_bpf.sh`](https://github.com/torvalds/linux/blob/v4.10/samples/bpf/test_tunnel_bpf.sh).


-------------

Traffic control:

* http://wiki.linuxwall.info/doku.php/en:ressources:dossiers:networking:traffic_control
* http://tldp.org/HOWTO/Traffic-Control-HOWTO/index.html
* https://wiki.archlinux.org/index.php/Advanced_traffic_control
* http://lartc.org/howto/lartc.adv-filter.policing.html#AEN1393
* http://lartc.org/howto/lartc.netfilter.html
* http://man7.org/linux/man-pages/man8/tc-bpf.8.html

DPDK:

* https://blog.selectel.com/introduction-dpdk-architecture-principles/
* https://www.privateinternetaccess.com/blog/2016/01/linux-networking-stack-from-the-ground-up-part-1/