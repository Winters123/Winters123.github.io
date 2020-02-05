---
layout: post
title:  "Netmap Learning Notes"
categories: QUIC acceleration
tags:  QUIC acceleration kernel-bypass 
author: Xiangrui Yang
---

* content
{:toc}
### Netmap

#### Kernel:

1. **RX**:

   > $[rdh, rdt)$ indicates the availuable slots for **NIC**.
   >
   > receive data from NIC: **Update head (rdh)**;
   >
   > allocate new skb on the ring: **Update the tail (rdt)**.

2. **TX**:

   > `kernel` allocates `skb` to a data massage that received from `user space`. 
   >
   > the `driver` links the `skb` to the ring, and **updates the tail (tdt)**.
   >
   > NIC reads the massage from DMA and sends it over the link.
   >
   > When DMA is over, the **NIC updates the ring head and possibly send an interrupt**. 
   >
   > When the driver noticed that, it will free the `skb`.

#### Netmap:

​	During the initialization, the `netmap ring` and `netmap buffers` are **pre-allocated**. And each `ring slot` pointed to a `netmap buffer`.

**NOTE:** the `netmap ring` and `netmap_skbuff` are allocated in the **shared memory**.

![image-20200118095437229](https://github.com/Winters123/Winters123.github.io/blob/master/_posts/2020-02-05-netmap/image-20200118095437229.png?raw=true)

1. **RX**:

   > initially on the `netmap ring`, $head = tail$, which indicates that the data is **empty**.
   >
   > and all `netmap buffers` belong to the NIC.
   >
   > When a packet is put on the `netmap buffers`, the NIC notifies netmap application. And moves the `rdh` one slot.
   >
   > then the netmap application orders a `ring sync` (should be a system call), the `tail` pointer is updated to reflect the new data.
   >
   > now the netmap application can read the message, when its done, it moves the `head`. (keeps the equation that $head = tail $).
   >
   > Then the netmap application orders a `ring sync` again, which moves the `rdt`.

2. **TX**:

   > kinda similar (well.. should be opposite to the process in **RX**) to **RX**.

#### Data Structures:



TIP:

from [here](https://gcc.gnu.org/onlinedocs/gcc-4.0.2/gcc/Type-Attributes.html).

```
aligned (`alignment`)
```

This attribute specifies a minimum alignment (in bytes) for variables of the specified type. For example, the declarations:

```
          struct S { short f[3]; } __attribute__ ((aligned (8)));
          typedef int more_aligned_int __attribute__ ((aligned (8)));
     
```

force the compiler to insure (as far as it can) that each variable whose type is `struct S` or `more_aligned_int` will be allocated and aligned *at least* on a 8-byte boundary. On a SPARC, having all variables of type `struct S` aligned to 8-byte boundaries allows the compiler to use the `ldd` and `std` (doubleword load and store) instructions when copying one variable of type `struct S` to another, thus improving run-time efficiency.

1. **netmap_ring**

![image-20200118104702457](https://github.com/Winters123/Winters123.github.io/blob/master/_posts/2020-02-05-netmap/image-20200118104301982.png?raw=true?raw=true)

2. **nm_des:**

   ![image-20200118154322910](https://github.com/Winters123/Winters123.github.io/blob/master/_posts/2020-02-05-netmap/image-20200118154322910.png?raw=true)

3. **netmap_if:**

   ![image-20200118154626341](https://github.com/Winters123/Winters123.github.io/blob/master/_posts/2020-02-05-netmap/image-20200118154626341.png?raw=true)

4. to obtain a buffer pointed by a slot in a ring:

   `void *buf = NETMAP_BUF(ring, slot->buf_idx);`

5. **Useful functions:**

   ```c
   nm_ring_next(struct netmap_ring *, unint32_t) //increment ring pointers with wrap around.
   nm_ring_empty(struct netmap_ring *) //true iff there are no new packets (RX ring) or there are slots available 
   nm_ring_space(struct netmap_ring *) //number of available slots for TX/RX ring.
   nm_close(struct nm_desc *) //pass the num returned by nm_open() to clean up and free it.
   
   ```

   ```c
   struct pollfd pfd = {
       .fd = nmd->fd,
       .events = POLLIN /* and/or POLLOUT */
   }
   int i = poll(pfd, 1, timeout);
   //note: no need to sync again when poll() returns!
   ```








vagrant used for VM configuration

