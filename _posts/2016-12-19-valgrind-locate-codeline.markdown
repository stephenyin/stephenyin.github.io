---
layout: post
title:  "通过 valgrind 输出的偏移地址定位内存泄漏的源码行号"
date:   2016-12-19 16:40:33 +0800
categories: Debug
tags: Linux Debug
---

>有时用 valgrind 定位内存泄露问题时，当内存泄露的位置在动态库( so 文件)中时, 输出的调用栈为问号(*???*)并且没有指明源码的行号.即使尝试了加 *-g* 的编译参数并且程序退出前不执行 *dlclose*,也无济于事.

```
==29941== 17 bytes in 1 blocks are definitely lost in loss record 29 of 197  
==29941==    at 0x402A185: malloc (vg_replace_malloc.c:292)  
==29941==    by 0x4048585: ??? (in /home/xxx/ElastosRDKforDevice/Targets/rdk/x86.gnu.linux.dbg/bin/Elastos.Runtime.so)  
==29941==    by 0x40799F9: ??? (in /home/xxx/ElastosRDKforDevice/Targets/rdk/x86.gnu.linux.dbg/bin/Elastos.Runtime.so)  
==29941==    by 0x407AE2A: ??? (in /home/xxx/ElastosRDKforDevice/Targets/rdk/x86.gnu.linux.dbg/bin/Elastos.Runtime.so)  
==29941==    by 0x407ACC0: ??? (in /home/xxx/ElastosRDKforDevice/Targets/rdk/x86.gnu.linux.dbg/bin/Elastos.Runtime.so)  
==29941==    by 0x407ACF9: ??? (in /home/xxx/ElastosRDKforDevice/Targets/rdk/x86.gnu.linux.dbg/bin/Elastos.Runtime.so)  
==29941==    by 0x400ED76: call_init.part.0 (dl-init.c:78)  
==29941==    by 0x400EE63: _dl_init (dl-init.c:36)  
==29941==    by 0x400110E: ??? (in /lib/i386-linux-gnu/ld-2.19.so)  
```

这种情况下通过每行的偏移地址配合 addr2line 也是可以定位到具体的代码行:

**1. 重新运行 valgrind 命令执行内存泄露的检查工作.**

**2. 查看 valgrind log 中的 thread Id, 如上 *29941***

**3. 在程序退出之前 cat /proc/29941/maps 文件, 可以看到加载动态库的信息**

```
04038000-040ab000 r-xp 00000000 08:05 6560496    /home/xxx/ElastosRDKforDevice/Targets/rdk/x86.gnu.linux.dbg/bin/Elastos.Runtime.so  
040ab000-040ac000 r--p 00073000 08:05 6560496    /home/xxx/ElastosRDKforDevice/Targets/rdk/x86.gnu.linux.dbg/bin/Elastos.Runtime.so  
040ac000-040ad000 rw-p 00074000 08:05 6560496    /home/xxx/ElastosRDKforDevice/Targets/rdk/x86.gnu.linux.dbg/bin/Elastos.Runtime.so  
```

我们关心的是 *r-xp* 这一行, 最前边的 ***04038000*** 就是这个动态库加载的 base 地址.

**4. 然后用发生内存泄露的地址 *0x4048585* 减去 base 地址 *0x4038000*, 得到 *0x10585* 的相对偏移地址**

**5. 利用 addr2line 工具**

```
addr2line -e ./Elastos.Runtime.so 0x10585
```

得到具体源码行号:

```
/home/xxx/ElastosRDKforDevice/Sources/ElastosSources/Elastos/Runtime/Library/eltypes/elstring/elsharedbuf.cpp:9
```

**打完收工.**