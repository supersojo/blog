---
title: sctp中一个bug
date: 2019-6-4 22:01:20
tags:
- sctp
categories:
- network
- kernel
---
 
## 问题
syzkaller是Google团队开发的一款针对Linux内核进行模糊测试的开源工具,目前还在不断的维护之中。 
订阅了相关linux内核的邮件列表后，会收到google的sykaller发现的内核bug的相关报告。
```
Hello,
 syzbot found the following crash on:
 HEAD commit:    9221dced Merge tag 'for-linus-20190601' of git://git.kerne..
 git tree:       upstream
 console output: https://syzkaller.appspot.com/x/log.txt?x=114cdc0ea00000
 kernel config:  https://syzkaller.appspot.com/x/.config?x=1fa7e451a5cac069
 dashboard link: https://syzkaller.appspot.com/bug?extid=a9e23ea2aa21044c2798
 compiler:       gcc (GCC) 9.0.0 20181231 (experimental)
 userspace arch: i386
 Unfortunately, I don't have any reproducer for this crash yet.
 IMPORTANT: if you fix the bug, please add the following tag to the commit:
 Reported-by: syzbot+a9e23ea2aa21044c2798@syzkaller.appspotmail.com
 ==================================================================
 BUG: KASAN: slab-out-of-bounds in rt_cache_valid+0x158/0x190
 net/ipv4/route.c:1556
 Read of size 2 at addr ffff8880654f3ac7 by task syz-executor.0/26603
 CPU: 0 PID: 26603 Comm: syz-executor.0 Not tainted 5.2.0-rc2+ #9 Hardware name: Google Google Compute Engine/Google Compute Engine, BIOS Google 01/01/2011 Call Trace:
   __dump_stack lib/dump_stack.c:77 [inline]
   dump_stack+0x172/0x1f0 lib/dump_stack.c:113
   print_address_description.cold+0x7c/0x20d mm/kasan/report.c:188
   __kasan_report.cold+0x1b/0x40 mm/kasan/report.c:317
   kasan_report+0x12/0x20 mm/kasan/common.c:614
   __asan_report_load2_noabort+0x14/0x20 mm/kasan/generic_report.c:130
   rt_cache_valid+0x158/0x190 net/ipv4/route.c:1556
   __mkroute_output net/ipv4/route.c:2332 [inline]
   ip_route_output_key_hash_rcu+0x819/0x2d50 net/ipv4/route.c:2564
   ip_route_output_key_hash+0x1ef/0x360 net/ipv4/route.c:2393
   __ip_route_output_key include/net/route.h:125 [inline]
   ip_route_output_flow+0x28/0xc0 net/ipv4/route.c:2651
   ip_route_output_key include/net/route.h:135 [inline]
   sctp_v4_get_dst+0x467/0x1260 net/sctp/protocol.c:435
   sctp_transport_route+0x12d/0x360 net/sctp/transport.c:297
   sctp_assoc_add_peer+0x53e/0xfc0 net/sctp/associola.c:663
   sctp_process_param net/sctp/sm_make_chunk.c:2531 [inline]
   sctp_process_init+0x2491/0x2b10 net/sctp/sm_make_chunk.c:2344
   sctp_cmd_process_init net/sctp/sm_sideeffect.c:667 [inline]
   sctp_cmd_interpreter net/sctp/sm_sideeffect.c:1369 [inline]
   sctp_side_effects net/sctp/sm_sideeffect.c:1179 [inline]
   sctp_do_sm+0x3a30/0x50e0 net/sctp/sm_sideeffect.c:1150
   sctp_assoc_bh_rcv+0x343/0x660 net/sctp/associola.c:1059
   sctp_inq_push+0x1e4/0x280 net/sctp/inqueue.c:80
   sctp_backlog_rcv+0x196/0xbe0 net/sctp/input.c:339
   sk_backlog_rcv include/net/sock.h:945 [inline]
   __release_sock+0x129/0x390 net/core/sock.c:2412
   release_sock+0x59/0x1c0 net/core/sock.c:2928
   sctp_wait_for_connect+0x316/0x540 net/sctp/socket.c:9039
   __sctp_connect+0xab2/0xcd0 net/sctp/socket.c:1226
   __sctp_setsockopt_connectx+0x133/0x1a0 net/sctp/socket.c:1334
   sctp_setsockopt_connectx_old net/sctp/socket.c:1350 [inline]
   sctp_setsockopt net/sctp/socket.c:4644 [inline]
   sctp_setsockopt+0x22c0/0x6d10 net/sctp/socket.c:4608
   compat_sock_common_setsockopt+0x106/0x140 net/core/sock.c:3137
   __compat_sys_setsockopt+0x185/0x380 net/compat.c:383
   __do_compat_sys_setsockopt net/compat.c:396 [inline]
   __se_compat_sys_setsockopt net/compat.c:393 [inline]
   __ia32_compat_sys_setsockopt+0xbd/0x150 net/compat.c:393
   do_syscall_32_irqs_on arch/x86/entry/common.c:337 [inline]
   do_fast_syscall_32+0x27b/0xd7d arch/x86/entry/common.c:408
   entry_SYSENTER_compat+0x70/0x7f arch/x86/entry/entry_64_compat.S:139
 RIP: 0023:0xf7ff5849
 Code: 85 d2 74 02 89 0a 5b 5d c3 8b 04 24 c3 8b 14 24 c3 8b 3c 24 c3 90 90
 90 90 90 90 90 90 90 90 90 90 51 52 55 89 e5 0f 34 cd 80 <5d> 5a 59 c3 90
 90 90 90 eb 0d 90 90 90 90 90 90 90 90 90 90 90 90
 RSP: 002b:00000000f5df10cc EFLAGS: 00000296 ORIG_RAX: 000000000000016e
 RAX: ffffffffffffffda RBX: 0000000000000007 RCX: 0000000000000084
 RDX: 000000000000006b RSI: 000000002055bfe4 RDI: 000000000000001c
 RBP: 0000000000000000 R08: 0000000000000000 R09: 0000000000000000
 R10: 0000000000000000 R11: 0000000000000000 R12: 0000000000000000
 R13: 0000000000000000 R14: 0000000000000000 R15: 0000000000000000
 Allocated by task 480:
   save_stack+0x23/0x90 mm/kasan/common.c:71
   set_track mm/kasan/common.c:79 [inline]
   __kasan_kmalloc mm/kasan/common.c:489 [inline]
   __kasan_kmalloc.constprop.0+0xcf/0xe0 mm/kasan/common.c:462
   kasan_slab_alloc+0xf/0x20 mm/kasan/common.c:497
   slab_post_alloc_hook mm/slab.h:437 [inline]
   slab_alloc mm/slab.c:3326 [inline]
   kmem_cache_alloc+0x11a/0x6f0 mm/slab.c:3488
   dst_alloc+0x10e/0x200 net/core/dst.c:93
   rt_dst_alloc+0x83/0x3f0 net/ipv4/route.c:1624
   __mkroute_output net/ipv4/route.c:2337 [inline]
   ip_route_output_key_hash_rcu+0x8f3/0x2d50 net/ipv4/route.c:2564
   ip_route_output_key_hash+0x1ef/0x360 net/ipv4/route.c:2393
   __ip_route_output_key include/net/route.h:125 [inline]
   ip_route_output_flow+0x28/0xc0 net/ipv4/route.c:2651
   ip_route_output_key include/net/route.h:135 [inline]
   sctp_v4_get_dst+0x467/0x1260 net/sctp/protocol.c:435
   sctp_transport_route+0x12d/0x360 net/sctp/transport.c:297
   sctp_assoc_add_peer+0x53e/0xfc0 net/sctp/associola.c:663
   sctp_process_param net/sctp/sm_make_chunk.c:2531 [inline]
   sctp_process_init+0x2491/0x2b10 net/sctp/sm_make_chunk.c:2344
   sctp_sf_do_unexpected_init net/sctp/sm_statefuns.c:1541 [inline]
   sctp_sf_do_unexpected_init.isra.0+0x7cd/0x1350 net/sctp/sm_statefuns.c:1441
   sctp_sf_do_5_2_1_siminit+0x35/0x40 net/sctp/sm_statefuns.c:1670
   sctp_do_sm+0x121/0x50e0 net/sctp/sm_sideeffect.c:1147
   sctp_assoc_bh_rcv+0x343/0x660 net/sctp/associola.c:1059
   sctp_inq_push+0x1e4/0x280 net/sctp/inqueue.c:80
   sctp_backlog_rcv+0x196/0xbe0 net/sctp/input.c:339
   sk_backlog_rcv include/net/sock.h:945 [inline]
   __release_sock+0x129/0x390 net/core/sock.c:2412
   release_sock+0x59/0x1c0 net/core/sock.c:2928
   sctp_wait_for_connect+0x316/0x540 net/sctp/socket.c:9039
   __sctp_connect+0xab2/0xcd0 net/sctp/socket.c:1226
   sctp_connect net/sctp/socket.c:4846 [inline]
   sctp_inet_connect+0x29c/0x340 net/sctp/socket.c:4862
   __sys_connect+0x264/0x330 net/socket.c:1834
   __do_sys_connect net/socket.c:1845 [inline]
   __se_sys_connect net/socket.c:1842 [inline]
   __ia32_sys_connect+0x72/0xb0 net/socket.c:1842
   do_syscall_32_irqs_on arch/x86/entry/common.c:337 [inline]
   do_fast_syscall_32+0x27b/0xd7d arch/x86/entry/common.c:408
   entry_SYSENTER_compat+0x70/0x7f arch/x86/entry/entry_64_compat.S:139
 Freed by task 9:
   save_stack+0x23/0x90 mm/kasan/common.c:71
   set_track mm/kasan/common.c:79 [inline]
   __kasan_slab_free+0x102/0x150 mm/kasan/common.c:451
   kasan_slab_free+0xe/0x10 mm/kasan/common.c:459
   __cache_free mm/slab.c:3432 [inline]
   kmem_cache_free+0x86/0x260 mm/slab.c:3698
   dst_destroy+0x29e/0x3c0 net/core/dst.c:129
   dst_destroy_rcu+0x16/0x19 net/core/dst.c:142
   __rcu_reclaim kernel/rcu/rcu.h:222 [inline]
   rcu_do_batch kernel/rcu/tree.c:2092 [inline]
   invoke_rcu_callbacks kernel/rcu/tree.c:2310 [inline]
   rcu_core+0xba5/0x1500 kernel/rcu/tree.c:2291
   __do_softirq+0x25c/0x94c kernel/softirq.c:293
 The buggy address belongs to the object at ffff8880654f3a00
   which belongs to the cache ip_dst_cache of size 176 The buggy address is located 23 bytes to the right of
   176-byte region [ffff8880654f3a00, ffff8880654f3ab0) The buggy address belongs to the page:
 page:ffffea0001953cc0 refcount:1 mapcount:0 mapping:ffff8880a76ad600
 index:0xffff8880654f3c00
 flags: 0x1fffc0000000200(slab)
 raw: 01fffc0000000200 ffffea00026be808 ffffea000181c088 ffff8880a76ad600
 raw: ffff8880654f3c00 ffff8880654f3000 0000000100000002 0000000000000000 page dumped because: kasan: bad access detected
 Memory state around the buggy address:
   ffff8880654f3980: fb fb fb fb fb fb fc fc fc fc fc fc fc fc fc fc
   ffff8880654f3a00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
   ffff8880654f3a80: 00 00 00 00 00 00 fc fc fc fc fc fc fc fc fc fc
                                               ^
     ffff8880654f3b00: fb fb fb fb fb fb fb fb fb fb fb fb fb fb fb fb
     ffff8880654f3b80: fb fb fb fb fb fb fc fc fc fc fc fc fc fc fc fc ================================================================== 
 
 This bug is generated by a bot. It may contain errors.
 See https://goo.gl/tpsmEJ for more information about syzbot.
 syzbot engineers can be reached at syzkaller@googlegroups.com.
 syzbot will keep track of this bug report. See:
 https://goo.gl/tpsmEJ#status for how to communicate with syzbot.
```

## 分析
从上面的信息可以看出在rt_cache_valid中访问了已经释放的dst。rt_cache_valid函数在rcu读临界区内，
先检查下是否是rcu锁问题。rcu锁的作用是防止dst在有读者的时候被释放，而出问题代码本身位于读临界区，
此时dst的回收代码不会执行到。dst是正常引用计数到达0时被rcu softirq reclaim掉的。dst引用统计被put了
多次导致计数为0 被释放了。跟踪下sctp下dst的释放流程，sctp_transport_route先release dst再执行get_dst。
由于release dst不在rcu的读临界区，在dst引用计数为0导致dst被回收后，在执行get_dst流程时引用的dst早已
被rcu sofirq释放。最简单的修复把release dst也放到rcu读临界区，这样在release dst和get dst的时候不
会触发rcu softirq的reclaim动作。

