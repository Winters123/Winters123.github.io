---
layout: post
title:  "CPU usage analysis of quicly"
categories: QUIC_acceleration
tags:  QUIC_acceleration quicly measurement
author: Xiangrui Yang
---

###12-06: CPU usage analysis of `quicly`



> **Abstract:** This blog is about our initial measurement results of `quicly`. The results give us some insights that **with a QUIC implementation based on UNIX socket, crypto functions only consume very limited CPU resources. And this is because kernel network stack processing cost too much CPU resources.** 
>
> 

First we lunched 15 instances of QUIC server/client pairs in parallel on two different servers to profile (uisng `perf`) the CPU usage of `quicly`. During the measurement, each server was sending a 100MB data block to the paired client simultaneously. And the following figure is a snapshot from `server side`. Most CPU time is consumed by `__x64_sys_select` and `__select` functions. 

> According to [this page](https://www.gnu.org/software/libc/manual/html_node/Waiting-for-I_002fO.html), `select` blocks the calling process until there is activity on any of the specific sets of the file descriptors (or until timeout).

![image-20191206140221867](https://github.com/Winters123/Winters123.github.io/blob/master/_posts/2019-12-06-quicly/image-20191206140221867.png?raw=true?raw=trued)

This is a snapshot from `client side`. most CPU time is consumed by `do_syscall_64`, more specifically, by `osq_lock`. Since the lock is in kernel, we assume that this should be something to do with **reading different skb_buff to user space for different sockets**. 

![image-20191206145045154](https://github.com/Winters123/Winters123.github.io/blob/master/_posts/2019-12-06-quicly/image-20191206145045154.png?raw=true?raw=true)

In order to verify it, we used `perf` to re-profile CPU usage when there was just **one QUIC connection**. And the result is shown in these two figures. From the 1st figure, we cannot see the CPU usage from osq_lock anymore, which can verify that the assumption is correct. And from 2nd figure, we found that **__libc_recvmsg contributes most to the CPU usage in user space, accounting 7.29%.** 

![image-20191206152610831](https://github.com/Winters123/Winters123.github.io/blob/master/_posts/2019-12-06-quicly/image-20191206152610831.png?raw=true?raw=true) 

Then, from `server `, the CPU usages of `__x64_sys_select` and `__SELECT` went down to **13.71%** (**21.65%** in former setting) and **20.01%** (**47.71%** in former setting). Meanwhile, the CPU usages of `__libc_sendmsg` and `__sys_sendmsg` went up to **30.09%** (**12.41%** in former setting) and **23.04%** (**15.43%** in former setting). And we did notice an increase in overall throughput when we had just one QUIC connection.

![image-20191206152332001](https://github.com/Winters123/Winters123.github.io/blob/master/_posts/2019-12-06-quicly/image-20191206152332001.png?raw=true?raw=true)



And we also did some measurements on overall CPU usage of quicly:

**Action:** server ------------------(100MB)--------------------------> client

We tested the highest IPC(instructions per cycle) on the server is 4. According to this [blog](http://www.brendangregg.com/blog/2017-05-09/cpu-utilization-is-wrong.html), if IPC is higher than 1, it is likely that the workload is CPU-bound, if it is less than 1, the workload is more likely to be memory-bound. From the result, we can be sure that from the client side (who is receiving the data), the workload is more memory-bound. However, the IPC on server-side is almost 1, which suggesting that the workload is between CPU-bound and memory-bound (if I understand it correctly).

<table>
  <tr>
    <th colspan="3">CPU USAGE (server)</th>
    <th colspan="3">CPU USAGE (client)</th>
  </tr>
  <tr>
    <td>5 conn</td>
    <td>10 conn</td>
    <td>15 conn</td>
    <td>5 conn</td>
    <td>10 conn</td>
    <td>15 conn</td>
  </tr>
  <tr>
    <td>36.60%</td>
    <td>57.10%</td>
    <td>71.30%</td>
    <td>100.90%</td>
    <td>156.90%</td>
    <td>222.50%</td>
  </tr>
  <tr>
    <td>1.04 IPC, 1.142GHz</td>
    <td>1.02 IPC, 1.131GHz</td>
    <td>1 IPC, 1.304GHz</td>
    <td>0.54 IPC 2.451GHz</td>
    <td>0.39 IPC, 2.285GHz</td>
    <td>0.33 IPC, 2.113GHz</td>
  </tr>
</table>



