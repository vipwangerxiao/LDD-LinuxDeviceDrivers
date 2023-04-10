
## 4.4 主动的页面回收(Proactive Reclaim)
-------

### 4.4.X WSS(Working Set Size Estimation)
-------

[系统软件工程师必备技能 - 进程内存的 working set size(WSS)测量](https://blog.csdn.net/juS3Ve/article/details/85333717)


1.  冷热页区分:  为了能识别那些可以回收的页面, 必须对那些不常用的页面有效地进行跟踪, 即 idle page tracking.

2.  进程内存的 working set size(WSS) 估计: 为了在回收了内存之后还能满足业务的需求, 保障业务性能不下降, 需要能预测出业务运行所需要的实际最小内存. brendangregg 大神对此也有描述, [Working Set Size Estimation](https://www.brendangregg.com/wss.html), 并设计了 wss 工具 [Working Set Size (WSS) Tools for Linux](https://github.com/brendangregg/wss).

3.  Meta(原 Facebook) 开发了 [Senpai](https://github.com/facebookincubator/senpai)

4. 2022 International Conference on Service Science (ICSS) 的论文 [eBPF-based Working Set Size Estimation in Memory Management](https://ieeexplore.ieee.org/abstract/document/9860164) 提出了一种基于 eBPF 程序来估计 WSS 的方法.


### 4.4.1 Idle and stale page tracking
-------

[Idle and stale page tracking](https://lwn.net/Articles/461461)

首先是 Google 的方案, 这个特性很好的诠释了上面量两部分的内容, 参见 [V2: idle page tracking / working set estimation](https://lore.kernel.org/patchwork/patch/268228). 其主要实现思路如下:

1.  Google 最开始实现的方案是内核跟踪空闲页面的信息, 并暴露到新增的一个 procfs 的 bitmap 节点 `/sys/kernel/mm/page_idle/bitmap`,  然后 user-space 进程会频繁读取该 sysfs. 参见 [idle memory tracking](https://lore.kernel.org/lkml/cover.1437303956.git.vdavydov@parallels.com), 不过这个方案 CPU 占用率偏高, 并且内存浪费的也不少.

2.  所以目前 Google 尝试在此基础上进行优化, 形成一个新方案, 基于一个名为 [kstaled 的 kernel thread](https://lkml.org/lkml/2011/9/16/368), 参见 [V2, 0/9: idle page tracking / working set estimation](https://lore.kernel.org/lkml/1317170947-17074-1-git-send-email-walken@google.com). 这个 kernel thread 会利用 page 里的 flag 来跟踪 idle page, 所以不会再浪费更多系统内存了, 不过仍然需要挺多 CPU 时间的. 还有一个新加的 kreclaimd 线程来扫描内存, 回收那些空闲 (没人读写访问) 太久的页面. CPU 的开销并不小, 会随着需要跟踪的内存空间大小, 以及扫描的频率而线性增加. 在一个 512GB 内存的系统上, 可能会需要一个 CPU 完全用于做这部分工作. 大多数的时间都是在遍历 reverse-map 列表来查找 page mapping. 他们试过把 reverse-map 的查找去掉, 而是创建一个 PMD page table 链表, 可以改善这部分开销. 这样能减少 CPU 占用率到原来的 2/7. 还有另一个优化是把 kreclaimd 的扫描去掉而直接利用 kstaled 传入的一组页面, 也有明显的效果.

基于 kstaled/kreclaimd 的方案, 没有合入主线, 但是 idle memory tracking 的方案在优化后, 于 4.3 合入了主线, 命名为 [IDLE_PAGE_TRACKING](https://www.kernel.org/doc/html/latest/admin-guide/mm/idle_page_tracking.html). 作者基于这个特性进程运行所需的实际内存预测 (WSS), 并提供了一[系列工具 idle_page_tracking](https://github.com/sjp38/idle_page_tracking) 来完成这个工作, 参见 [Idle Page Tracking Tools](https://sjp38.github.io/post/idle_page_tracking).

Facebook 指出他们也面临过同样的问题, 所有的 workload 都需要放到 container 里去执行, 用户需要明确申明需要使用多少内存, 不过其实没人知道自己真的会用到多少内存, 因此用户申请的内存数量都太多了, 也就有了类似的 overcommit 和 reclaim 问题. Facebook 的方案是采用 [PSI(pressure-stall information)](https://lwn.net/Articles/759781), 根据这个来了解内存是否变得很紧张了, 相应的会把 LRU list 里最久未用的 page 砍掉. 假如这个举动导致更多的 refault 发生. 不过通过调整内存的回收就调整的激进程度可以缓和 refault. 从而达到较合理的结果, 同时占用的 CPU 时间也会小得多. 描述其设计方案的论文在 ASPLOS’22 中发表, 并成为四篇 Best Paper 之一, 参见 [TMO: Transparent Memory Offloading in Datacenters](https://dl.acm.org/doi/10.1145/3503222.3507731). 网络上的分析 [知乎, 大荒落 TMO: Transparent Memory Offloading in Datacenters](https://zhuanlan.zhihu.com/p/477786756).

参见:

phoronix 报道 [Meta's Transparent Memory Offloading Saves Them 20~32% Of Memory Per Linux Server](https://www.phoronix.com/scan.php?page=news_item&px=Meta-Transparent-TMO).

| 日期 | LWN | 翻译 |
|:---:|:----:|:---:|
| 2022/06/20 | [Transparent memory offloading: more memory at a fraction of the cost and power](https://engineering.fb.com/2022/06/20/data-infrastructure/transparent-memory-offloading-more-memory-at-a-fraction-of-the-cost-and-power/) | [公众号 - SegmentFault 思否 --Meta “透明内存卸载” 功能亮相：可为 Linux 服务器节省 20%-32% 内存](https://mp.weixin.qq.com/s/J9hXRtEE-1x9a-I_hGwNPQ) |

Meta(原 Facebook) 博客 [Transparent memory offloading: more memory at a fraction of the cost and power](https://engineering.fb.com/2022/06/20/data-infrastructure/transparent-memory-offloading-more-memory-at-a-fraction-of-the-cost-and-power).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2011/09/28 | Rik van Riel <riel@redhat.com> | [V2: idle page tracking / working set estimation](https://lore.kernel.org/patchwork/patch/268228) | Google 的 kstaled 方案, 通过跟踪那些长期和回收未使用页面, 来减少内存使用, 同时不降低业务的性能. | v2 ☐ | [LORE v1,0/8](https://lore.kernel.org/all/1316230753-8693-1-git-send-email-walken@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/9](https://lore.kernel.org/lkml/1317170947-17074-1-git-send-email-walken@google.com) |
| 2015/07/19 | Vladimir Davydov <vdavydov@parallels.com> | [idle memory tracking](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=d3691d2c6d3e72624c987bbef6f322631bbb2d5d) | Google 的 idle page 跟踪技术, CONFIG_IDLE_PAGE_TRACKING 跟踪长期未使用的页面. | v9 ☑ 4.3-rc1 | [PatchWork RFC]https://lore.kernel.org/lkml/cover.1437303956.git.vdavydov@parallels.com), [REDHAT Merge](https://lists.openvz.org/pipermail/devel/2015-October/067103.html), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=33c3fc71c8cfa3cc3a98beaa901c069c177dc295) |

### 4.4.2 [Proactively reclaiming idle memory](https://lwn.net/Articles/787611)
-------

[LWN: 主动回收较少使用的内存页面](https://blog.csdn.net/Linux_Everything/article/details/96416633)

[[RFC,-V6,0/6] NUMA balancing: optimize memory placement for memory tiering system](https://lore.kernel.org/patchwork/patch/1393431)

[Software-defined far memory in warehouse scale computers](https://blog.acolyer.org/2019/05/22/sw-far-memory)

[LSF/MM 2019](https://lwn.net/Articles/lsfmm2019) 期间, 主动回收 IDLE 页面的议题引起了开发者的关注. 通过对业务持续一段时间的页面使用进行监测, 回收掉那些不常用的或者没必要的页面, 在满足业务需求的前提下, 可以节省大量的内存. 这可能比启发式的 kswapd 更有效. 这包括两部分的内容:


后来还有一些类似的特性也达到了很好的效果.

1.  intel 在对 NVDIMM/PMEM 进行支持的时候, 为了将热页面尽量使用快速的内存设备, 而冷页面尽量使用慢速的内存设备. 因此实现了冷热页跟踪机制. 完善了 idle page tracking 功能, 实现 per process 的粒度上跟踪内存的冷热. 在 reclaim 时将冷的匿名页面迁移到 PMEM 上 (只能迁移匿名页). 同时利用一个 userspace 的 daemon 和 idle page tracking, 来将热内存(在 PMEM 上的) 迁移到 DRA M 中. [ept-idle](https://github.com/intel/memory-optimizer/tree/master/kernel_module).

2.  openEuler 实现的 etmem 内存分级扩展技术, 通过 DRAM + 内存压缩 / 高性能存储新介质形成多级内存存储, 对内存数据进行分级, 将分级后的内存冷数据从内存介质迁移到高性能存储介质中, 达到内存容量扩展的目的, 从而实现内存成本下降.

3.  Amazon 的开发人员 SeongJae Park 基于 DAMON 分析内存的访问, 在此基础上实现了[主动的内存回收机制 DAMON-based Reclamation](https://damonitor.github.io/doc/html/v29-darc-rfc-v2/admin-guide/mm/damon/reclaim.html). 使用 DAMON 监控数据访问, 以找出特定时间段内未访问的冷页, 优先将他们回收.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/02/09 | David Rientjes <rientjes@google.com> | [smaps: extract pmd walker from smaps code](https://lore.kernel.org/patchwork/patch/73779) | Referenced Page flag, 通过在 /proc/PID/smap 中新增 pages referenced 计数的支持, 同时引入 `/proc/PID/clear_refs` 允许用户重置进程页面的 referenced 标记. 通过这种方式用户可以找出特定时间段内被访问过的页. 从而估计出进程的 WSS, 区分冷热页. | v1 ☑ 2.6.22-rc1 | [PatchWork v1 0/3](https://lore.kernel.org/patchwork/patch/73777) |
| 2007/10/09 | Matt Mackall <mpm@selenic.com> | [maps4: pagemap monitoring v4](https://lore.kernel.org/patchwork/patch/95279) | 引入 CONFIG_PROC_PAGE_MONITOR, 管理了 `/proc/pid/clear_refs`, `/proc/pid/smaps`, `/proc/pid/pagemap`, `/proc/kpagecount`, `/proc/kpageflags` 多个接口. | v1 ☑ 2.6.25-rc1 | [PatchWork v4 0/12](https://lore.kernel.org/patchwork/patch/95279) |
| 2018/12/26 | Fengguang Wu <fengguang.wu@intel.com> | [PMEM NUMA node and hotness accounting/migration](https://lore.kernel.org/patchwork/patch/1027864) | 尝试使用 NVDIMM/PMEM 作为易失性 NUMA 内存, 使其对普通应用程序和虚拟内存透明. 其中引入 `/proc/PID/idle_pages` 接口, 用于用户空间驱动的热点页面进行统计. 实现冷热页扫描, 以及页面回收路径下的被动内核冷页面迁移, 改进了用于活动用户空间热 / 冷页面迁移的 move_pages() | RFC v2 ☐ 4.20 | PatchWork RFC,v2,00/21](https://lore.kernel.org/patchwork/patch/1027864), [LKML](https://lkml.org/lkml/2018/12/26/138), [github/intel/memory-optimizer](http://github.com/intel/memory-optimizer) |
| 2021/03/18 | liubo <liubo254@huawei.com> | [etmem: swap and scan](https://gitee.com/openeuler/kernel/issues/I3W4XW) | openEuler 实现的内存分级扩展技术. | v1 ☐ 4.19 | [etmem tools](https://gitee.com/src-openeuler/etmem) |
| 2021/10/19 | SeongJae Park <sjpark@amazon.com> | [Introduce DAMON-based Proactive Reclamation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=bec976b691437d056a92964cb7af07ee1a54221a) | 该补丁集基于 DAMOS 改进了用于生产质量的通用数据访问模式内存管理的引擎, 并在其之上实现了主动回收. | v4 ☑ [5.16-rc1](https://kernelnewbies.org/Linux_5.16#DAMON-based_proactive_memory_reclamation.2C_operation_schemes_and_physical_memory_monitoring) | [PatchWork RFC,00/13](https://patchwork.kernel.org/project/linux-mm/cover/20210720131309.22073-1-sj38.park@gmail.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC,v2,00/14](https://patchwork.kernel.org/project/linux-mm/patch/20210608115254.11930-15-sj38.park@gmail.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC,v3,00/15](https://patchwork.kernel.org/project/linux-mm/cover/20210720131309.22073-1-sj38.park@gmail.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v4 00/15](https://lore.kernel.org/all/20211019150731.16699-1-sj@kernel.org) |


### 4.4.3 提供用户态触发主动回收的接口(syscall or sysfs)
-------

*   process_mrelease


华为终端手机上实现了一个名为 [xreclaimer(Fast process memory reclaimer)](https://github.com/gatieme/MobileModels/blob/huawei/noh-mate40/mm/xreclaimer) 快速回收进程内存的方案, 在进程退出时快速回收.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/08/09 | SeongJae Park <sjpark@amazon.com> | [mm: introduce process_mrelease system call](https://lore.kernel.org/patchwork/patch/1474134) | 引入 process_mrelease() 加速进程的清理, 以更快地释放被杀死的进程的内存.<br> 我们经常希望能杀死不必要的进程, 为更重要的进程释放内存. 例如 Facebook 的 OOM killer 守护程序 oomd 和 Android 的低内存 killer 守护程序 lmkd. 对于这样的系统组件, 能够快速有效地释放内存非常. 但是不幸的是, 当一个进程接收到 SIGKILL 信号的时候并不一定能及时释放自己的内存, 这可能会受到很多因素的影响, 譬如进程的状态(它可能是处于 uninterruptible sleep 态)、正在运行进程的的 core 的 OPP 级别等. 而通过调用这个新的 process_mrelease() 系统调用, 它会在调用者的上下文中释放被杀死的进程的内存. 这种方式下, 内存的释放更可控, 因为它直接在当前 CPU 上运行, 只取决于调用者任务的优先级大小. 释放内存的工作量也将由调用方承担. | v9 ☑ [5.15-rc1](https://kernelnewbies.org/LinuxChanges#Linux_5.15.Introduce_process_mrelease.282.29_system_call) | [PatchWork v9,1/2](https://lore.kernel.org/patchwork/patch/1474134) |

*   per-memcg proactive reclaim

参见 [Proactive reclaim for tiered memory and more](https://lwn.net/Articles/894849)

当前的 MG-LRU 提议[引入了一个 debugfs](https://lore.kernel.org/linux-mm/20220208081902.3550911-12-yuzhao@google.com). 可用于 MGLRU 的调试分析, 从而帮助用户出发主动回收. 这是对 MGLRU 的 lru_gen debugfs 机制的一个增量添加, 但是, 由于这是个 debug 接口, 本身独立于正在使用的回收机制(包括 CONFIG_LRU_GEN 和没有 CONFIG_LRU_GEN), 且没有直接证据表明他是直接用于进行主动回收的, 但是大家都相信通过对这些 debug 信息进行分析可以进行有效地辅助直接回收的进行.

谷歌数据中心使用了用户空间主动回收器. 此外, Meta 的[论文 TMO](https://dl.acm.org/doi/pdf/10.1145/3503222.3507731) 最近引用了一个非常类似的特性, 用于用户空间主动回收.

Google 的 Yosry Ahmed 设计的 [memcg: introduce per-memcg proactive reclaim](https://patchwork.kernel.org/project/linux-mm/cover/20220407224244.1374102-1-yosryahmed@google.com) 针对 MEMCG 提供 SYSFS 作为导出系统级分级信息的接口. 用户空间主动回收器可以持续探测 MEMCG 以回收少量内存. 随着 LRU 不断排序, 这将提供更准确和最新的工作集估计, 并可能提供更具确定性的内存过度使用行为. 内存超分配控制器可以对正在运行的应用程序不断变化的行为提供更主动的响应, 而不是被动响应. 在这种情况下, 用户空间回收器的目的不是完全替代 KSWAPD 或直接回收, 而是主动识别内存节约机会, 回收策略设置的一些冷页, 以释放内存, 用于要求更高的作业或安排新作业. 谷歌数据中心使用了这种用户空间主动回收器.

前面提到 Meta(曾经叫 Facebook) 的论文 [TMO: Transparent Memory Offloading in Datacenters](https://dl.acm.org/doi/10.1145/3503222.3507731) 讨论了一个非常类似的 per-memcg 回收机制, 用于用户空间主动回收.

此外 [Mechanism to induce memory reclaim](https://lore.kernel.org/all/5df21376-7dd1-bf81-8414-32a73cea45dd@google.com) 提出了一些新的想法: 引入一个 per-node 的 sysfs 机制来诱导内存回收, 这对于全局 (非 memcg 的) 回收很有用, 即使在内核或挂载中没有启用 MEMCG, 也是可能的. 这可以选择一个 memcg id 来引导对 memcg 层次结构的回收. 在较低的情况下, 系统上的每个 NUMA 节点 N 将是一个 `/sys/devices/system/node/nodeN/reclaim` 机制. (这类似于现有的 per-Node 的 sysfs compact 机制, 用于从用户空间触发内存规整.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/04/07 | Yosry Ahmed <yosryahmed@google.com> | [memcg: introduce per-memcg proactive reclaim](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=eae3cb2e87ff84547e66211b81301a8f9122840f) | MEMCG 中引入 memory.reclaim 使得用户空间可以通过持续探测 MEMCG 并触发主动回收以回收少量内存. 随着 LRU 不断排序, 这将提供更准确和最新的工作集估计, 并可能提供更具确定性的内存过度使用行为. 内存超分配控制器可以对正在运行的应用程序不断变化的行为提供更主动的响应, 而不是被动响应. 在这种情况下, 用户空间回收器的目的不是完全替代 KSWAPD 或直接回收, 而是主动识别内存节约机会, 回收策略设置的一些冷页, 以释放内存, 用于要求更高的作业或安排新作业.<br> 谷歌数据中心使用了此用户空间主动回收器. | v2 ☑ 5.19-rc1 | [LORE v1,0/1](https://lore.kernel.org/all/20220331084151.2600229-1-yosryahmed@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/4](https://lore.kernel.org/r/20220407224244.1374102-1-yosryahmed@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/4](https://lore.kernel.org/r/20220408045743.1432968-1-yosryahmed@google.com))<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/4](https://lore.kernel.org/r/20220421234426.3494842-1-yosryahmed@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/4](https://lore.kernel.org/r/20220425190040.2475377-1-yosryahmed@google.com) |
| 2022/04/16 | Davidlohr Bueso <dave@stgolabs.net> | [Mechanism to induce memory reclaim](https://lore.kernel.org/all/5df21376-7dd1-bf81-8414-32a73cea45dd@google.com) |  LSFMM 2022 | v1 ☐☑ | [LORE RFC v1,0/6](https://lore.kernel.org/all/5df21376-7dd1-bf81-8414-32a73cea45dd@google.com) |
| 2022/05/18 | Vaibhav Jain <vaibhav@linux.ibm.com> | [memcg: provide reclaim stats via'memory.reclaim'](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=eae3cb2e87ff84547e66211b81301a8f9122840f) | 之前引入的 memcg 文件 memory.reclaim 是只写的, 为用户空间提供了一种触发主动回收的方法. 然而, 像扫描和回收的页面数量这样的回收统计仍然不能直接提供给用户空间. 这个补丁将 memory.reclaim 扩展为可读, 它从与每个 memcg 相关的 'struct vmpressure' 回收过程中返回扫描 / 回收的页面数量. 这将让用户空间评估如何成功地从 memcg 的内存中触发主动回收. | v1 ☐☑ 5.19-rc1 | [LORE v1,0/1](https://lore.kernel.org/r/20220518223815.809858-1-vaibhav@linux.ibm.com) |
| 2022/12/02 | Mina Almasry <almasrymina@google.com> | [mm: Add nodes= arg to memory.reclaim](https://lore.kernel.org/all/20221202223533.1785418-1-almasrymina@google.com) | nodes=arg 指示内核仅扫描给定节点以进行主动回收.<br>"nodes"参数用于允许用户空间根据其策略独立控制降级和回收: 如果内存. 在具有降级目标的节点上调用回收, 它将首先尝试降级;<br> 如果在没有降级目标的节点上调用它, 它将只尝试回收. | v3 ☐☑✓ | [LORE](https://lore.kernel.org/all/20221202223533.1785418-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20221130020328.1009347-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/1](https://lore.kernel.org/r/20221202223533.1785418-1-almasrymina@google.com) |


### 4.4.4 主动回收与内存分级
-------


另外一方面随着内存分级的日益普及, Davidlohr Bueso 在 LSFMM 2022 上提交了一个是引发一些 [mm: proactive reclaim and memory tiering topics](https://patchwork.kernel.org/project/linux-mm/cover/20220416053902.68517-1-dave@stgolabs.net). 主要思路和想法启发自 David 的系统级主动回收的讨论.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/04/16 | Davidlohr Bueso <dave@stgolabs.net> | [mm: proactive reclaim and memory tiering topics](https://patchwork.kernel.org/project/linux-mm/cover/20220416053902.68517-1-dave@stgolabs.net/) |  LSFMM 2022 | v1 ☐☑ | [LORE RFC v1,0/6](https://lore.kernel.org/all/20220416053902.68517-1-dave@stgolabs.net) |


### 4.4.5 硬件驱动的冷热页扫描
-------

[[LSF/MM/BPF TOPIC] Using hardware counters to determine hot/cold pages](https://lore.kernel.org/all/6bbf2c47-05ab-b78c-3165-2eff18962d6d@linux.ibm.com)

PowerPC 体系结构 (POWER10) 支持热/冷页面跟踪功能(Hot/Cold page tracking), 参见 [HotChips2020-Server_Processors_IBM_Starke_POWER10_v33](https://hc32.hotchips.org/assets/program/conference/day1/HotChips2020_Server_Processors_IBM_Starke_POWER10_v33.pdf), 该功能以可配置的页面大小粒度, 提供访问计数器和访问关联详细信息. 早期 RFC 版本可以在 [kvaneesh/linux](https://github.com/kvaneesh/linux/commit/b472e2c8080823bb4114c286270aea3e18ffe221) 找到.

可能适用的场景:

1. 页面回收/降级.

2. THP 利用率

3. 页面提升.


# 5 Swappiness
-------


## 5.1 Swappiness 倾向
-------

Swappiness 用于设置将页面从物理内存交换到交换空间以及从页面缓存中删除页面之间的平衡. 可以通过引入 `/proc/sys/vm/swappiness` 控制系统在进行 swap 时, 内存使用的相对权重.

swappiness 参数值可设置范围在 `0~100` 之间.

1.      参数值越低, 就会让 Linux 系统尽量少用 swap 分区, 多用内存;

2.      参数值越高就是反过来, 使内核更多的去使用 swap 空间.

假设 swappiness 值为 60, 表示当剩余物理内存低于 40%(40=100-60)时, 开始使用 swap 分区.

[swappiness 参数的含义和设置](https://www.maixj.net/ict/swappiness-22576)

[Linux 内核页回收 swappiness 参数有着什么作用](https://www.linuxprobe.com/linuxkernel-swappiness-meaning.html)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2002/10/12 | Andrew Morton <akpm@digeo.com> | [reduced and tunable swappiness](https://github.com/gatieme/linux-history/commit/8f7a14042e1f5fc814fa60956e3a2dcf5744a0ed) | 引入 `/proc/sys/vm/swappiness` 控制 VM 取消页面映射和交换内容的倾向. 在所有用于控制交换性的机制是: 宁愿回收 page cache, 也不愿将匿名页放到非活动列表. 对 swappiness 机制的控制如下:<br>1. 如果系统中有大量的匿名页, 我们倾向于将匿名页安置到非活动列表中.<br>2. 如果页面回收处于困境(比如更多的扫描正在发生), 然后选择将匿名页带到非活动列表. 这基本上是 2.4 算法.<br>3. 如果 `/proc/sys/vm/swappiness` 控制高, 则倾向于将映射的页面带到非活动列表中.<br> 执行起来很简单: 将上述三件事计算成百分比并相加, 如果[超过 100%, 那么开始回收匿名页](https://elixir.bootlin.com/linux/v2.5.43/source/mm/vmscan.c#L594). | v1 ☑ [2.5.43](https://elixir.bootlin.com/linux/v2.5.43/source/mm/vmscan.c#L588) | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/685701)<br>*-*-*-*-*-*-*-* <br>[COMMIT HISTORY](https://github.com/gatieme/linux-history/commit/8f7a14042e1f5fc814fa60956e3a2dcf5744a0ed) |
| 2002/10/12 | Christoph Lameter <clameter@engr.sgi.com> | [vmscan: skip reclaim_mapped determination if we do not swap](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2903fb1694dcb08a3c1d9d823cfae7ba30e66cd3) | 如果不是 may_swap, 则跳过 swappiness 机制检查回收匿名页的流程. | v1 ☑ [2.6.16](https://elixir.bootlin.com/linux/v2.6.16/source/mm/vmscan.c#L1205) | [COMMIT 1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=80e4342601abfafacb5f20571e40b56d73d10819), [COMMIT 2](https://github.com/gatieme/linux-history/commit/072eaa5d9cc3e63f567ffd9ad87b36194fdd8010), [COMMIT 3](https://github.com/gatieme/linux-history/commit/072eaa5d9cc3e63f567ffd9ad87b36194fdd8010) |
| 2007/10/16 | Andrea Arcangeli <andrea@suse.de> | [make swappiness safer to use](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4106f83a9f86afc423557d0d92ebf4b3f36728c1) | Swappiness 不是一个安全的 sysctl.<br>1. 将它设置为 0 会挂起系统. 这是一种极端情况, 但即使将其设置为 10 或更低, 也会浪费大量 cpu 而没有取得很大进展.<br>2. 有一些客户想要使用 swappiness, 但由于当前实现的原因, 他们不能使用(如果你改变它, 系统停止交换, 它真的停止交换, 如果你真的必须交换一些东西, 任何事情都不能正常工作).<br> 这个补丁来自 Kurt Garloff 使 swappiness 使用更安全, 并且没有更多巨大的 cpu 占用或低 swappiness 值引起的挂起. 计算 swap_tendency 时考虑活动和非活动页面的不平衡比例 $\frac {NR_ACTIVE}{NR_INACTIVE + 1} \times \frac {vm\_swappiness + 1}{100} \times \frac {mapped\_ratio}{100}$. | v1 ☑ [2.6.24](https://kernelnewbies.org/Linux_3.16#Memory_management) | [PatchWork v3](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4106f83a9f86afc423557d0d92ebf4b3f36728c1) |
| 2007/11/26 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [per-zone and reclaim enhancements for memory controller take 3 [8/10] modifies vmscan.c for isolate globa/cgroup lru activity](https://lkml.org/lkml/2007/11/26/366) | 隔离全局和 cgroup 内存回收, 防止 cgroup 内存回收影响全局内存回收. per-zone 的页面回收感知 MEMCG 的其中一个补丁. 将原本 swap_tendency 和 reclaim_mapped 的处理流程分离新一个新函数 calc_reclaim_mapped() 中.<br> 当使用内存控制器时, 有 2 级内存回收.<br>1. 区域内存回收, 因为系统 / zone 内内存短缺.<br>2. cgroup 内存回收, 因为达到限制.<br> 这两个可以通过 sc->mem_cgroup 参数来区分. 这个补丁试图使内存 cgroup 回收程序避免影响系统 / 区域内存回收. 这个补丁插入 if (scan_global_lru()) 并钩子到 memory_cgroup 回收支持函数. | v3 ☑ [2.6.25-rc1](https://elixir.bootlin.com/linux/v2.6.25/source/mm/vmscan.c#L1057) | [PatchWork v3](https://lore.kernel.org/patchwork/patch/98041), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1cfb419b394ba82745c54ff05436d598ecc2dbd5) |
| 2008/06/11 | Rik van Riel <riel@redhat.com> | [VM pageout scalability improvements (V12)](https://lore.kernel.org/patchwork/patch/118966) | 这里我们关心的是它将 LRU 中匿名页和文件页分开成两个链表进行管理时引入的平衡策略, 用于平衡我们扫描匿名列表和扫描文件列表的数量. 引入 [get_scan_ratio()](https://elixir.bootlin.com/linux/v2.6.28/source/mm/vmscan.c#L1332) 来确定确定对匿名页 LRU 列表和文件页 LRU 列表的扫描力度. 每一组 LRU
列表的相对值是通过查看我们已经旋转回活动列表而不是驱逐的页面的部分来确定的. %[0] 指定对匿名页 LRUs 施加多大压力, 而 %[1] 确定对文件 LRUs 施加多大压力. | v12 ☑ [2.6.28-rc1](https://kernelnewbies.org/Linux_2_6_28#Various_core) | [PatchWork v2](https://lore.kernel.org/patchwork/patch/118966), [关键 commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4f98a2fee8acdb4ac84545df98cccecfd130f8db) |
| 2021/07/27 | Rik van Riel <riel@redhat.com> | [mm: Enable suspend-only swap spaces](https://lore.kernel.org/patchwork/patch/1468228) | 引入一个新的 SWAP_FLAG_HIBERNATE_ONLY 添加一个交换区域, 但是不允许进行通用交换, 只能在 SUSPEND/HIBERNATE 时挂起到磁盘时使用. 目前不能在不启用特定区域的通用交换的情况下启用休眠, 对此的一种半变通方法是将对 swapon() 的调用延迟到尝试休眠之前, 然后在休眠完成之后调用 swapoff(). 这有点笨拙, 而且在保持交换脱离 hibernate 区域方面也不起作用. 目前 SWAP_FLAG_HIBERNATE_ONLY 设置的交换区域将不会出现在 SwapTotal 和 SwapFree 下的 /proc/meminfo 中, 因为它们不能作为常规交换使用, 但是这些区域仍然出现在 /proc/swap 中. | v4 ☐ | [PatchWork v4](https://lore.kernel.org/patchwork/patch/1468228) |
| 2022/02/17 | Peter Xu <peterx@redhat.com> | [mm: Rework zap ptes on swap entries](https://patchwork.kernel.org/project/linux-mm/cover/20220217060746.71256-1-peterx@redhat.com/) | 615245 | v5 ☐☑ | [LORE v5,0/4](https://lore.kernel.org/r/20220217060746.71256-1-peterx@redhat.com) |





## 5.2 MEMCG Swap
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/12/02 | KOSAKI Motohiro <kosaki.motohiro@jp.fujitsu.com> | [memcg: swappiness](https://lkml.org/lkml/2008/12/2/21) | 引入 per-memcg 的 swappiness, 可以用来对 per-memcg 进行精确控制. | v2 ☑ [2.6.29-rc1](https://kernelnewbies.org/Linux_2_6_29#Memory_controller_swap_management_and_other_improvements) | [LKML](https://lkml.org/lkml/2008/12/2/21), [PatchWork](https://lore.kernel.org/patchwork/patch/136809) |
| 2014/04/14 | Michal Hocko <mhocko@suse.cz> | [vmscan: memcg: Always use swappiness of the reclaimed memcg swappiness and oom_control](https://lore.kernel.org/patchwork/patch/459015) | NA | RFC ☑  | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/459015), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=688eb988d15af55c1d1b70b1ca9f6ce58f277c20) |
| 2014/04/16 | Johannes Weiner <hannes@cmpxchg.org> | [mm: memcontrol: remove hierarchy restrictions for swappiness and oom_control](https://lore.kernel.org/patchwork/patch/456852) | swappiness 和 oom_control 目前不能被调整在一个等级的一部分的 memcg, 但不是那个等级的根. 当打开层次模式时, 他们无法配置 per-memcg 的 swappiness 和 oom_control. 但是这种限制并没有很好的理由. swappiness 和 oom_control 的设置是从其限制触发回收和 OOM 调用的任何 memcg 获取的, 而不管它在层次结构树中的位置. 这个补丁允许 <br>1. 在任何组上设置 swappiness. root memcg 上的配置读取了全局虚拟机的 swappiness, 也使它是可写的.<br>2. 允许在任何非根 memcg 上禁用 OOM 杀手.  | RFC ☑ 3.16-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/456852), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3dae7fec5e884a4e72e5416db0894de66f586201) |


## 5.3 pagemap swap
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/07/30 | Tiberiu A Georgescu <tiberiu.georgescu@nutanix.com> | [pagemap: swap location for shared pages](https://lore.kernel.org/patchwork/patch/1470271) | NA | v2 ☐ v5.14-rc4 | [PatchWork RFC,0/4](https://lore.kernel.org/patchwork/patch/1470271) |
| 2021/08/17 | Peter Xu <peterx@redhat.com> | [mm: Enable PM_SWAP for shmem with PTE_MARKER](https://lore.kernel.org/patchwork/patch/1473423) | 这个补丁集在 shmem 上启用 pagemap 的 PM_SWAP. IOW 用户空间将能够检测 shmem 页面是否被换出, 就像匿名页面一样.<br> 可以使用  CONFIG_PTE_MARKER_PAGEOUT 来启用该特性. 当启用时, 它会在 shmem 页面上带来 0.8% 的换入性能开销, 所以作者还没有将它设置为默认值. 然而, 以作者的看法, 0.8% 仍然在一个可接受的范围内, 我们甚至可以使它最终默认. | v2 ☐ v5.14-rc4 | [PatchWork RFC,0/4](https://lore.kernel.org/patchwork/patch/1473423) |
| 2022/10/23 | Kairui Song <ryncsn@gmail.com> | [swap: add a limit for readahead page-cluster value](https://patchwork.kernel.org/project/linux-mm/patch/20221023162533.81561-1-ryncsn@gmail.com/) | 687949 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221023162533.81561-1-ryncsn@gmail.com) |


## 5.4 swap IO
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/12/16 | NeilBrown <neilb@suse.de> | [Repair SWAP-over-NFS](https://patchwork.kernel.org/project/linux-mm/cover/163969801519.20885.3977673503103544412.stgit@noble.brown) | NA | v2 ☐ | [PatchWork 00/18,V2](https://patchwork.kernel.org/project/linux-mm/cover/163969801519.20885.3977673503103544412.stgit@noble.brown) |

# 6 PageCache
-------

[The future of the page cache](https://lwn.net/Articles/712467)

[vmtouch](https://github.com/hoytech/vmtouch) 是一个 Linux 下管理和控制文件系统缓存的工具. 它可以用来, 查看一个文件 (或者目录) 哪些部分在内存中, 把文件调入内存, 把文件清除出内存, 把文件锁住在内存中而不被换出到磁盘上. 参见 [vmtouch——Linux 下的文件缓存管理神器播](https://blog.csdn.net/weixin_29762151/article/details/116554695), [vmtouch - the Virtual Memory Toucher](https://www.cnblogs.com/zengkefu/p/5636273.html), [vmtouch 实现原理解析](https://blog.csdn.net/DKH63671763/article/details/87990669), [ebpf 实践之 查看文件占用的缓存大小](https://www.jianshu.com/p/32aff371d6f5), [利用 vmtouch 管理文件的 page cache](http://wangxuemin.github.io/2016/02/15 / 利用 vmtouch 管理文件的 page%20cache)

## 6.1 PAGE CACHE
-------


| 补丁 | 描述 |
|:---:|:---:|
| [Import 1.1.69](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/diff/mm/filemap.c?id=ea8a68b948397fa9cd1c8a1a6b61a4dc4bc99ec5) | 引入 page cache |
| [Import 1.3.21](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/diff/mm/filemap.c?id=13e539119b02e4af5daf6c55b31e6273c9a084a6) | 完善了 page cache 的功能 |
| [Import 1.3.50](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/diff/mm/filemap.c?id=22accfc2b4fe4bc6a635c70c1a05a9a80abc81ea) | 引入了 shrink_mmap() 机制来释放 page cache 的空间. |
| [Import 1.3.53](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/diff/mm/filemap.c?id=0e8625c7689bef9a38293aa927add4d9704e48be) | 引入了 page_cache_size, 统计 page cache 的大小. |
| [Linux 2.3.7pre1](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/diff/mm/filemap.c?h=2.3.7pre1&id=344971f8de0ecf3fb7ea642e319aad5865b23529) | 统一了 page cache 和 buffer 的框架. |
| [Import 2.3.16pre1](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/diff/mm/filemap.c?id=9aa2c66ac214f71cb051ba7c1adf313d9e160ee1) | 引入了 pagemap-LRU |


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/06/17 | Matthew Wilcox <willy@infradead.org> | [Convert page cache to XArray](https://lore.kernel.org/patchwork/patch/951137) | 将 page cahce 的组织结构从 radix tree 切换到 [xarray](https://lwn.net/Articles/745073). | v13 ☑ 4.20-rc1 | [PatchWork v14,00/74](https://lore.kernel.org/patchwork/patch/950304) |
| 2020/06/10 | Matthew Wilcox <willy@infradead.org> | [Large pages in the page cache](https://lore.kernel.org/patchwork/patch/1254710) | NA | v6 ☑ 5.9-rc1 | [PatchWork RFC,v6,00/51](https://patchwork.kernel.org/project/linux-mm/cover/20200610201345.13273-1-willy@infradead.org) |
| 2020/06/29 | Matthew Wilcox <willy@infradead.org> | [THP prep patches](https://patchwork.kernel.org/project/linux-mm/cover/20200629151959.15779-1-willy@infradead.org) | NA | v1 ☑ 2.5.8 | [PatchWork](https://patchwork.kernel.org/project/linux-mm/cover/20200629151959.15779-1-willy@infradead.org) |
| 2020/04/22 | Jan Kara <jack@suse.cz> | [mm: Speedup page cache truncation](https://lore.kernel.org/patchwork/patch/1229535) | 页面缓存到 xarray 的转换(关键 commit 69b6c1319b6"mm:Convert truncate to xarray") 使页面缓存截断的性能降低了约 10%, 参见 [Truncate regression due to commit 69b6c1319b6](https://lore.kernel.org/linux-mm/20190226165628.GB24711@quack2.suse.cz). 本系列补丁旨在改进截断, 以恢复部分回归.<br>1.
第一个补丁修复了一个长期存在的错误, 我在调试补丁时发现了 xas_for_each_marked().<br>2. 剩下的补丁则致力于停止清除 xas_store() 中的标记, 从而将截断性能提高约 6%. | v2 ☐ | [PatchWork 0/23,v2](https://patchwork.kernel.org/project/linux-mm/cover/20200204142514.15826-1-jack@suse.cz) |
| 2020/10/25 | Kent Overstreet <kent.overstreet@gmail.com> | [generic_file_buffered_read() improvements](https://lore.kernel.org/patchwork/patch/1324435) | 这是一个小补丁系列, 已经在 bcachefs 树中出现了一段时间. 在缓冲读取路径中, 我们在页面缓存中查找一个页面, 然后在循环中从该页面进行复制, 即在查找每个页面之间混合数据副本. 当我们从页面缓存中进行大量读取时, 这是相当大的开销.<br> 这只是重写了 generic_file_buffered_read() 以使用 find_get_pages_contig() 并处理页面数组. 对于大型缓冲读取, 这是一个非常显著的性能改进, 并且不会降低单页读取的性能.<br>generic_file_buffered_read() 被分解成多个函数, 这些函数在某种程度上更容易理解. | v2 ☑ 5.11-rc1 | [2020/06/10](https://lore.kernel.org/patchwork/patch/1254393)<br>*-*-*-*-*-*-*-* <br>[2020/06/19 v2](https://lore.kernel.org/patchwork/patch/1254398)<br>*-*-*-*-*-*-*-* <br>[2020/06/19 v3](https://lore.kernel.org/patchwork/patch/1258432)<br>*-*-*-*-*-*-*-* <br>[2020/10/17 v4](https://lore.kernel.org/patchwork/patch/1322156)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2,0/2](https://lore.kernel.org/patchwork/patch/1324435) |
| 2021/10/20 | Yang Shi <shy828301@gmail.com> | [Solve silent data loss caused by poisoned page cache (shmem/tmpfs)](https://patchwork.kernel.org/project/linux-mm/cover/20210930215311.240774-1-shy828301@gmail.com) | 为了让文件系统意识到有毒的 (poisoned) 页面.<br> 在讨论拆分页面缓存 THP 以脱机有毒页面的补丁时, Noaya 提到了一个[更大的问题](https://lore.kernel.org/linux-mm/CAHbLzkqNPBh_sK09qfr4yu4WTFOzRy+MKj+PA7iG-adzi9zGsg@mail.gmail.com/T/#m0e959283380156f1d064456af01ae51fdff91265), 它阻止了这一工作的进行, 因为如果发生不可纠正的错误, 页面缓存页面将被截断. 深入研究后发现, 如果页面脏, 这种方法 (截断有毒页面) 可能导致所有非只读文件系统的静默数据丢失. 对于内存中的文件系统, 例如 shmem/tmpfs, 情况可能更糟, 因为数据块实际上已经消失了. 为了解决这个问题, 我们可以将有毒的脏页面保存在页面缓存中, 然后在任何后续访问时通知用户, 例如页面错误、读 / 写等. 可以按原样截断干净的页, 因为稍后可以从磁盘重新读取它们. 结果是, 文件系统可能会发现有毒的页面, 并将其作为健康页面进行操作, 因为除了页面错误之外, 所有文件系统实际上都不会检查页面是否有毒或是否在所有相关路径中. 通常, 在将有毒页面保存在页面缓存中以解决数据丢失问题之前, 我们需要让文件系统知道有毒页面. | v3 ☐ | [2021/09/30 PatchWork RFC,v3,0/5](https://patchwork.kernel.org/project/linux-mm/cover/20210930215311.240774-1-shy828301@gmail.com)<br>*-*-*-*-*-*-*-* <br>[2021/10/14 PatchWork RFC,v4,0/6](https://patchwork.kernel.org/project/linux-mm/cover/20211014191615.6674-1-shy828301@gmail.com)<br>*-*-*-*-*-*-*-* <br>[2021/10/20 PatchWork v5,0/6](https://patchwork.kernel.org/project/linux-mm/cover/20211020210755.23964-1-shy828301@gmail.com) |
| 2022/06/24 | CGEL <cgel.zte@gmail.com> | [drop caches: skip tmpfs and ramfs](https://patchwork.kernel.org/project/linux-mm/patch/20220624182121.995294-1-ran.xiaokai@zte.com.cn/) | 653700 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220624182121.995294-1-ran.xiaokai@zte.com.cn) |
| 2022/07/01 | Jens Axboe <axboe@kernel.dk> | [mm: honor FGP_NOWAIT for page cache page allocation](https://patchwork.kernel.org/project/linux-mm/patch/fda6ea77-97c5-7235-421a-21ca5b2c48f9@kernel.dk/) | 655931 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/fda6ea77-97c5-7235-421a-21ca5b2c48f9@kernel.dk)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/dd47b882-1a18-317c-9906-e73c5487678c@kernel.dk) |


## 6.2 页面预读(readahead)
-------

[linux 文件预读发展过程](https://blog.csdn.net/jinking01/article/details/106541116)

[浅谈 Linux Kernel 的预读算法](http://www.caturra.cc/2021/08/31 / 浅谈 linux-kernel 的预读算法) 基于 Linux 4.18.20 对预读进行了

[LWN: Readahead: the documentation I wanted to read](https://lwn.net/Articles/888715)

kai_ding 的 [Linux 文件系统预读](https://blog.csdn.net/kai_ding/article/details/17322787) 以三个实际读取的实例程序讲解了 linux-3.12 的预读算法. [Linux 文件系统预读 (一)](https://blog.csdn.net/kai_ding/article/details/17322787) 讲解了单进程规则顺序读时预读算法的工作. [Linux 文件系统预读 (二)](https://blog.csdn.net/kai_ding/article/details/19957763) 讲解了单进程不规则顺序读 (一共进行了三次读, 顺序读, 且读的大小不定, 有超过最大预读量的, 也有低于最大预读量的) 下预读的工作行为. [Linux 文件系统预读 (三)](https://blog.csdn.net/kai_ding/article/details/20112753) 讲解了多进程交织顺序读行为下的预读是如何处理的.

[2.4.18 预读算法详解](https://blog.csdn.net/liuyuanqing2010/article/details/6705338)

[Linux readahead: less tricks for more](https://www.kernel.org/doc/ols/2007/ols2007v2-pages-273-284.pdf)


从用户角度来看, 这一节对了解 Linux 内核发展帮助不大, 可跳过不读; 但对于技术人员来说, 本节可以展现教材理论模型到工程实现的一些思考与折衷, 还有软件工程实践中由简单粗糙到复杂精细的演变过程


系统在读取文件页时, 如果发现存在着顺序读取的模式时, 就会预先把后面的页也读进内存, 以期之后的访问可以快速地访问到这些页, 而不用再启动快速的磁盘读写.

### 6.2.1 原始的预读框架
-------

Linux 内核的一大特色就是支持最多的文件系统, 并拥有一个虚拟文件系统 (VFS) 层. 早在 2002 年, 也就是 2.5 内核的开发过程中, Andrew Morton 在 VFS 层引入了文件预读的基本框架, 以统一支持各个文件系统. 如图所示, Linux 内核会将它最近访问过的文件页面缓存在内存中一段时间, 这个文件缓存被称为 pagecache. 如下图所示. 一般的 read()操作发生在应用程序提供的缓冲区与 pagecache 之间. 而预读算法则负责填充这个 pagecache. 应用程序的读缓存一般都比较小, 比如文件拷贝命令 cp 的读写粒度就是 4KB; 内核的预读算法则会以它认为更合适的大小进行预读 I/O, 比比如 16-128KB.


![以 pagecache 为中心的读和预读](./images/0002-1-readahead_page_cache.gif)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2002/04/09 | Andrew Morton <akpm@digeo.com> | [readahead](https://github.com/gatieme/linux-history/commit/8fa498462272fec2c16a92a9a7f67d005225b640) | 统一的预读框架, 预读算法的雏形 | v1 ☑ 2.5.8 | [HISTORY commit](https://github.com/gatieme/linux-history/commit/8fa498462272fec2c16a92a9a7f67d005225b640) |
| 2003/02/03 | Andrew Morton <akpm@digeo.com> | [implement posix_fadvise64()](https://github.com/gatieme/linux-history/commit/fccbe3844c29beed4e665b1a5aafada44e133adc) | 引入 posix_fadvise64 | v1 ☑ 2.5.60 | [HISTORY commit](https://github.com/gatieme/linux-history/commit/fccbe3844c29beed4e665b1a5aafada44e133adc) |


### 6.2.2 早期预读算法及其优化
-------

一开始, 内核的预读方案如你所想, 很简单. 就是在内核发觉可能在做顺序读操作时, 就把后面的 128 KB 的页面也读进来.

大约一年之后, Linus Torvalds 把 mmap 缺页 I/O 的预取算法单独列出, 从而形成了 read-around/read-ahead 两个独立算法.


![Linux 中的 read-around, read-ahead 和 direct read](./images/0002-2-readahead_algorithm.gif)

1.  read-around 算法适用于那些以 mmap 方式访问的程序代码和数据, 它们具有很强的局域性 (locality of reference) 特征. 当有缺页事件发生时, 它以当前页面为中心, 往前往后预取共计 128KB 页面.

2.  readahead 算法主要针对 read() 系统调用, 它们一般都具有很好的顺序特性. 但是随机和非典型的读取模式也大量存在, 因而 readahead 算法必须具有很好的智能和适应性.

*   read-around 与 read-ahead 算法

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2003/07/03 | Andrew Morton <akpm@zip.com.au> | [readahead optimisations](https://github.com/gatieme/linux-history/commit/82a333fa1948869322f32a67223ea8d0ae9ad8ba) | 最早的 read-around 思想实现 | v1 ☑ 2.5.27 | [HISTORY commit](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?h=v2.5.27&id=b6938a7bd23a74cfa7a81c0765ec255ea1c7e12e) |
| 2003/07/03 | Linus Torvalds <torvalds@home.osdl.org> | [Simplify and speed up mmap read-around handling](https://github.com/gatieme/linux-history/commit/82a333fa1948869322f32a67223ea8d0ae9ad8ba) | Linus Torvalds 把 mmap 缺页 I/O 的预取算法单独列出, 从而形成了 [read-around](https://elixir.bootlin.com/linux/v2.5.75/source/mm/filemap.c#L997)/read-ahead 两个独立算法 | v1 ☑ 2.5.75 | [HISTORY commit](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?h=v2.5.75&id=82a333fa1948869322f32a67223ea8d0ae9ad8ba) |

这种固定的 128 KB 预读方案显然不是最优的. 它没有考虑系统内存使用状况和进程读取情况. 当内存紧张时, 过度的预读其实是浪费, 预读的页面可能还没被访问就被踢出去了. 还有, 进程如果访问得凶猛的话, 且内存也足够宽裕的话, 128KB 又显得太小家子气了.


*   顺序读与随机读

后续通过 Steven Pratt、Ram Pai 等人的大量工作, readahead 算法进一步完善. 其中最重要的一点是实现了对随机读的完好支持. 随机读在数据库应用中处于非常突出的地位. 在此之前, 预读算法以离散的读页面位置作为输入, 一个多页面的随机读会触发 "顺序预读". 这导致了预读 I/O 数的增加和命中率的下降. 改进后的算法通过监控所有完整的 read() 调用, 同时得到读请求的页面偏移量和数量, 因而能够更好的区分顺序读和随机读.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2005/01/03 | Steven Pratt <slpratt@austin.ibm.com>, Ram Pai <linuxram@us.ibm.com> | [Simplified readahead](https://github.com/gatieme/linux-history/commit/6f734a1af323ab4690610ecd575198ae219b6fe8) | 引入读大小参数, 代码简化及优化; 支持随机读. | v1 ☑ 2.6.11 | [HISTORY commit 1](https://github.com/gatieme/linux-history/commit/6f734a1af323ab4690610ecd575198ae219b6fe8), [HISTORY commit 2](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?h=v2.6.11&id=250c01d06ccb125519cc9958d938f41736868be9) |
| 2005/03/07 | Oleg Nesterov <oleg@tv-sign.ru> | [readahead: improve sequential read detection](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?id=671ccb4b50a6ef21e8c0ed0ef9070098295e1e61) | 支持非对齐顺序读. | v1 ☑ [2.6.12](https://kernelnewbies.org/Linux_2_6_12) | [commit 1](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?id=671ccb4b50a6ef21e8c0ed0ef9070098295e1e61), [commit 2](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?id=577a3dd8fd68d24056075fdf479a1627586f8c46) |


*   顺序性检测

参见 [Linux 内核的文件预读机制详细详解](https://blog.csdn.net/kunyus/article/details/104620057)

为了保证预读命中率, Linux 只对顺序读 (sequential read) 进行预读. 内核通过验证如下两个条件来判定一个 read() 是否顺序读:

1.  这是文件被打开后的[第一次读](https://elixir.bootlin.com/linux/v2.6.22/source/mm/readahead.c#L476), 并且读的是[文件首部](https://elixir.bootlin.com/linux/v2.6.22/source/mm/readahead.c#L494);

2.  当前的读请求与前一 (记录的) 读请求在文件内的位置是连续的.

如果不满足上述顺序性条件, 就判定为随机读. 任何一个随机读都将终止当前的顺序序列, 从而终止预读行为 (而不是缩减预读大小). 注意这里的空间顺序性说的是文件内的偏移量, 而不是指物理磁盘扇区的连续性. 在这里 Linux 作了一种简化, 它行之有效的基本前提是文件在磁盘上是基本连续存储的, 没有严重的碎片化.

### 6.2.3 按需预读(On-demand Readahead)
-------

**2.6.23(2007 年 10 月发布)** 的内核引入了在这个领域耕耘许久的吴峰光的一个[按需预读的算法]((https://lwn.net/Articles/235164). 所谓的按需预读, 就是内核在读取某页不在内存时, 同步把页从外设读入内存, 并且, 如果发现是顺序读取的话, 还会把后续若干页一起读进来, 这些预读的页叫预读窗口; 当内核读到预读窗口里的某一页时, 如果发现还是顺序读取的模式, 会再次启动预读, 异步地读入下一个预读窗口.

该算法关键就在于适当地决定这个预读窗口的大小, 和哪一页做为异步预读的开始. 它的启发式逻辑也非常简单, 但取得不了错的效果. 此外, 对于两个进程在同一个文件上的交替预读, 2.6.24 增强了该算法, 使其能很好地侦测这一行为.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2005/10/06 | WU Fengguang <wfg@mail.ustc.edu.cn> | [Adaptive file readahead](https://lwn.net/Articles/155510) | 自适应预读算法 | v1 ☐ | [LWN](https://lwn.net/Articles/155097) |
| 2009/4/10 | Wu Fengguang <fengguang.wu@intel.com> | [filemap and readahead fixes for linux-next](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=d30a11004e3411909f2448546f036a011978062e) | bugfix | v1 ☑ 2.6.31-rc1 | [LORE v1,00/14](https://lore.kernel.org/lkml/20090407115039.780820496@intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/9](https://lore.kernel.org/lkml/20090410060957.442203404@intel.com), [LKML v2,0/9](https://lkml.org/lkml/2009/4/10/37) |
| 2009/4/10 | Wu Fengguang <fengguang.wu@intel.com> | [context readahead for concurrent IO](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=10be0b372cac50e2e7a477852f98bf069a97a3fa) | bugfix | v1 ☑ 2.6.31-rc1 | [LORE v1,0/3](https://lkml.kernel.org/lkml/20090410131247.764370473@intel.com) |
| 2011/05/17 | WU Fengguang <wfg@mail.ustc.edu.cn> | [512K readahead size with thrashing safe readahead](https://lwn.net/Articles/372384) | 将每次预读窗口大小的最大值从 128KB 增加到了 512KB, 其中增加了一个统计接口(tracepoint, stat 节点等). | v3 ☐ | [PatchWork](https://lore.kernel.org/patchwork/patch/190891), [LWN](https://lwn.net/Articles/234784) |
| 2011/05/17 | WU Fengguang <wfg@mail.ustc.edu.cn> | [on-demand readahead](https://lwn.net/Articles/235164) | on-demand 预读算法 | v1 ☑ [2.6.23-rc1](https://kernelnewbies.org/Linux_2_6_23#On-demand_read-ahead) | [LWN](https://lwn.net/Articles/234784) |


### 6.2.4 Page Cache Fault
-------

```cpp
do_read_fault()
-=> do_fault_around()
```


| 时间   | 作者 | 特性  | 描述  |  是否合入主线  | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2014/04/07 | Kirill A. Shutemov <kirill.shutemov@linux.intel.com> | [mm: map few pages around fault address ifthey are in page cache](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=f1820361f83d556a7f0a9f629100f3825e594328) | 文件页读取触发 page_fault 时, do_read_fault() 处理过程中, 如果发现当前文件页有 page cache 映射, 则尝试围绕缺页异常的地址预读更多的文件页面, 减少缺页异常次数. 使用 [vm_ops->map_pages()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8c6e50b0290c4c708a3e6462729e1e9151a9a7df)(默认是 filemap_map_pages()) 来完成预读. | v3 ☑✓ 3.15-rc1 | [LORE v3,0/2](https://lore.kernel.org/all/1393530827-25450-1-git-send-email-kirill.shutemov@linux.intel.com) |
| 2014/06/04 | "Kirill A. Shutemov" <kirill.shutemov@linux.intel.com> | [mm: document do_fault_around() feature](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=1fdb412bd825998efbced3a16f6ce7e0329728cf) | TODO | v1 ☑✓ 3.16-rc1 | [LORE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=1fdb412bd825998efbced3a16f6ce7e0329728cf) |
| 2014/07/23 | Konstantin Khlebnikov <koct9i@gmail.com> | [mm: do not call do_fault_around for non-linear fault](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c118678bc79e8241f9d3434d9324c6400d72f48a) | 参见 [PROBLEM: repeated remap_file_pages on tmpfs triggers bug on process exit](https://lkml.org/lkml/2014/7/14/335) | v1 ☑✓ 3.16-rc7 | [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c118678bc79e8241f9d3434d9324c6400d72f48a) |
| 2014/5/8 | Konstantin Khlebnikov <koct9i@gmail.com> | [mm: FAULT_AROUND_ORDER patchset performance data for powerpc](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c118678bc79e8241f9d3434d9324c6400d72f48a) | Kirill A. Shutemov 在 [commit 8c6e50b029 ("mm: introduce vm_ops->map_pages()")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=1fdb412bd825998efbced3a16f6ce7e0329728cf) 尝试围绕缺页异常的地址预读更多的文件页面, 以此减少 minor page faults 的数量.<br>1. 本补丁创建基础设施, 通过 mm/Kconfig 修改 FAULT_AROUND_ORDER 的值. 这将使体系结构维护者能够基于该体系结构的性能数据来决定合适的 FAULT_AROUND_ORDER 值(默认为 4).<br>2. 列出 powerpc (平台 pseries) 的性能数字, 并对 powerpc 的 pseries 平台进行初始化. | v1 ☑✓ 3.16-rc7 | [LKML V4,0/2](https://www.lkml.org/lkml/2014/5/8/118) |


### 6.2.5 readahead support for filesystem
-------

| 时间 | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:---:|:----:|:---:|:---:|:----------:|:----:|
| 2022/05/16 | Hsin-Yi Wang <hsinyi@chromium.org> | [Implement readahead for squashfs](https://patchwork.kernel.org/project/linux-mm/cover/20220516105100.1412740-1-hsinyi@chromium.org/) | 641926 | v1 ☐☑ | [LORE v1,0/2](https://lore.kernel.org/r/20220516105100.1412740-1-hsinyi@chromium.org)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/3](https://lore.kernel.org/r/20220601103922.1338320-1-hsinyi@chromium.org)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/3](https://lore.kernel.org/r/20220606150305.1883410-1-hsinyi@chromium.org)<br>*-*-*-*-*-*-*-* <br>[LORE v7,0/4](https://lore.kernel.org/r/20220617083810.337573-1-hsinyi@chromium.org) |



### 6.2.6 More readahead Patchset
-------

| 时间 | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:---:|:----:|:---:|:---:|:----------:|:----:|
| 2022/06/07 | Alistair Popple <apopple@nvidia.com> | [mm/filemap.c: Always read one page in do_sync_mmap_readahead()](https://patchwork.kernel.org/project/linux-mm/patch/20220607083714.183788-1-apopple@nvidia.com/) | 647906 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220607083714.183788-1-apopple@nvidia.com) |


## 6.3 DirtyPage
-------


### 6.3.1 页面写回
-------


从用户角度来看, 这一节对了解 Linux 内核发展帮助不大, 可跳过不读; 但对于技术人员来说, 本节可以展现教材理论模型到工程实现的一些思考与折衷, 还有软件工程实践中由简单粗糙到复杂精细的演变过程


当进程改写了文件缓存页, 此时内存中的内容与 ** 后备存储设备 (backing device)** 的内容便处于不一致状态, 此时这种页面叫做 ** 脏页(dirty page).** 内核会把这些脏页写回到后备设备中. 这里存在一个折衷: 写得太频繁(比如每有一个脏页就写一次) 会影响吞吐量; 写得太迟 (比如积累了很多个脏页才写回) 又可能带来不一致的问题, 假设在写之前系统崩溃, 则这些数据将丢失, 此外, 太多的脏页会占据太多的可用内存. 因此, 内核采取了几种手段来写回:

1) 设一个后台门槛(background threshold), 当系统脏页数量超过这个数值, 用后台线程写回, 这是异步写回.

2) 设一个全局门槛 (global threshold), 这个数值比后台门槛高. 这是以防系统突然生成大量脏页, 写回跟不上, 此时系统将扼制(throttle) 生成脏页的进程, 让其开始同步写回.



#### 6.3.1.1 由全局的脏页门槛到每设备脏页门槛
-------

**2.6.24(2008 年 1 月发布)**


内核采取的第 2 个手段看起来很高明 - 扼制生成脏页的进程, 使其停止生成, 反而开始写回脏页, 这一进一退中, 就能把全局的脏页数量拉低到全局门槛下. 但是, 这存在几个微妙的问题:



> **1\.** 有可能这大量的脏页是要写回某个后备设备 A 的, 但被扼制的进程写的脏页则是要写回另一个后备设备 B 的. 这样, 一个不相干的设备的脏页影响到了另一个 (可能很重要的) 设备, 这是不公平的.
> **2\.** 还有一个更严重的问题出现在栈式设备上. 所谓的栈式设备 (stacked device) 是指多个物理设备组成的逻辑设备, 如 LVM 或 software RAID 设备上. 操作在这些逻辑设备上的进程只能感知到这些逻辑设备. 假设某个进程生成了大量脏页, 于是, 在逻辑设备这一层, 脏页到达门槛了, 进程被扼制并让其写回脏页, 由于进程只能感知到逻辑设备这一层, 所以它觉得脏页已经写下去了. 但是, 这些脏页分配到底下的物理设备这一层时, 可能每个物理设备都还没到达门槛, 那么在这一层, 是不会真正往下写脏页的. 于是, 这种极端局面造成了死锁: 逻辑设备这一层以为脏页写下去了; 而物理设备这一层还在等更多的脏页以到达写回的门槛.





2.6.24 引入了一个新的改进[Smarter write throttling](https://lwn.net/Articles/245600), 就是把全局的脏页门槛替换为每设备的门槛. 这样第 1 个问题自然就解决了. 第 2 个问题其实也解决了. 因为现在是每个设备一个门槛, 所以在物理设备这一层, 这个门槛是会比之前的全局门槛低很多的, 于是出现上述问题的可能性也不存在了.



那么问题来了, 每个设备的门槛怎么确定? 设备的写回能力有强有弱 (SSD 的写回速度比硬盘快多了), 一个合理的做法是根据当前设备的写回速度分配给等比例的带宽(门槛). ** 这种动态根据速度调整的想法在数学上就是[指数衰减](https://zh.wikipedia.org/wiki/%E6%8C%87%E6%95%B0%E8%A1%B0%E5%87%8F) 的理念: 某个量的下降速度和它的值成比例.** 所以, 在这个改进里, 作者引入了一个叫 "** 浮动比例 **" 的库, 它的本质就是一个 ** 根据写回速度进行指数衰减的级数 **. (这个库跟内核具体的细节无关, 感兴趣的可以研究这个代码: [[PATCH 19/23] lib: floating proportions [LWN.net]](https://lwn.net/Articles/245603). 然后, 使用这个库, 就可以 "实时地" 计算出每个设备的带宽(门槛).



#### 6.4.1.2 引入更具体扩展性的回写线程
-------

**2.6.32(2009 年 12 月发布)**


Linux 内核在脏页数量到达一定门槛时, 或者用户在命令行输入 _sync_ 命令时, 会启用后台线程来写回脏页, 线程的数量会根据写回的工作量在 2 个到 8 个之间调整. 这些写回线程是面向脏页的, 而不是面向后备设备的. 换句话说, 每个回写线程都是在认领系统全局范围内的脏页来写回, 而这些脏页是可能属于不同后备设备的, 所以回写线程不是专注于某一个设备.



不过随着时间的推移, 这种看似灵巧的方案暴露出弊端.

> **1\.** 由于每个回写线程都是可以服务所有后备设备的, 在有多个后备设备, 且回写工作量大时, 线程间的冲突就变得明显了(毕竟, 一个设备同一时间内只允许一个线程写回), 当一个设备有线程占据, 别的线程就得等, 或者去先写别的设备. 这种冲突对性能是有影响的.
> **2\.** 线程写回时, 把脏页组成一个个写回请求, 挂在设备的请求队列上, 由设备去处理. 显然, 每个设备的处理请求能力是有限的, 即队列长度是有限的. 当一个设备的队列被线程 A 占满, 新来的线程 B 就得不到请求位置了. 由于线程是负责多个设备的, 线程 B 不能在这设备上干等, 就先去忙别的, 以期这边尽快有请求空位空出来. 但如果线程 A 写回压力大, 一直占着请求队列不放, 那么 A 就及饥饿了, 时间太久就会出问题.



针对这种情况, 2.6.32 为每个后备设备引入了专属的写回线程, 参见 [Flushing out pdflush](https://lwn.net/Articles/326552), 换言之, 现在的写回线程是面向设备的. 在写回脏页时, 系统会根据其所属的后备设备, 派发给专门的线程去写回, 从而避免上述问题.





#### 6.3.1.3 动态的脏页生成扼制和写回扼制算法
-------

**3.1(2011 年 11 月发布), 3.2(2012 年 1 月发布)**


本节一开始说的写回扼制算法, 其核心就是 ** 谁污染谁治理: 生成脏页多的进程会被惩罚, 让其停止生产, 责成其进行义务劳动, 把系统脏页写回.** 在 5.1 小节里, 已经解决了这个方案的针对于后备设备门槛的一些问题. 但还存在一些别的问题.



> **1.** 写回的脏页的 ** 破碎性导致的性能问题 **. 破碎性这个词是我造的, 大概的意思是, 由于被罚的线程是同步写回脏页到后备设备上的. 这些脏页在后备设备上的分布可能是很散乱的, 这就会造成频繁的磁盘磁头移动, 对性能影响非常大. 而 Linux 存在的一个块层 (block layer, 倒数第 2 个子系统会讲) 本来就是要解决这种问题, 而现在写回机制是相当于绕过它了.
>
> **2\.** 系统根据当前可用内存状况决定一个脏页数量门槛, 一到这个门槛值就开始扼制脏页生成. 这种做法太粗野了点. 有时启动一个占内存大的程序(比如一个 kvm), 脏页门槛就会被急剧降低, 就会导致粗暴的脏页生成扼制.
>
> **3\. 从长远看, 整个系统生成的脏页数量应该与所有后备设备的写回能力相一致.** 在这个方针指导下, 对于那些过度生产脏页的进程, 给它一些限制, 扼制其生成脏页的速度. 内核于是设置了一个 ** 定点 **, 在这个定点之下, 内核对进程生成脏页的速度不做限制; 但一超过定点就开始粗暴地限制.



3.1, 3.2 版本中, 来自 Intel 中国的吴峰光博士针对上述问题, 引入了动态的脏页生成扼制和[写回扼制算法 Dynamic writeback throttling[(https://lwn.net/Articles/405076). 其主要核心就是, 不再让受罚的进程同步写回脏页了, 而是罚它睡觉; 至于脏页写回, 会委派给专门的[写回线程](https://lwn.net/Articles/326552), 这样就能利用块层的合并机制, 尽可能把磁盘上连续的脏页合并再写回, 以减少磁头的移动时间.



至于标题中的 "动态" 的概念, 主要有下:

> **1\.** 决定受罚者睡多久. 他的算法中, 动态地去估算后备设备的写回速度, 再结合当前要写的脏页量, 从而动态地去决定被罚者要睡的时间.
> **2\.** 平缓地修改扼制的门槛. 之前进程被罚的门槛会随着一个重量级进程的启动而走人骤降, 在吴峰光的算法中, 增加了对全局内存压力的评估, 从而平滑地修改这一门槛.
> **3\.** 在进程生成脏页的扼制方面, 吴峰光同样采取反馈调节的做法, 针对写回工作量和写回速度, 平缓地 (尽量) 把系统的脏页生成控制在定点附近.


### 6.3.2 dirty limits
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2005/10/06 | WU Fengguang <wfg@mail.ustc.edu.cn> | [per-zone dirty limits v3](https://lore.kernel.org/patchwork/patch/268866) | per-zone 的脏页限制.<br> 在回收期间回写单个文件页面显示了糟糕的 IO 模式, 但在 VM 有其他方法确保区域中的页面是可回收的之前, 我们不能停止这样做. 随着时间的推移, 出现了一些建议, 当在内存压力期间出现需要清理页面时, 至少要在 inode-proximity 中对页面进行写转. 但是即使这样也会中断来自刷新器的回写, 而不能保证附近的 inode-page 位于同一问题区域. 脏页面之所以会到达 LRU 列表的末尾, 部分原因在于脏限制是一个全局限制, 而大多数系统都有多个大小不同的 LRU 列表. 多个节点有多个区域, 有多个文件列表, 但与此同时, 除了在遇到脏页时回收它们之外, 没有任何东西可以平衡这些列表之间的脏页.  | v3 ☐ | [PatchWork](https://lore.kernel.org/patchwork/patch/268866) |


### 6.3.4 其他
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/07/13 | Jan Kara <jack@suse.cz> | [writeback: Fix bandwidth estimates](https://patchwork.kernel.org/project/linux-mm/cover/20210712165811.13163-1-jack@suse.cz) | NA | v2 ☐ | [PatchWork 0/5,v2](https://patchwork.kernel.org/project/linux-mm/cover/20210712165811.13163-1-jack@suse.cz) |
| 2011/08/10 | Mel Gorman <mgorman@suse.de> | [Reduce filesystem writeback from page reclaim v3](https://lore.kernel.org/all/1312973240-32576-1-git-send-email-mgorman@suse.de) | 1312973240-32576-1-git-send-email-mgorman@suse.de | v3 ☐☑✓ | [LORE v3,0/7](https://lore.kernel.org/all/1312973240-32576-1-git-send-email-mgorman@suse.de) |



# 7 大内存页支持
-------


我们知道现代操作系统都是以页面 (page) 的方式管理内存的. 一开始, 页面的大小就是 4K, 在那个时代, 这是一个相当大的数目了. 所以众多操作系统, 包括 Linux , 深深植根其中的就是一个页面是 4K 大小这种认知, 尽管现代的 CPU 已经支持更大尺寸的页面(X86 体系能支持 2MB, 1GB).



我们知道虚拟地址的翻译要经过页表的翻译, CPU 为了支持快速的翻译操作, 引入了 TLB 的概念, 它本质就是一个页表翻译地址结果的缓存, 每次页表翻译后的结果会缓存其中, 下一次翻译时会优先查看 TLB, 如果存在, 则称为 TLB hit; 否则称为 TLB miss, 就要从访问内存, 从页表中翻译. 由于这是一个 CPU 内机构, 决定了它的尺寸是有限的, 好在由于程序的局部性原理, TLB 中缓存的结果很大可能又会在近期使用.



但是, 过去几十年, 物理内存的大小翻了几番, 但 TLB 空间依然局限, 4KB 大小的页面就显得捉襟见肘了. 当运行内存需求量大的程序时, 这样就存在更大的机率出现 TLB miss, 从而需要访问内存进入页表翻译. 此外, 访问更多的内存, 意味着更多的缺页中断. 这两方面, 都对程序性能有着显著的影响.


LWN 上 Mel 写的关于 Huge Page 的连载.

[Huge pages part 1 (Introduction)](https://lwn.net/Articles/374424)

[Huge pages part 2: Interfaces](https://lwn.net/Articles/375096)

[Huge pages part 3: Administration](https://lwn.net/Articles/376606)

[Huge pages part 4: benchmarking with huge pages](https://lwn.net/Articles/378641)

[Huge pages part 5: A deeper look at TLBs and costs](https://lwn.net/Articles/379748)

[[v2,0/4] x86, mm: Handle large PAT bit in pud/pmd interfaces](https://lore.kernel.org/patchwork/patch/579540)

[[v12,0/10] Support Write-Through mapping on x86](https://lore.kernel.org/patchwork/patch/566256/)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/01/26 | Matthew Wilcox <willy@infradead.org><br>Dave Jiang <dave.jiang@intel.com> | [Support for transparent PUD pages](https://lwn.net/Articles/669232) | NA | ☑ 4.11-rc1 | [PatchWork RFC](https://patchwork.kernel.org/project/linux-nvdimm/patch/148545059381.17912.8602162635537598445.stgit@djiang5-desk3.ch.intel.com) |
| 2021/07/30 | Hugh Dickins <hughd@google.com> | [tmpfs: HUGEPAGE and MEM_LOCK fcntls and memfds](https://lwn.net/Articles/669232) | NA  | ☑ 4.11-rc1 | [PatchWork 00/16](https://patchwork.kernel.org/project/linux-mm/cover/2862852d-badd-7486-3a8e-c5ea9666d6fb@google.com) |




## 7.1 标准大页 HugeTLB 支持
-------


### 7.1.1 引入 HugeTLB
-------


如果能使用更大的页面, 则能很好地解决上述问题. 试想如果使用 2MB 的页(一个页相当于 512 个连续的 4KB 页面), 则所需的 TLB 表项由原来的 512 个变成 1 个, 这将大大提高 TLB hit 的机率; 缺页中断也由原来的 512 次变为 1 次, 这对性能的提升是不言而喻的.


然而, 前面也说了 Linux 对 4KB 大小的页面认知是根植其中的, 想要支持更大的页面, 需要对非常多的核心的代码进行大改动, 这是不现实的. 于是, 为了支持大页面, 有了一个所谓 HugeTLB 的实现.


它的实现是在系统启动时, 按照用户指定需求的最大大页的大小和个数, 预留足够的物理内存空间. 用户在程序中可以使用 **mmap()** 系统调用或共享内存的方式映射和访问这些大页, 参考官方文档: [HugeTLB Pages](https://www.kernel.org/doc/html/latest/admin-guide/mm/hugetlbpage.html).

当然, 现在也存在一些用户态工具, 可以帮助用户更便捷地使用. 具体可参考此文章: [Huge pages part 2: Interfaces [LWN.net]](https://lwn.net/Articles/375096).

同 tmpfs 类似, 基于 hugetlbfs 创建文件后, 再进行 mmap() 映射, 就可以访问这些 huge page 了, 使用方法可参考内核源码"tools/testing/selftests/vm" 中的示例.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2002/09/15 | Rohit Seth/Andrew Morton <akpm@digeo.com> | [hugetlb pages](https://github.com/gatieme/linux-history/commit/c9d3808fc28fc873dcf0dc95315f644997fde1d0) | 实现 ia32 的 HugeTLB. 由 CONFIG_HUGETLB_PAGE 控制. | v1 ☑ 2.5.36 | [HISTORY COMMIT](https://github.com/gatieme/linux-history/commit/c9d3808fc28fc873dcf0dc95315f644997fde1d0) |
| 2002/09/15 | Bill Irwin/Andrew Morton <akpm@digeo.com> | [hugetlbfs file system](https://github.com/gatieme/linux-history/commit/9f3336ab7c42d631f5ed50d73e1eea7bd9268892) | 为 hugetlb page 实现了一个小巧的  ram-backed 的文件系统. 通过 CONFIG_HUGETLBFS 控制. | v1 ☑ 2.5.46 | [HISTORY COMMIT](https://github.com/gatieme/linux-history/commit/9f3336ab7c42d631f5ed50d73e1eea7bd9268892) |
| 2004/04/11 | William Lee Irwin III <wli@holomorphy.com> | [hugetlb consolidation](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?id=c8b976af1af10de3d92968bf7d4bd5415e8a3778) | 整合了各个架构下 hugetlb 实现中的冗余代码, 实现了统一的 hugetlb 框架. 注意这里将整个 HugeTLB 的代码都用 CONFIG_HUGETLBFS 选项来控制开启. | v1 ☑ 2.6.6-rc1 | [HISTORY COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/history/history.git/commit/?id=c8b976af1af10de3d92968bf7d4bd5415e8a3778) |

至此 [v2.6.6](https://elixir.bootlin.com/linux/v2.6.6/source) 版本, HugeTLB 的功能和框架已经趋于完善.

HugeTLB 的功能, 涉及到两个模块: HugeTLB 和 HugeTLBFS.

HugeTLB 相当于是 HugeTLB 页面管理者, 页面的分配及释放, 都由此模块负责. 由 [CONFIG_HUGETLB_PAGE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e3390f67a7267daa227380b6f1bbf13c7ddd4aff) 标记.

HugeTLBFS 则用于向用户提供一套基于文件系统的巨页使用界面, 其下层功能的实现, 则依赖于 HugeTLB. 由 [CONFIG_HUGETLBFS] 控制.

首先是 HugeTLB 模块:

1.  通过内核启动参数 [`hugepages=`](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L115) 来指定预留的 HugeTLB 的数量.

2.  然后内核启动时 [`hugetlb_init()`](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L87) 中会通过分配 [`alloc_fresh_huge_page()`](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L45).

3.  所有分配的 HugeTLB 页面会由 [`enqueue_huge_page(struct page *page)`](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L20) 添加到 [`hugepage_freelists`](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L17) 链表中.

4.  提供了 `/proc/sys/vm/nr_hugepages` sysctl 接口 [查看](https://elixir.bootlin.com/linux/v2.6.6/source/kernel/sysctl.c#L734) 和[配置](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L183)当前内核中 HugeTLB 大页数目 [max_huge_pages](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L16). 在配置的过程中内核会通过 [`alloc_fresh_huge_page()`](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L159) 和 [`enqueue_huge_page(page)`](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L163) 扩展页面数, 已经通过 [`try_to_free_low()`](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L132) 和 [`update_and_free_page()`](https://elixir.bootlin.com/linux/v2.6.6/source/mm/hugetlb.c#L176) 释放那些不需要的页面. 从而动态地提供大页的数量.

其次看 HugeTLBFS 模块:

### 7.1.2 HugeTLB Pool
-------

commit [b45b5bd65f66 ("hugepage: Strict page reservation for hugepage inodes")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b45b5bd65f668a665db40d093e4e1fe563533608)


commit [a43a8c39bbb4 ("tightening hugetlb strict accounting")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a43a8c39bbb493c9e93f6764b350de2e33e18e92)

随后 v2.6.24 HugeTLB 又引入了 dynamic pool 和 overcommit.


由于匿名映射 MAP_PRIVATE 不使用保留的大面内存, 因此分配可能由于 HugeTLB 池太小而失败. commit [7893d1d505d5 ("hugetlb: Try to grow hugetlb pool for MAP_PRIVATE mappings")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7893d1d505d59db9d4f35165c8b6d3c6dff40a32) 引入动态扩展 HugeTLB 页面池的的机制.<br>1. 尝试通过 `alloc_buddy_huge_page()` 从 buddy 分配器中获取一个剩余的大页.<br>2. 通过 surplus_huge_pages 和 surplus_huge_pages_node 来记录动态扩展的页面.<br>3. 那么这种情况下如果通过 nr_hugepages sysctl 动态修改 HugeTLB 页面的数量, 则通过 `set_max_huge_pages()-=>adjust_pool_surplus()` 来完成动态扩展.

commit [e4e574b767ba ("hugetlb: Try to grow hugetlb pool for MAP_SHARED mappings")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e4e574b767ba63101cfda2b42d72f38546319297)


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/10/16 | Adam Litke <agl@us.ibm.com> | [hugetlb_dynamic_pool](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=af767cbdd78f293485c294113885d95e7f1da123) | 引入 dynamic_pool. | v1 ☑ [2.6.24-rc1](https://lwn.net/Articles/255649) | [HISTORY COMMIT 0/6](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=af767cbdd78f293485c294113885d95e7f1da123) |
| 2007/12/17 | Nishanth Aravamudan <nacc@us.ibm.com> | [hugetlb: introduce nr_overcommit_hugepages sysctl](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=d5dbac87b4343d98ae509fb787efb77f8ddc484b) | 1. 移除了 [Revert"hugetlb: Add hugetlb_dynamic_pool sysctl"](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=368d2c6358c3c62b3820a8a73f9fe9c8b540cdea)<br>2. 引入了 [nr_overcommit_hugepages](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d1c3fb1f8f29c41b0d098d7cfb3c32939043631f) sysctl.<br>3. [Documentation: update hugetlb information](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=d5dbac87b4343d98ae509fb787efb77f8ddc484b) 更新了 hugetlb 的文档, 至此 hugetlb 的功能已经趋于完善. | v1 ☑ 2.6.24-rc6 | [HISTORY COMMIT](https://github.com/gatieme/linux-history/commit/d5dbac87b4343d98ae509fb787efb77f8ddc484b) |


### 7.1.2 SHM_HUGETLB
-------


通过 shmget()/shmat() 也可以使用 hugetlb, 调用 shmget() 申请共享内存时要加上 SHM_HUGETLB 标志即可. 具体使用可以参考 selftests 用例中的 `hugepage-shm.c`.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2002/10/30 | Bill Irwin/Andrew Morton <akpm@digeo.com> | [hugetlbfs backing for SYSV shared memory](https://github.com/gatieme/linux-history/commit/bba2dd58c14a371b1062e585a280059fc6e9364f) | 实现 SHM_HUGETLB, 为进程的 SHEM 支持 hugetlb .<br> 对于 hugetlb 接口的用户来说, 最常见的请求之一就是使用 shm 的数据库. 这个补丁导出的功能基本上相当于 tmpfs, 将调用序列添加到 `ipc/shm.c`, 并在 `fs/hugetlbfs/inode.c` 中散列出一个小的支持函数, 这样如果用户空间将一个标志传递给 shmget(), shm 段可能是 hugetlbpage 支持的. | v1 ☑ 2.5.46 | [HISTORY COMMIT](https://github.com/gatieme/linux-history/commit/bba2dd58c14a371b1062e585a280059fc6e9364f) |
| 2002/10/30 | Rohit Seth | [hugetlbpage documentation update](https://github.com/gatieme/linux-history/commit/a2f6cc8614e920b7b86782ac8391a15165631157) | 更新 hugetlb 的文档, 同时增加了 SHM_HUGETLB 的测试用例 `Documentation/vm/hugetlbpage.txt`, 该用例随后被重命名为 [`Documentation/vm/hugepage-shm.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=70bace8c1edefa700c7f7af522c5374ef63860ae), 最终被[移动到了 selftests 路径下](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f0f57b2b1488). | v1 ☑ 2.5.64 | [HISTORY COMMIT](https://github.com/gatieme/linux-history/commit/a2f6cc8614e920b7b86782ac8391a15165631157) |


### 7.1.3 MAP_HUGETLB
-------


但是每次使用 huge page 都需要将 hugetlbfs 挂载到文件系统到某个节点上, 部署起来很不方便, 我们只想要点匿名页面, 要搞的那么麻烦吗?
于是 2.6.32 内核通过支持 MAP_HUGETLB 方式来使用内存, 避免了烦琐的 mount 操作, 对用户更友好.

至此, 通过 mmap(), 调用时指定 MAP_HUGETLB 标志就可以使用 hugetlb 了, 方便了很多.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2009/08/14 | Eric B Munson <ebmunson@us.ibm.com> | [Add pseudo-anonymous huge page mappings V3](https://lore.kernel.org/linux-man/cover.1250258125.git.ebmunson@us.ibm.com) | 这个补丁集为 mmap 添加了一个标志 [MAP_HUGETLB](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4e52780d41a741fb4861ae1df2413dd816ec11b1), 允许用户请求使用大页面支持映射.<br> 这个映射借用[大页 shm 代码的功能](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6bfde05bf5c9682e255c6a2c669dc80f91af6296), 在内核内部挂载上创建一个文件, 并使用它近似于匿名映射 MAP_ANONYMOUS.<br>1. MAP_HUGETLB 标志是 MAP_ANONYMOUS 的修饰符, 如果两个标志都没有被预设, 它就不能工作.<br>2. 一个新的标志是必要的, 因为当前除了在 hugetlbfs 挂载上创建一个文件外, 没有其他方法可以映射到到大页.<br>3. 对于用户空间, 这个映射的行为就像匿名映射, 因为这个文件在内核之外是不可访问的. | RFC ☐ | [PatchWork 0/3](https://lore.kernel.org/linux-man/cover.1250258125.git.ebmunson@us.ibm.com) |


### 7.1.4 More huge page sizes
-------

在 2.6.25 版本的时候, 通过 commit [4ec161cf73bc ("Add hugepagesz boot-time parameter")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4ec161cf73bc0b4e5c36843638ef9171896fc0b9) 添加了一个启动参数 hugepagesz, 可以指定预留的 HugeTLB 大页的 pagesize.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/05/26 | Andi Kleen <ak@suse.de>/Nick Piggin <npiggin@suse.de> | [multi size, giant hugetlb support, 1GB for x86, 16GB for powerpc](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=0d9ea75443dc7e37843e656b8ebc947a6d16d618) | 多种 hugepage sizes 的 HugeTLB 的支持. 引入了 [struct hstate](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a5516438959d90b071ff0a484ce4f3f523dc3152) 来管理不同 page size 的 HugeTLB. 每个 pagesize 的页面由 [hstate 自己的链表](https://elixir.bootlin.com/linux/v2.6.27/source/mm/hugetlb.c#L384)管理.<br> 引入了 [PUD 级别](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ceb868796181dc95ea01a110e123afd391639873) 和 [巨页(> MAX_ORDER)](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aa888a74977a8f2120ae9332376e179c39a6b07d) 的 Huge Page. 随后对 [x86_64](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=39c11e6c05b7fedbf7ed4df3908b25f622d56204) 以及 [POWERPC](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0d9ea75443dc7e37843e656b8ebc947a6d16d618) 等架构做了支持. POWERPC 更是支持了 [16GB 级别的巨页](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ec4b2c0c8312d1118c2acd00c89988ecf955d5cc). 可以通过启动参数 hugepagesz 参数指定大页的大小. | v1 ☑ 2.6.27-rc1 | [LKML 00/23](https://lore.kernel.org/all/20080525142317.965503000@nick.local0.net) |


huge page 最开始只支持 PMD 级别 (基础页 4K, 则 PMD 级别为 2MB) 的大页, 自 3.8 版本加入这个 [commit 42d7395feb56 ("mm: support more pagesizes for MAP_HUGETLB/SHM_HUGETLB")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=42d7395feb56f0655cd8b68e06fc6063823449f8) 之后, 利用 shmget()/mmap() 的 flag 参数中未使用的 bits, 可以支持其他的 huge page 大小(比如 1GB).

进一步的, ARM64 MMU 支持一个连续位(Contiguous bit), 该位表示 TTE 是可缓存在单个 TLB 条目中的一组连续条目之一. 通过对该位的支持可以增加新的中间大页的大小. 可用的巨大页面大小取决于基本页面大小 PAGE_SIZE. 参见 [arm64: Add support for PTE contiguous bit.](https://lore.kernel.org/lkml/1450380686-20911-1-git-send-email-dwoods@ezchip.com)


[arm64: hugetlb: partial revert of 66b3923a1a0f](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ff7925848b50050732ac0401e0acf27e8b241d7b)

[Revert"arm64: hugetlb: partial revert of 66b3923a1a0f"](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ab2e1b89230fa80328262c91d2d0a539a2790d6f)

[arm64: Re-enable support for contiguous hugepages](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5cd028b9d90403bf24c8bf7915ed61c7a9bfce6c)

在不使用连续页面的情况下, 大页面大小如下所示:

```cpp
-----------------------------
| Page Size |  PMD  |  PUD  |
-----------------------------
|     4K    |   2M  |   1G  |
|    16K    |  32M  |       |
|    64K    | 512M  |       |
-----------------------------
```

对于 4KB 的 PAGE_SIZE, 使用 Contiguous bit 的相邻位将 16 页的集合分组,
对于 64KB 的 PAGE_SIZE, 它将 32 页的集合分组. 这将在每种情况下启用两个新的巨大页面大小, 因此完整的可用大小集如下所示.
如果使用 16KB 的 PAGE_SIZE, 则连续位在 PTE 级别将 128 页分组, 在 PMD 级别将 32 页分组.

```cpp
---------------------------------------------------
| Page Size | CONT PTE |  PMD  | CONT PMD |  PUD  |
---------------------------------------------------
|     4K    |   64K    |   2M  |    32M   |   1G  |
|    16K    |    2M    |  32M  |     1G   |       |
|    64K    |    2M    | 512M  |    16G   |       |
---------------------------------------------------
```

如果基本页面大小设置为 64KB, 则默认情况下会启用 2MB 页面. 在将来, 4KB 和 64KB 的页面都可以使用 2MB 作为默认的巨大页面大小.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/12/05 | Andi Kleen <ak@linux.intel.com> | [mm: support more pagesizes for MAP_HUGETLB/SHM_HUGETLB](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=42d7395feb56f0655cd8b68e06fc6063823449f8) | Free page reporting 只支持伙伴系统中的页面, 它不能报告为 hugetlbfs 预留的页面. 这个补丁在 hugetlb 的空闲列表中增加了对报告巨大页的支持, 它可以被 virtio_balloon 驱动程序用于内存过载和预归零空闲页, 以加速内存填充和页面错误处理. | v7 ☑ 3.8-rc1 | [LWN v7](https://lwn.net/Articles/533650), [LORE v7](https://lore.kernel.org/lkml/1352157848-29473-2-git-send-email-andi@firstfloor.org), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=42d7395feb56f0655cd8b68e06fc6063823449f8) |
| 2015/12/17 | David Woods <dwoods@ezchip.com> | [arm64: Add support for PTE contiguous bit.](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=66b3923a1a0f77a563b43f43f6ad091354abbfe9) | 引入 hugetlb cgroup | v5 ☑ 4.5-rc1 | [LKML v5](https://lore.kernel.org/lkml/1450380686-20911-1-git-send-email-dwoods@ezchip.com) |
| 2017/08/22 | Punit Agrawal <punit.agrawal@arm.com> | [arm64: Enable contiguous pte hugepage support](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=828f193dd62a40ade5ea8b24cb8b0a22c30df673) | 20170822104249.2189-1-punit.agrawal@arm.com | v7 ☑✓ 4.14-rc1 | [LORE v7,0/9](https://lore.kernel.org/all/20170822104249.2189-1-punit.agrawal@arm.com) |
| 2022/03/23 | Steve Capper <steve.capper@arm.com> | [tlb: hugetlb: Add arm64 contiguous hint awareness](https://patchwork.kernel.org/project/linux-mm/patch/20220323165218.35499-1-steve.capper@arm.com/) | 该补丁实现了 per-arch 的 tlb_remove_huge_tlb_entry() 能力, 并提供了一个覆盖连续提示大小的 arm64 实现. 在更新 mmu_gather 结构时, tlb_remove_huge_tlb_entry 要支持不同 level HugePage 支持. 但是当前只支持了 PMD_SIZE 和 PUD_SIZE. 而 arm64 上还有两个巨页面需要覆盖: CONT_PTE_SIZE 和 CONT_PMD_SIZE, 当最终用户试图使用大页时, 由于相关 tlb 未正确更新 tlb 结构, 可能会出现 VM_BUG_ON. | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/all/20220323165218.35499-1-steve.capper@arm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20220330112543.863-1-steve.capper@arm.com) |


### 7.1.5 HugeTLB CGROUP
-------

参见内核文档 [HugeTLB Controller](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/hugetlb.html).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2012/07/31 | Aneesh Kumar K.V <aneesh.kumar@linux.vnet.ibm.com> | [hugetlb: Add HugeTLB controller to control HugeTLB allocation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=75754681fe79b84dde1048470a44eeb64192fad6) | 引入 hugetlb cgroup | v9 ☑ 3.6-rc1 | [LKML v1,0/9](https://lkml.org/lkml/2012/2/20/127)<br>*-*-*-*-*-*-*-* <br>[LKML v8,00/16](https://lkml.org/lkml/2012/6/9/22)[LKML v9,00/15](https://lkml.org/lkml/2012/6/13/161), [关键 COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2bc64a2046975410505bb119bba32705892b9255) |
| 2019/12/16 | Giuseppe Scrivano <gscrivan@redhat.com> | [mm: hugetlb controller for cgroups v2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=faced7e0806cf44095a2833ad53ff59c39e6748d) | cgroup v2 支持 hugetlb. | v1 ☑ 5.6-rc1 | [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=faced7e0806cf44095a2833ad53ff59c39e6748d) |
| 2021/09/29 | Baolin Wang <baolin.wang@linux.alibaba.com> | [Support hugetlb charge moving at task migration](https://lore.kernel.org/linux-mm/cover.1632843268.git.baolin.wang@linux.alibaba.com) | 当前, 在 hugetlb cgroup 中, 任务的统计信息不会在任务迁移时转移到新的 hugetlb cgroup 中, 这个补丁集增加了 hugetlb cgroup 计费统计迁移. | v1 ☐ | [LORE 0/2](https://lore.kernel.org/linux-mm/cover.1632843268.git.baolin.wang@linux.alibaba.com) |

当前任务试图分配更多的 hugetlb 内存, 而系统已经没有可用的预留空间时, mmap/shmget 会失败. 参见 [Hugetlbfs Reservation](https://www.kernel.org/doc/html/latest/vm/hugetlbfs_reserv.html). 然而, 如果一个任务试图分配的 hugetlb 内存只超过了它的 hugetlb_cgroup 限制, 内核却能 mmap/shmget 成功, 只有在触发 page fault 时, 系统会对该进程发送 SIGBUS 处理. 部分内核开发者对内核这种处理行为表示了不满. 更合理的操作是 hugetlb_cgroup 限制的任务在 mmap/shmget 时就该失败, 而不是当它们尝试使用多余的内存时才触发 SIGBUS.

产生这样问题的根本原因是, hugetlb_cgroup 的统计是在 page fault 的时候(hugetlb memory *fault* time), 而不是在预留的时候(*reservation* time.). 因此, 检查 hugetlb_cgroup 限制只能在 page fault 时进行, 如果发现任务超出限制了则会被 SIGBUS 处理.

建议解决方案是:

1.  一个名为 `hugetlb.xMB.reservation_[limit|usage]_in_bytes` 的新页计数器. 这个计数器的语义与 hugetlb.xMB 稍有不同.

2.  usage_in_bytes 跟踪所有已经发生 page fault 并被实际使用的 hugetlb 内存, reservation_usage_in_bytes 则跟踪所有 reserved 的 hugetlb 内存.

3.  如果一个任务试图保留超过 limit_in_bytes 允许的内存, 内核将允许它这样做. 但是, 如果一个任务试图预留比 reservation_limit_in_bytes 更多的内存, 内核将无法通过这个预留.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/02/11 | Mina Almasry <almasrymina@google.com> | [hugetlb_cgroup: Add hugetlb_cgroup reservation limits](https://lore.kernel.org/linux-mm/cover.1632843268.git.baolin.wang@linux.alibaba.com) | 当前, 在 hugetlb cgroup 中, 任务的统计信息不会在任务迁移时转移到新的 hugetlb cgroup 中, 这个补丁集增加了 hugetlb cgroup 计费统计迁移. | v1 ☐ | 2019/08/08 [PatchWork RFC](https://patchwork.kernel.org/project/linux-mm/patch/20190808194002.226688-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>2019/08/08 [PatchWork RFC,v2,0/5](https://patchwork.kernel.org/project/linux-mm/cover/20190808231340.53601-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v3,0/6](https://patchwork.kernel.org/project/linux-mm/cover/20190826233240.11524-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v5,0/7](https://patchwork.kernel.org/project/linux-mm/cover/20190919222421.27408-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v6,1/9](https://patchwork.kernel.org/project/linux-mm/patch/20191013003024.215429-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v12,1/9](https://patchwork.kernel.org/project/linux-mm/patch/20200211213128.73302-1-almasrymina@google.com) |


### 7.1.6 HugeTLB Migration
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2010/08/10 | Naoya Horiguchi <n-horiguchi@ah.jp.nec.com> | [Hugepage migration](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=86cdb465cf3a9d81058b517af05074157fa9dcdd) | 实现 hugepage 迁移.<br>1. 将 hugepage 迁移函数与原始迁移代码分开, 这是为了避免复杂性.<br>2. 在当前版本中, 定义了一些高级迁移例程来处理 hugepage, 但同时一些基础的辅助函数与原始迁移代码共享, 以避免增加重复.<br>HWPOISION 的部分参见 [HWPOISON for hugepage](https://lwn.net/Articles/387568). | v2 ☑ 2.6.37 | [LORE v2,0/9](https://lore.kernel.org/lkml/1281432464-14833-1-git-send-email-n-horiguchi@ah.jp.nec.com), [关键 COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=290408d4a25002f099efeee7b6a5778d431154d6) |
| 2013/09/11 | Michal Hocko <mhocko@suse.com> | [extend hugepage migration](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=86cdb465cf3a9d81058b517af05074157fa9dcdd) | 一步扩展 Hugepage 迁移.<br> HugePage 迁移现在仅适用于软脱机 soft offlining(将半损坏页面上的数据移动到另一个页面以保存数据). 但是它对页面迁移的其他用户也很有用, 所以这个补丁集试图扩展一些这样的用户以支持 hugepage.<br> 这个补丁集并没有在内存压缩中扩展页面迁移, 因为我认为内存压缩的用户主要希望通过安排原始页面来构建 thp, 但 hugepage 迁移并没有帮助.<br> 页面迁移的另一个用户 CMA 可以从 hugepage 迁移中获益, 但现在还未支持它.<br> 目前还没有启用 1GB Hugepage 的 Hugepage 迁移, 因为作者不确定 1GB Hugepage 的用户是否真的需要它. | v5 ☑ 3.12-rc1 | [LORE RFC,0/9](https://lore.kernel.org/lkml/1361475708-25991-1-git-send-email-n-horiguchi@ah.jp.nec.coms)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/8](https://lore.kernel.org/lkml/1374183272-10153-1-git-send-email-n-horiguchi@ah.jp.nec.com), [LKML v3,0/8](https://lkml.org/lkml/2013/7/18/542)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/8](https://lore.kernel.org/lkml/1374728103-17468-1-git-send-email-n-horiguchi@ah.jp.nec.com)<br>*-*-*-*-*-*-*-* <br>[LKML v5,0/8](https://lkml.org/lkml/2013/8/9/21)<br>*-*-*-*-*-*-*-* <br>[COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=86cdb465cf3a9d81058b517af05074157fa9dcdd) |
| 2018/09/04 | "Aneesh Kumar K.V" <aneesh.kumar@linux.ibm.com> | [mm/hugetlb: filter out hugetlb pages if HUGEPAGE migration is not](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=464c7ffbcb164b2e5cebfa406b7fc6cdb7945344) | TODO | v1 ☐☑✓ 4.19-rc3 | [LORE](http://lkml.kernel.org/r/20180824063314.21981-1-aneesh.kumar@linux.ibm.com), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=464c7ffbcb164b2e5cebfa406b7fc6cdb7945344) |


### 7.1.7 HugeTLB reserve & allocations
-------


#### 7.1.7.1 分配 HugeTLB 的常规路径
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2019/10/07 | Michal Hocko <mhocko@kernel.org> | [mm, hugetlb: allow hugepage allocations to excessively reclaim](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3f36d8669457605910cb7a40089b485949569c41) | NA | v1 ☑✓ 5.4-rc4 | [LORE](https://lore.kernel.org/all/20191007075548.12456-1-mhocko@kernel.org) |


#### 7.1.7.2 HugeTLB from ZONE_MOVABLE
-------

[commit 396faf0303d2 ("Allow huge page allocations to use GFP_HIGH_MOVABLE")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=396faf0303d273219db5d7eb4a2879ad977ed185) 允许在 ZONE_MOVABLE 上进行 hugetlb 的分配. HugeTLB 的页面通常是不能移动的, 所以不会从 ZONE_MOVABLE 中分配. 然而, 由于 ZONE_MOVABLE 区总是有可以迁移或回收的页面, 因此即使系统已经运行了很长时间, 它也有足够的连续内存可以用来满足大页的分配. 因此这允许管理员在运行时根据 ZONE_MOVABLE 的大小调整巨大页面池的大小.

这个补丁添加了一个名为 [hugepages_treat_as_movable](https://elixir.bootlin.com/linux/v2.6.23/source/kernel/sysctl.c#L895) 的新 sysctl. 使能 (该接口写入 1) 后将来对这个 HugeTLB 内存池的分配将从 ZONE_MOVABLE 区域分配. 这是通过将 HugeTLB 的页面分配 flag 从默认的 GFP_HIGHUSER 修改为 GFP_HIGHUSER_MOVABLE 来完成的. 尽管大页是不可移动的, 但我们不会引入额外的外部内存碎片, 因为大页正是我们所关心的最大的连续内存块分配需求.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/03/01 | Mel Gorman <mel@csn.ul.ie> | [Allow huge page allocations to use GFP_HIGH_MOVABLE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=396faf0303d273219db5d7eb4a2879ad977ed185) | [Create optional ZONE_MOVABLE to partition memory between movable and non-movable pages v2](https://lore.kernel.org/patchwork/patch/75221) 的其中一个补丁, 提供了一个 [sysctl hugepages_treat_as_movable](https://elixir.bootlin.com/linux/v2.6.23/source/mm/hugetlb.c#L31), 允许在 ZONE_MOVABLE 上进行 hugetlb 的分配. 这意味着, 如果页面没有被 mlock 锁定, 那么在系统的生命周期内, 可以将 hugetlb 的页面池调整为 ZONE_MOVABLE 的大小(huge page pool 的尺寸包含了 ZONE_MOVABLE 的尺寸). 尽管大页是不可移动的, 但是内核开发者们认为这不会引入额外的外部内存碎片, 因为大页不正是我们所关心的最大的连续块的分配请求么. | v2 ☑ 2.6.23-rc1 | [PatchWork v2](https://lore.kernel.org/lkml/20070301100802.30048.45045.sendpatchset@skynet.skynet.ie/), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=396faf0303d273219db5d7eb4a2879ad977ed185) |

hugepages_treat_as_movable 的目的是减少内存碎片, 而 hugetlb 页面的寿命一般都很长, 且范围足够大, 因此不会造成碎片,  所以在当时使用这个区域是可以接受的. 但是随着内核不断的发展情况发生了变化, 这个 ZONE_MOVABLE 区域的主要目的变成了保证可迁移性, 从而能够很好的支持虚拟机内存的热插拔. 如果我们允许不可迁移的 hugetlb 页面在 ZONE_MOVABLE 内存中, 热插拔失败的机会会很高. 因此不太合适用一个单独的 sysctl  hugepages_treat_as_movable 来控制 HugeTLB 从 ZONE_MOVABLE 中分配. 更合理的方式应该是去看当前是否支持 [HugeTLB 的迁移](). 内核引入了 hugepage_migration_support(ed) 函数来检查这个功能.

在 [extend hugepage migration](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=86cdb465cf3a9d81058b517af05074157fa9dcdd) 之时, 内核完成了支持 HugeTLB 页面的迁移大部分工作, 已经[准备好了移除 hugepages_treat_as_movable](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=86cdb465cf3a9d81058b517af05074157fa9dcdd). 因此后面直接[移除了 hugepages_treat_as_movable](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d6cb41cc44c63492702281b1d329955ca767d399), 只要内核支持 hugepage_migration_support(ed) 允许对 HugeTLB 的页面进行迁移, 就可以支持从 ZONE_MOVABLE 中迁移.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/01/31 | Michal Hocko <mhocko@suse.com> | [mm, hugetlb: remove hugepages_treat_as_movable sysctl](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=944d9fec8d7aee3f2e16573e9b6a16634b33f403) | 移除 hugepages_treat_as_movable, 不再允许 HugeTLB 从 ZONE_MOVABLE 中分配. 而是[使用 hugepage_migration_support(ed) 来做控制](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=83467efbdb7948146581a56cbd683a22a0684bbb). | RFC ☑ 4.16-rc1 | [LORE RFC](https://lore.kernel.org/all/20171003072619.8654-1-mhocko@kernel.org), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d6cb41cc44c63492702281b1d329955ca767d399) |


但是, HugeTLB 的分配仍然有两个严重的问题:

1. 分配不是 NUMA 感知的. 在 NUMA 机器上, 内核在节点之间平均分配引导时分配的大量页面. 例如, 假设您有一个四节点 NUMA 机器, 并希望在引导时分配四个 1G 的巨大页面. 内核将为每个节点分配一个巨大的页面. 另一方面, 有些用户希望能够指定应该从哪个 NUMA 节点分配巨大的页面. 这样他们就可以在特定的 NUMA 节点上放置虚拟机.

2. 在启动时预留的大页不能被释放. 只有在运行时分配的常规大页面不会有这些问题. 这是因为在 sysfs 中用于运行时分配的 HugeTLB 接口支持 NUMA, 运行时分配的页面可以通过好友分配器释放.

#### 7.1.7.3 HugeTLB CMA(支持运行时分配大页)
-------

HugeTLB 的分配和使用多数情况下是静态的. x86_64 支持 2M 和 1G 的巨页, 但是只有 2M 的巨页可以在运行时分配和释放. 如果想要分配 1G 的巨大页面, 在系统启动阶段通过命令行参数 `hugepagesz =` 和 `hugepages =` 来预留页面完成. 这是因为当前实现下, HugeTLB 子系统在运行时使用伙伴系统来分配器分配大页. 这意味着运行时的大页的大小被限制在 MAX_ORDER order. 对于支持巨页(即 gigantic pages, 页面大小大于 MAX_ORDER) 的对象, 这意味着不能在运行时分配这些页面.


为了克服这个问题, linux 内核在 v3.16 [hugetlb: add support gigantic page allocation at runtime](https://lore.kernel.org/lkml/1397152725-20990-1-git-send-email-lcapitulino@redhat.com) 完成了[在运行时分配 HugeTLB 巨页的支持](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=944d9fec8d7aee3f2e16573e9b6a16634b33f403). 它通过 CMA 而不是伙伴分配器分配巨页来实现这一点. 内核中[把 order 大于 MAX_ORDER 的称为巨页](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bae7f4ae14d47008a11b4358b167cb0ae186c06a), 通过 [hstate_is_gigantic()](https://elixir.bootlin.com/linux/v3.16/source/include/linux/hugetlb.h#L347) 来判定待分配的请求是不是巨页. 如果是大页的请求则通过 [alloc_fresh_huge_page()](https://elixir.bootlin.com/linux/v3.16/source/mm/hugetlb.c#L1652) 分配, 如果是巨页请求则通过 [alloc_fresh_gigantic_page()](https://elixir.bootlin.com/linux/v3.16/source/mm/hugetlb.c#L1650) 扫描[节点中的所有区域](https://elixir.bootlin.com/linux/v3.16/source/mm/hugetlb.c#L752), 以查找足够大的连续区域. [找到一个满足要求的之后](https://elixir.bootlin.com/linux/v3.16/source/mm/hugetlb.c#L757), 就使用 CMA 进行分配, 即调用 [alloc_contig_range()](https://elixir.bootlin.com/linux/v3.16/source/mm/hugetlb.c#L710) 进行实际分配. 需要注意的是通过 `hugepages =` 命令行选项在引导时分配的巨页也可以在运行时释放.

但是这个方案并不完善, 仍然有些悬而未决的问题, 比如:

*   随着运行时间的推移, 巨页分配所期望的大块连续区域往往会被逐渐消耗, 导致运行时分配巨页出现失败, 作者提到的目前避免这种情况的最好方法是, 在系统启动的早期进行巨大的页面分配. 其他可能的优化包括使用内存规整等 CMA 所支持的手段, 但这个补丁集未明确使用.

*   没有添加对 hugepage over commimit 的支持, 即在 `/proc/sys/vm/nr_overcommit_hugepages > 0` 时按需分配一个巨大的页面. 因为作者不确定在这种情况下分配一个巨页是否合理, 毕竟巨页的分配存在一定的开销.

因为问题 1 的存在, 导致运行时的 HugeTLB 分配机制并没有起到实质的作用. 因为它实际上只在系统加载的早期阶段能真正起作用, 此时大部分内存是空闲的. 一段时间后, 内存被不可移动的页面分割, 已经有了太多的内存外部碎片, 因此找到连续 1GB 块的机会接近于零. 而在大规模情况下, 重新启动服务器以分配巨页是非常昂贵和复杂的. 同时, 解决方案选择也比较困难. 因为即使当前系统中的工作负载没有使用, 那么在预留的 CMA 内存中再保留一定比例的内存用作 HugeTLB 的分配也是一种巨大的浪费: 因为并非所有工作负载都能从使用 1GB 的页面中获益.

随后 v5.7 版本, FaceBook 的开发者 Roman Gushchin [mm: using CMA for 1 GB hugepages allocation](https://lore.kernel.org/all/20200407163840.92263-1-guro@fb.com) 对问题 1 进行妥善地处理. 在启动时通过 [hugetlb_cma_reserve()](https://elixir.bootlin.com/linux/v5.7/source/mm/hugetlb.c#L5562) 预留一个专用的 CMA 区域 (通过启动参数 [`hugetlb_cma =`](https://elixir.bootlin.com/linux/v5.7/source/mm/hugetlb.c#L5556) 来指定预留的大小), 运行时使用 CMA 分配器和专用 CMA 区域来分配巨大的页面. 在这种情况下, 可以以很高的概率成功分配巨大的页面, 但是如果没有人使用 1GB 的巨大页面, 内存不会完全浪费. 因为它可以用于页面缓存、匿名内存、THP 等. 这个补丁同时适配了 [x86](https://elixir.bootlin.com/linux/v5.7/source/arch/x86/kernel/setup.c#L1162) 和 [arm64](https://elixir.bootlin.com/linux/v5.7/source/arch/arm64/mm/init.c#L463), 但是当前只支持 4K PAGE_SIZE 的情况. 由于事先不确定会在哪个 NUMA 节点上分配巨页内存, 因此预留时会尝试在[每个 NUMA 节点上都预留](https://elixir.bootlin.com/linux/v5.7/source/mm/hugetlb.c#L5587) 一定量的内存, 为 HugeTLB 预留的 CMA 内存专区存储在 [hugetlb_cma](https://elixir.bootlin.com/linux/v5.7/source/mm/hugetlb.c#L49) 数组中, 需要时通过 [cma_alloc()](https://elixir.bootlin.com/linux/v5.7/source/mm/hugetlb.c#L1260) 从预留区域 hugetlb_cma 中分配. 因此在未使用时这块内存完全可以用作他用,

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2014/06/04 | Luiz Capitulino <lcapitulino@redhat.com> | [hugetlb: add support gigantic page allocation at runtime](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=944d9fec8d7aee3f2e16573e9b6a16634b33f403) | 通过在 CMA 区域中分配 HugeTLB 来支持 HugeTLB 的运行时分配. | v1 ☑ 3.16-rc1 | [LORE v3,0/5](https://lore.kernel.org/lkml/1397152725-20990-1-git-send-email-lcapitulino@redhat.com), [关键 COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=944d9fec8d7aee3f2e16573e9b6a16634b33f403) |
| 2016/02/03 | Vlastimil Babka <vbabka@suse.cz> | [mm, hugetlb: don't require CMA for runtime gigantic pages](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=080fe2068e1c7f19f565b30b78baf78edf16a980) | commit 944d9fec8d7a ("hugetlb: hugetlb: add support for gigantic page allocation at runtime") 通过 alloc_contig_range() 支持了运行时巨页的分配, 但是这项支持仅在启用 CONFIG_CMA 时可用. 但是其实它不依赖于 MIGRATE_CMA 页面块和相关的基础设施, 只需要一些简单的调整就可以不依赖 CONFIG_CMA, 只需要 CONFIG_MEMORY_ISOLATION 而不是完全的 CONFIG_CMA.<br> 在这个补丁之后, 只需要启用了 CONFIG_MEMORY_ISOLATION, alloc_contig_range() 就支持运行时分配巨页, 而无需在页面分配器快速路径中进行特定于 cma 的检查. 注意 CONFIG_CMA slect 了 CONFIG_MEMORY_ISOLATION. | v1 ☑✓ 4.5-rc3 | [LORE](https://lore.kernel.org/all/1454521811-11409-1-git-send-email-vbabka@suse.cz) |
| 2019/03/27 | Alexandre Ghiti <alex@ghiti.fr> | [Fix free/allocation of runtime gigantic pages](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4eb0716e868eed963967adb0b1b11d9bd8ca1d01) | NA | v8 ☑✓ 5.2-rc1 | [LORE v8,0/4](https://lore.kernel.org/all/20190327063626.18421-1-alex@ghiti.fr) |
| 2020/04/10 | Roman Gushchin <guro@fb.com> | [mm: using CMA for 1 GB hugepages allocation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=cf11e85fc08cc6a4fe3ac2ba2e610c962bf20bc3) | 引入 hugetlb_cma 参数, 预留一块指定大小的 CMA 区域用来做 HugeTLB 的分配. 对于有多个 NUMA 节点的机器, 每个 NUMA 节点上都会进行预留. | v1 ☑ 5.7-rc1 | [LORE v5,0/2](https://lore.kernel.org/all/20200407163840.92263-1-guro@fb.com) |
| 2020/06/18 | Barry Song <song.bao.hua@hisilicon.com> | [arm64: mm: reserve hugetlb CMA after numa_init](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=618e07865b7453d02410c1f3407c2d78a670eabb) | ARM64 中 HugeTLB CMA 的预留应该在 numa_init 之后. | v1 ☑ 5.8-rc2 | [LORE v3](https://lore.kernel.org/all/20200617215828.25296-1-song.bao.hua@hisilicon.com), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=618e07865b7453d02410c1f3407c2d78a670eabb) |
| 2020/07/01 | Roman Gushchin <guro@fb.com> | [arm64/hugetlb: Reserve CMA areas for gigantic pages on 16K and 64K configs](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=abb7962adc80ab4f4313e8a065302525b6a9c2dc) | ARM64 下不光有[多种 PAGE_SIZE, 还有各类 CONT PAGE](https://elixir.bootlin.com/linux/v5.9/source/arch/arm64/mm/hugetlbpage.c#L25) 的情况. HugeTLB CMA 当前不支持 ARM64 下其他 PAGE_SIZE 情况下的预留, 因此引入 arm64_hugetlb_cma_reserve() 增加了 ARM64 下多种 PAGE_SIZE 的支持. | v1 ☑ 5.9-rc1 | [LORE v3](https://lore.kernel.org/all/1593578521-24672-1-git-send-email-anshuman.khandual@arm.com), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=abb7962adc80ab4f4313e8a065302525b6a9c2dc) |
| 2020/07/13 | Roman Gushchin <guro@fb.com> | [powerpc/hugetlb/cma: Allocate gigantic hugetlb pages using CMA](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=ef26b76d1af61b90eb0dd3da58ad4f97d8e028f8) | POWERPC 支持从 CMA 中分配 HugeTLB. | v5 ☑ 5.9-rc1 | [LORE v3](https://lore.kernel.org/all/20200407163840.92263-1-guro@fb.com), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ef26b76d1af61b90eb0dd3da58ad4f97d8e028f8) |
| 2022/04/13 | liupeng (DM) <liupeng256@huawei.com> | [hugetlb: Fix some incorrect behavior](https://patchwork.kernel.org/project/linux-mm/cover/20220413032915.251254-1-liupeng256@huawei.com/) | 631696 | v3 ☐☑ | [LORE v3,0/4](https://lore.kernel.org/r/20220413032915.251254-1-liupeng256@huawei.com) |
| 2022/04/13 | Anshuman Khandual <anshuman.khandual@arm.com> | [tlb/hugetlb: Add framework to handle PGDIR_SIZE HugeTLB pages](https://patchwork.kernel.org/project/linux-mm/patch/20220413100714.509888-1-anshuman.khandual@arm.com) | 改变 tlb_remove_huge_tlb_entry() 来适应更大的 PGDIR_SIZE HugeTLB 页面, 通过添加一个新的助手 tlb_flush_pgd_range(). 而这里也更新 struct mmu_gather 需要, 即添加一个新成员 cleared_pgds. | v1 ☐☑ | [LORE v1](https://lore.kernel.org/r/20220413100714.509888-1-anshuman.khandual@arm.com) |


#### 7.1.7.4 Numa Aware HugeTLB reserve
-------

HugeTLB CMA 在设计的时候, 已经考虑了 NUMA 的存在.

1.  在 [mm: using CMA for 1 GB hugepages allocation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=cf11e85fc08cc6a4fe3ac2ba2e610c962bf20bc3) 引入 hugetlb_cma 参数的过程中, Roman Gushchin 发现 CMA 并没有一个在指定 NUMA 节点上预留和分配内存的能力, 因此 [mm: cma: NUMA node interface](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8676af1ff2d28e64e5636147821bda7524cf007d) 增加了尝试在特定节点上 [预留](https://elixir.bootlin.com/linux/v5.7/source/include/linux/cma.h#L33) 和分配连续内存的能力. 注意如果指定的节点无法分配, 它将回退到其他节点进行分配.

2.  此外 [hugetlb: add support for gigantic page allocation at runtime](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=944d9fec8d7aee3f2e16573e9b6a16634b33f403) 在运行时动态分配 HugeTLB 内存时, 可以在指定节点下 `/sys/devices/system/node/node1/hugepages/hugepages-1048576kB/nr_hugepages` 的 HugeTLB 接口中进行动态分配.

随后内核开发者考虑预留 HugeTLB 空间时也可以精细化控制不同 NUMA NODE 上预留的空间, 通过扩展 HugeTLB 的启动参数解析工作, 增加各个节点上的精细化预留.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/10/05 | Zhenguo Yao <yaozhenguo1@gmail.com> | [hugetlbfs: Extend the definition of hugepages parameter to support node allocation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=b5389086ad7be0453c55e0069a89856d1fbdf605) | 当前内核允许指定启动时要分配的 hugepages 数目. 但目前所有节点的 hugepages 都是平衡的. 在某些场景中, 我们只需要在一个节点中使用 hugepags.<br> 例如: DPDK 需要与 NIC 位于同一节点的 hugepages. 如果 DPDK 在 node1 中需要 4 个 1G 大小的 HugePage, 并且系统有 16 个 numa 节点. 我们必须在内核 cmdline 中保留 64 个 HugePage. 但是, 只使用了 4 个 hugepages. 其他的应该在引导后释放. 如果系统内存不足(例如: 64G), 这将是一项不可能完成的任务. 因此, 添加 hugepages_node 内核参数以指定启动时要分配的 hugepages 的节点数. | v1 ☑ 5.16-rc1 | [2021/08/20 PatchWork RFC](https://patchwork.kernel.org/project/linux-mm/patch/20210820030536.25737-1-yaozhenguo1@gmail.com)<br>*-*-*-*-*-*-*-* <br>[2021/08/23 PatchWork v1]](https://patchwork.kernel.org/project/linux-mm/patch/20210823130154.75070-1-yaozhenguo1@gmail.com)<br>*-*-*-*-*-*-*-* <br>[2021/10/05 PatchWork  v8](https://patchwork.kernel.org/project/linux-mm/patch/20211005054729.86457-1-yaozhenguo1@gmail.com), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b5389086ad7be0453c55e0069a89856d1fbdf605) |
| 2021/10/15 | Baolin Wang <baolin.wang@linux.alibaba.com> | [hugetlb: Support node specified when using cma for gigantic hugepages](https://patchwork.kernel.org/project/linux-mm/patch/bb790775ca60bb8f4b26956bb3f6988f74e075c7.1634261144.git.baolin.wang@linux.alibaba.com) | 所有在线节点的 hugepages 运行时分配的 CMA 区域大小是平衡的, 但我们还希望指定每个节点的 CMA 大小, 或者在某些情况下仅指定一个节点, 这与 [hugetlb 的分 numa 节点指定大小](https://patchwork.kernel.org/project/linux-mm/patch/20211005054729.86457-1-yaozhenguo1@gmail.com)类似. | v3 ☐ | [2021/10/10 PatchWork v1](https://patchwork.kernel.org/project/linux-mm/patch/bb790775ca60bb8f4b26956bb3f6988f74e075c7.1634261144.git.baolin.wang@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[2021/10/15 PatchWork v3](https://patchwork.kernel.org/project/linux-mm/patch/bb790775ca60bb8f4b26956bb3f6988f74e075c7.1634261144.git.baolin.wang@linux.alibaba.com) |
| 2021/11/17 | Mina Almasry <almasrymina@google.com> | [hugetlb: Add `hugetlb.*.numa_stat` file](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f47761999052b1cc987dd3e3d3adf47997358fc0) | 添加 `hugetlb.*.numa_stat`, 它显示 hugetlb 在 cgroup 的 numa 使用信息. 类似于 memory.numa_stat. | v8 ☑ 5.17-rc1 | [PatchWork v1](https://patchwork.kernel.org/project/linux-mm/patch/20211019215437.2348421-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://patchwork.kernel.org/project/linux-mm/patch/20211020190952.2658759-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v7](https://patchwork.kernel.org/project/linux-mm/patch/20211117201825.429650-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v8](https://lore.kernel.org/all/20211123001020.4083653-1-almasrymina@google.com) |


### 7.1.8 代码段大页
-------

#### 7.1.8.1 用户程序代码段大页
-------

由于应用程序的代码段 text 和数据段 data 都是直接映射到程序二进制文件本身的, 因此如果不通过修改程序运行时库等方式, 很难把这些段使用 HugeTLB 来映射. 但是将这些段映射成大页可以有效地减少 iTLB-miss. 网上也经常会看到相关的一些讨论. 参见 [stackoverflow: usage of hugepages for text and data segments](https://stackoverflow.com/questions/36576462/usage-of-hugepages-for-text-and-data-segments).

Google 的工程师 Mina Almasry 提出了一种新的思路, 通过 [mremap 的方式重新映射程序的的 ELF 到大页上](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=550a7d60bd5e35a56942dba6d8a26752beb26c9f), 来支持代码段大页. 作者提供了一个用户态工具库 [chromium/hugepage_text](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chromeos/hugepage_text) 来辅助完成这项工作. 可以在尽可能不修改应用程序代码的情况下完成代码段大页的映射, 可以显著提升应用程序的性能. 具体使用也可以参考作者提交的测试用例 [commit 12b613206474 ("mm, hugepages: add hugetlb vma mremap() test")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=12b613206474cea36671d6e3a7be7d1db7eb8741).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/10/14 | Mina Almasry <almasrymina@google.com> | [mm, hugepages: add mremap() support for hugepage backed vma](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=12b613206474cea36671d6e3a7be7d1db7eb8741) | 通过简单地重新定位页表项, 使得 mremap() 支持 hugepage 的 vma 段. 页表条目被重新定位到 mremap() 上的新虚拟地址.<br> 作者验证的测试场景是一个简单的 bench: 它在 hugepages 中重新加载可执行文件的 ELF 文本, 这大大提高了上述可执行文件的执行性能.<br> 将 hugepages 上的 mremap 操作限制为原始映射的大小, 因为底层 hugetlb 保留还不能处理到更大的大小的重映射.<br> 在 mremap () 操作期间, 我们检测 pmd_shared 的映射, 并在 mremap () 期间取消这些映射的共享. 在访问和故障时, 再次建立共享. | v1 ☑ 5.16-rc1 | [PatchWork v1](https://patchwork.kernel.org/project/linux-mm/patch/20210730221522.524256-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v4,1/2](https://patchwork.kernel.org/project/linux-mm/patch/20211006194515.423539-1-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v7,1/2](https://lore.kernel.org/all/20211013195825.3058275-1-almasrymina@google.com), [关键 COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=550a7d60bd5e35a56942dba6d8a26752beb26c9f)<br>*-*-*-*-*-*-*-* <br>[PatchWork v8,1/2](https://patchwork.kernel.org/project/linux-mm/patch/20211014200542.4126947-1-almasrymina@google.com) |
| 2022/02/02 | Mike Kravetz <mike.kravetz@oracle.com> | [Add hugetlb MADV_DONTNEED support](https://patchwork.kernel.org/project/linux-mm/cover/20220128222605.66828-1-mike.kravetz@oracle.com/) | 609660 | v1 ☐☑ | [PatchWork v1,0/3](https://lore.kernel.org/all/20220128222605.66828-1-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2,0/3](https://lore.kernel.org/r/20220202014034.182008-1-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/3](https://lore.kernel.org/r/20220215002348.128823-1-mike.kravetz@oracle.com) |

#### 7.1.8.1 内核代码段大页
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/10/07 | Song Liu <song@kernel.org> | [vmalloc_exec for modules and BPF programs](https://lore.kernel.org/all/20220818224218.2399791-1-song@kernel.org) | 这个补丁集允许内核动态分配的代码段 (比如模块、bpf 程序、各种 trampoline 等) 共享巨大的页面. 这个想法源自于 [Peter Zijlstra 在讨论 mm/vmalloc: introduce vmalloc_exec which allocates RO+X memory](https://lore.kernel.org/bpf/Ys6cWUMHO8XwyYgr@hirez.programming.kicks-ass.net) 中的建议. 最终目标是仅在 2MB 页面中承载内核文本(当前仅针对对于 x86_64). | v1 ☐☑✓ | [2022/08/18 LORE v1,0/5](https://lore.kernel.org/all/20220818224218.2399791-1-song@kernel.org)<br>*-*-*-*-*-*-*-* <br>[2022/10/07 LORE v2,0/4](https://lore.kernel.org/all/20221007234315.2877365-1-song@kernel.org) |
| 2022/05/19 | Song Liu <song@kernel.org> | [module: introduce module_alloc_huge](https://lore.kernel.org/all/20220520031548.338934-5-song@kernel.org) | TODO | v3 ☐☑✓ | [LORE v3,4/8](https://lore.kernel.org/all/20220520031548.338934-5-song@kernel.org) |
| 2021/12/07 | Xu Yu <xuyu@linux.alibaba.com> | [anolis: mm, thp: introduce hugetext framework](https://github.com/gatieme/linux/commit/39350c152353da9bf5f8f8eb729cd3004cc2fc3c) | [OpenAnolis Cloud-Kernel 代码大页 - mysql 性能测试](https://openanolis.cn/sig/Cloud-Kernel/doc/475049355931222178) | v3 ☐☑✓ | [COMMIT 00/10](https://lore.kernel.org/all/20220520031548.338934-5-song@kernel.org) |


### 7.1.9 HugeTLBFS
--------


| 时间 | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:---:|:----:|:---:|:----:|:---------:|:----:|
| 2015/06/22 | Mike Kravetz <mike.kravetz@oracle.com> | [hugetlbfs: add fallocate support](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log?id=72079ba0dfefc1444b4ef98a2fa3d040838a775f) | 为 HugeTLBFS 增加了 fallocate 功能. 目前, Hugetlbfs 用于那些想要对巨大页面使用进行高度控制的应用程序. 通常, 大型 HugeTLBFS 文件用于将大量巨面映射到应用程序进程中. 应用程序知道何时不再使用这些大文件中的页面范围, 理想情况下希望将它们释放回子池或全局池以作其他用途. fallocate() 系统调用提供了一个用于预分配和在文件中打孔的接口. 这是基于 shmem 版本的, 但它有相当大的分歧. 我们既不用担心交换, 也不用担心新文件的密封. 这允许我们在 HugeTLBFS 文件中移动物理内存, 而不需要映射它. 这也使我们能够支持 MADV_REMOVE, 因为它目前是使用 fallocate () 实现的. MADV_REMOVE 允许我们从 HugeTLBFS 文件中间删除数据, 这在以前是不可能的. | v5 ☐☑✓ | [LORE RFC,0/4](https://lore.kernel.org/lkml/1429225378-22965-1-git-send-email-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,00/10](https://lore.kernel.org/lkml/1437502184-14269-1-git-send-email-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/9](https://lore.kernel.org/all/1435019919-29225-1-git-send-email-mike.kravetz@oracle.com) |
| 2022/06/13 | Mike Kravetz <mike.kravetz@oracle.com> | [hugeTLBfs: zero partial pages during fallocate hole punch](https://patchwork.kernel.org/project/linux-mm/patch/20220613180858.15933-1-mike.kravetz@oracle.com/) | NA | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220613180858.15933-1-mike.kravetz@oracle.com) |



### 7.1.10 HugeTLB vmemmap
-------

#### 7.1.10.1 DMEMFS
--------


针对 kernel 中元数据对内存资源占用过高的问题, 腾讯云设计了全新的文件系统 Dmemfs(Direct Memory File System), 可以直接管理部分系统预留的虚拟机内存服务, 提高系统的资源利用率降低平台成本. 这个方案不仅提高了系统的资源利用率, 能够降低平台成本并最终让利于用户, 同时也给系统开销降低提供了一种新的思路.



| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/12/07 | Yulei Zhang <yulei.kernel@gmail.com>/<yuleixzhang@tencent.com> | [Enhance memory utilization with DMEMFS](https://lwn.net/ml/linux-kernel/cover.1607332046.git.yuleixzhang@tencent.com) | DMEMFS 目的非常明确, 减少云场景下 struct page 结构体本身的内存消耗. 效果也是非常明显, 320G 可以省出 5G 的内存. | RFC, v2 ☐ | [2020/10/8 LKML 00/35](https://lkml.org/lkml/2020/10/8/139)<br>*-*-*-*-*-*-*-* <br>[2020/12/07 PatchWork RFC,V2,00/37](https://patchwork.kernel.org/project/linux-mm/cover/cover.1607332046.git.yuleixzhang@tencent.com) |

#### 7.1.10.2 HVO(HugeTLB Vmemmap Ooptimize)
--------


*   Free some vmemmap pages of HugeTLB page

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/05/10 | Muchun Song <songmuchun@bytedance.com> | [Free some vmemmap pages of HugeTLB page](https://patchwork.kernel.org/project/linux-mm/cover/20210510030027.56044-1-songmuchun@bytedance.com) | HugeTLB 使用的大量 struct page 里面有一部分没啥用可以省略. 复合页中只有头页面 (compound_head) 中的信息, 剩余尾页面 (tail pages) 中填充的信息是一样的. 因此一个 2M 的 HugeTLB 在 4K page size 的 x86_64 上会用到 512 个 struct page, 这 512 个 struct page 结构体本身占用了 sizeof(struct page) * 512 / PAGE_SIZE = 8 pages. 所以, 其实可以将最后 7 个 tail pages 的信息有点冗余, 将他们合并为一个页面(将 page 2~7 全部映射到 page 1), 这样只需要实际占用 2 个 page 就完成了映射. 这组补丁节省了大量的内存空间, 当然负面作用是分配和释放的时候会慢个 2 倍, 不过都比较小, MS 级别. | v23 ☑ 5.14-rc1 | [PatchWork v23,0/9](https://patchwork.kernel.org/project/linux-mm/cover/20210510030027.56044-1-songmuchun@bytedance.com) |
| 2021/10/18 | Muchun Song <songmuchun@bytedance.com> | [Free the 2nd vmemmap page associated with each HugeTLB page](https://lore.kernel.org/patchwork/patch/1459641) | NA  | v2 ☐ | [2021/07/14 PatchWork RFC](https://patchwork.kernel.org/project/linux-mm/cover/20210714091800.42645-1-songmuchun@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[2021/10/18 PatchWork v6,0/5](https://patchwork.kernel.org/project/linux-mm/cover/20211018102043.78685-1-songmuchun@bytedance.com) |
| 2022/02/08 | Muchun Song <songmuchun@bytedance.com> | [arm64: mm: hugetlb: add support for free vmemmap pages of HugeTLB](https://patchwork.kernel.org/project/linux-mm/patch/20220208054632.66534-1-songmuchun@bytedance.com/) | 本系列文章可以极大地减少 2MB HugeTLB 页面的结构页面开销. 与之前的方法相比, 它进一步减少了 2MB HugeTLB 的结构页面开销 12.5%, 这意味着每 1TB HugeTLB 的结构页面开销为 2GB. | v2 ☑ 5.18-rc1 | [PatchWork v2,2/2](https://lore.kernel.org/all/20220208054632.66534-2-songmuchun@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[LORE v7,0/5](https://lore.kernel.org/all/20211101031651.75851-1-songmuchun@bytedance.com) |
| 2022/02/10 | Joao Martins <joao.m.martins@oracle.com> | [sparse-vmemmap: memory savings for compound devmaps (device-dax)](https://patchwork.kernel.org/project/linux-mm/cover/20220210193345.23628-1-joao.m.martins@oracle.com/) | 这个系列, 通过追求类似于 Muchun Song [Free some vmemmap pages of HugeTLB page](https://patchwork.kernel.org/project/linux-mm/cover/20210510030027.56044-1-songmuchun@bytedance.com) 的方法, 针对带有 compound_pages 的 devmap, 最小化了 struct page 的开销. | v5 ☐☑ | [PatchWork v5,0/5](https://lore.kernel.org/r/20220210193345.23628-1-joao.m.martins@oracle.com) |



| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/03/18 | Muchun Song <songmuchun@bytedance.com> | [add hugetlb_free_vmemmap sysctl](https://patchwork.kernel.org/project/linux-mm/cover/20220307130708.58771-1-songmuchun@bytedance.com/) | 621009 | v3 ☐☑ | [2022/03/07 LORE v3,0/4](https://lore.kernel.org/r/20220307130708.58771-1-songmuchun@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[2022/03/18 LORE v4,0/4](https://lore.kernel.org/r/20220318100720.14524-1-songmuchun@bytedance.com) |
| 2022/04/12 | Muchun Song <songmuchun@bytedance.com> | [add hugetlb_optimize_vmemmap sysctl](https://patchwork.kernel.org/project/linux-mm/cover/20220412111434.96498-1-songmuchun@bytedance.com/) | 631440 | v7 ☐☑ | [LORE v7,0/4](https://lore.kernel.org/r/20220412111434.96498-1-songmuchun@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[LORE v8,0/4](https://lore.kernel.org/r/20220413144748.84106-1-songmuchun@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[LORE v9,0/4](https://lore.kernel.org/r/20220429121816.37541-1-songmuchun@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[LORE v10,0/4](https://lore.kernel.org/r/20220509062703.64249-1-songmuchun@bytedance.com) |
| 2022/06/28 | Muchun Song <songmuchun@bytedance.com> | [Simplify hugetlb vmemmap and improve its readability](https://lore.kernel.org/all/20220628092235.91270-1-songmuchun@bytedance.com) | TODO | v2 ☐☑✓ | [LORE 0/6](https://lore.kernel.org/lkml/20220613063512.17540-1-songmuchun@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/8](https://lore.kernel.org/all/20220628092235.91270-1-songmuchun@bytedance.com) |
| 2022/08/02 | Joao Martins <joao.m.martins@oracle.com> | [[v1] mm/hugetlb_vmemmap: remap head page to newly allocated page](https://patchwork.kernel.org/project/linux-mm/patch/20220802180309.19340-1-joao.m.martins@oracle.com/) | 664896 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220802180309.19340-1-joao.m.martins@oracle.com) |


### 7.1.11 HugeTLB High-Granularity Mapping
-------

HugeTLB 高粒度映射 (HugeTLB High-Granularity Mapping, HGM)(早期也叫 HugeTLB Double Mapping) 的概念. 从广义上讲, 本系列将教 HugeTLB 如何以不同粒度映射 HugeTLB 页面, 更重要的是, 如何部分映射 HugeTLB 页面. 高粒度映射不会分解大页面本身; 它只影响它们的映射方式.

对于复制后的热迁移, 使用 userfaultfd, 一直以来必须安装一个完整的大页面, 才能允许客户访问该页面. 这是因为, 现在, 要么整个大页要么被映射, 要么没有映射. 所以 Guest 要么可以访问整个页面, 要么一个都不能访问. 这使得 1G HugeTLB 支持的虚拟机复制后热迁移完全不可行的.

因此能够将 HugeTLB 内存按照 PAGE_SIZE pte 进行映射, 在复制后热迁移和内存故障处理中具有重要意义.


通过使用 HugeTLB 高粒度映射, 我们可以映射一个大页中的 PAGE_SIZE 大小的页面, 从而允许客户机只访问 PAGE_SIZE 块, 并在访问其他页面块时触发 Page Fault. 这使用户空间可以灵活地将 PAGE_SIZE 内存块安装到一个巨大的页面中, 使得迁移 1G 支持的虚拟机完全可行, 并且极大地减少了 2M 支持的虚拟机在复制后的 vCPU 暂停时间. 参见 phoronix 报道 [Google Moves Forward With HugeTLB HGM For The Linux Kernel](https://www.phoronix.com/news/Linux-HugeTLB-HGM).

1. 在通过网络完全复制一个巨大的页面后, 我们将希望将映射分解为正常情况下的样子 (例如, 一个 PUD 对应一个 1G 页面). 我们没有让内核自动完成这一工作, 而是让用户空间来告诉我们折叠一个范围 (通过 [MADV_COLLAPSE](https://lore.kernel.org/linux-mm/20220604004004.954674-10-zokeefe@google.com)).

2. 当在 HugeTLB 页面中发现内存错误时, 如果我们可以只映射包含错误的 PAGE_SIZE 部分, 这将是理想的. 这就是 THPs 能够做到的. 使用高粒度映射, 我们可以做到这一点, 但在本补丁系列中没有解决这个问题.

3. Userspace API 层次, 提供了两种利用高粒度映射的方法: 用户空间与高粒度映射交互的方式主要有两种:① 在适当配置的 userfaultfd VMA 中使用 UFFDIO_CONTINUE 创建它们.② 使用 MADV_COLLAPSE 分解高粒度映射.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2022/06/24 | James Houghton <jthoughton@google.com> | [hugetlb: Introduce HugeTLB high-granularity mapping](https://patchwork.kernel.org/project/linux-mm/cover/20220624173656.2033256-1-jthoughton@google.com/) | 引入了 HugeTLB 高粒度映射 (HGM)(曾经叫做 HugeTLB double mapping) 的概念. 高粒度映射不需要分解大页面本身; 它只影响它们的映射方式. 从广义上讲, 它主要探索如何在不同粒度上映射 HugeTLB 页面, 更重要的是, 部分映射 HugeTLB 页面. | v1 ☐☑ | [LORE RFC,00/26](https://lore.kernel.org/linux-mm/20220624173656.2033256-1-jthoughton@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,00/47](https://lore.kernel.org/r/20221021163703.3218176-1-jthoughton@google.com)<br>*-*-*-*-*-*-*-* <br>[2023/01/05 LORE v2,rebase](https://lore.kernel.org/lkml/20230105101844.1893104-1-jthoughton@google.com) |

### 7.1.12 HugeTLB Freeing
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:----:|:--:|:----------:|:----:|
| 2023/01/01 | James Houghton <jthoughton@google.com> | [hugetlb: unshare some PMDs when splitting VMAs](https://patchwork.kernel.org/project/linux-mm/patch/20230101230042.244286-1-jthoughton@google.com) | PMD 共享只能在 PUD_SIZE 对齐的 VMA 中进行; 然而, 有可能在不首先共享 PMD 的情况下拆分 HugeTLB VMA. 在某些情况下, 这不是问题, 例如 userfaultfd_register 和 mprotect, 其中 PMD 在执行任何操作之前都是非共享的. 然而, mbind()和 madvise(MADV_DONTDUMP)可以在不首先取消共享的情况下导致拆分. 在 hugetlb_vm_op_open 中取消共享似乎很理想, 但这只会在新 VMA 中取消共享 PMD. | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20230101230042.244286-1-jthoughton@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20230104231910.1464197-1-jthoughton@google.com) |


### 7.1.x More HugeTLB Patchset
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/12/22 | Liang Li <liliang.opensource@gmail.com> | [add support for free hugepage reporting](https://lore.kernel.org/patchwork/patch/1355899) | Free page reporting 只支持伙伴系统中的页面, 它不能报告为 hugetlbfs 预留的页面. 这个补丁在 hugetlb 的空闲列表中增加了对报告巨大页的支持, 它可以被 virtio_balloon 驱动程序用于内存过载和预归零空闲页, 以加速内存填充和页面错误处理. | RFC ☐ | [PatchWork RFC,0/3](https://patchwork.kernel.org/project/linux-mm/cover/20201222074538.GA30029@open-light-1.localdomain) |
| 2021/10/07 | Mike Kravetz <mike.kravetz@oracle.com> | [hugetlb: add demote/split page functionality](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=8531fc6f52f5fc201f43d8c36c2606e25748b1c4) | 实现了 hugetlb 降低策略. 提供了一种 "就地" 将 hugetlb 页面分割为较小的页面的方法. | v4 ☑ 5.16-rc1 | [2021/07/21 PatchWork 0/8](https://patchwork.kernel.org/project/linux-mm/cover/20210721230511.201823-1-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2021/08/16 PatchWork RESEND,0/8](https://patchwork.kernel.org/project/linux-mm/cover/20210816224953.157796-1-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2021/09/23 PatchWork v2,0/4](https://patchwork.kernel.org/project/linux-mm/cover/20210923175347.10727-1-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2021/10/07 PatchWork v4,0/5](https://patchwork.kernel.org/project/linux-mm/cover/20211007181918.136982-1-mike.kravetz@oracle.com) |
| 2022/02/01 | Anshuman Khandual <anshuman.khandual@arm.com> | [mm/hugetlb: Generalize ARCH_WANT_GENERAL_HUGETLB](https://patchwork.kernel.org/project/linux-mm/patch/1643718465-4324-1-git-send-email-anshuman.khandual@arm.com/) | 610328 |v1 ☐☑ | [PatchWork v1,0/1](https://lore.kernel.org/r/1643718465-4324-1-git-send-email-anshuman.khandual@arm.com) |
| 2022/03/07 | Mike Kravetz <mike.kravetz@oracle.com> | [hugetlb: do not demote poisoned hugetlb pages](https://patchwork.kernel.org/project/linux-mm/patch/20220307215707.50916-1-mike.kravetz@oracle.com/) | 621194 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220307215707.50916-1-mike.kravetz@oracle.com) |
| 2013/03/20 | David Rientjes <rientjes@google.com> | [mm, hugetlb: include hugepages in meminfo](https://lore.kernel.org/all/alpine.DEB.2.02.1303201207110.28997@chino.kir.corp.google.com) | alpine.DEB.2.02.1303201207110.28997@chino.kir.corp.google.com | v2 ☐☑✓ | [LORE](https://lore.kernel.org/all/alpine.DEB.2.02.1303201207110.28997@chino.kir.corp.google.com) |
| 2022/04/15 | Christophe Leroy <christophe.leroy@csgroup.eu> | [mm, hugetlbfs: Allow for"high"userspace addresses](https://patchwork.kernel.org/project/linux-mm/patch/ab847b6edb197bffdfe189e70fb4ac76bfe79e0d.1650033747.git.christophe.leroy@csgroup.eu/) | 632590 | v10 ☐☑ | [LORE v10](https://lore.kernel.org/r/ab847b6edb197bffdfe189e70fb4ac76bfe79e0d.1650033747.git.christophe.leroy@csgroup.eu) |
| 2022/05/08 | Mike Kravetz <mike.kravetz@oracle.com> | [hugetlb: Change huge pmd sharing synchronization again](https://patchwork.kernel.org/project/linux-mm/cover/20220420223753.386645-1-mike.kravetz@oracle.com/) | 之前 [v5.7 中添加了](https://lore.kernel.org/linux-mm/43faf292-245b-5db5-cce9-369d8fb6bd21@infradead.org) commit c0d0381ade79 ("hugetlbfs: use i_mmap_rwsem for more pmd sharing synchronization") 时, 社区就[报告了性能回归](https://lore.kernel.org/lkml/20200622005551.GK5535@shao2-debian). 当时,  Mike Kravetz 提出了一个[解决回归的建议](https://lore.kernel.org/linux-mm/20200706202615.32111-1-mike.kravetz@oracle.com), 但没有被合入. 这个补丁集使用一个新的 hugetlb 特定的 per-vma rw 信号量来同步 PMD 共享. 为了更好的演示这个性能回归, 作者构造了一个简单的测试用例. | v2 ☐☑ | [2022/04/20 LORE v2,0/6](https://lore.kernel.org/r/20220420223753.386645-1-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2022/05/08 LORE v3,0/8](https://lore.kernel.org/r/20220508183420.18488-1-mike.kravetz@oracle.com) |
| 2022/05/27 | Mike Kravetz <mike.kravetz@oracle.com> | [hugetlb: speed up linear address scanning](https://patchwork.kernel.org/project/linux-mm/cover/20220527225849.284839-1-mike.kravetz@oracle.com/) | 645704 | v1 ☐☑ | [LORE v1,0/3](https://lore.kernel.org/r/20220527225849.284839-1-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-*<br>[LORE v1,0/4](https://lore.kernel.org/r/20220616210518.125287-1-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-*<br>[[LORE v2,0/4](https://lore.kernel.org/r/20220621235620.291305-1-mike.kravetz@oracle.com) |
| 2022/08/24 | Mike Kravetz <mike.kravetz@oracle.com> | [hugetlb: Use new vma mutex for huge pmd sharing synchronization](https://lore.kernel.org/all/20220824175757.20590-1-mike.kravetz@oracle.com) | TODO | v1 ☐☑✓ | [LORE v1,0/8](https://lore.kernel.org/all/20220824175757.20590-1-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/9](https://lore.kernel.org/all/20220914221810.95771-1-mike.kravetz@oracle.com) |
| 2022/09/16 | Mike Kravetz <mike.kravetz@oracle.com> | [hugetlb: freeze allocated pages before creating hugetlb pages](https://patchwork.kernel.org/project/linux-mm/patch/20220916214638.155744-1-mike.kravetz@oracle.com/) | 677766 | v2 ☐☑ | [LORE v2,0/1](https://lore.kernel.org/r/20220916214638.155744-1-mike.kravetz@oracle.com)<br>*-*-*-*-*-*-*-*<br>[LORE v3,0/1](https://lore.kernel.org/r/20220921202702.106069-1-mike.kravetz@oracle.com) |
| 2022/09/21 | Doug Berger <opendmb@gmail.com> | [mm/hugetlb: hugepage migration enhancements](https://patchwork.kernel.org/project/linux-mm/cover/20220921223639.1152392-1-opendmb@gmail.com/) | 679187 | v1 ☐☑ | [LORE v1,0/3](https://lore.kernel.org/r/20220921223639.1152392-1-opendmb@gmail.com) |
| 2022/09/01 | Muchun Song <songmuchun@bytedance.com> | [mm: hugetlb: eliminate memory-less nodes handling](https://patchwork.kernel.org/project/linux-mm/patch/20220901083023.42319-1-songmuchun@bytedance.com/) | 673136 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220901083023.42319-1-songmuchun@bytedance.com) |
| 2022/10/19 | 黄杰 <huangjie.albert@bytedance.com> | [mm: hugetlb: support for shared memory policy](https://patchwork.kernel.org/project/linux-mm/patch/20221019092928.44146-1-huangjie.albert@bytedance.com/) | 为 hugetlb_vm_ops 实现 get/set_policy() 确保所有共享这个大页面文件的进程的 mempolicy 一致.<br> 在一些共享巨大页面的场景中: 如果我们需要限制 node0 内 vm 的内存使用, 那么我将 qemu 的 mempilciy 绑定设置为 node0, 但如果有一个进程 (如 virtiofsd) 与 vm 共享内存, 在这种情况下. 如果页面错误是由 virtiofsd 触发的, 分配的内存可能会到 node1, 而 node1 取决于 virtiofsd. 虽然我们可以使用 qemu 提供的内存预分配来避免这个问题, 但这种方法将显著增加 vm 的创建时间 (几秒钟, 取决于内存大小). 在我们连接 hugetlb_vm_ops(set/get_policy) 之后: 由 shmget() 创建的带有 SHM_HUGETLB 标志的共享内存段和 mmap(MAP_SHARED|MAP_HUGETLB) 也支持共享策略. | v2 ☐☑ | [LORE](https://lore.kernel.org/all/20221012081526.73067-1-huangjie.albert@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2](https://lore.kernel.org/all/20221019092928.44146-1-huangjie.albert@bytedance.com) |
| 2022/10/30 | Mike Kravetz <mike.kravetz@oracle.com> | [[v5] hugetlb: simplify hugetlb handling in follow_page_mask](https://patchwork.kernel.org/project/linux-mm/patch/20221030225825.40872-1-mike.kravetz@oracle.com/) | 690306 | v5 ☐☑ | [LORE v5,0/1](https://lore.kernel.org/r/20221030225825.40872-1-mike.kravetz@oracle.com) |


## 7.2 透明大页的支持
-------

hugetlb 的使用依赖于用户主动预留并使用, 适用于用户明确需要使用大量连续大块页面或者使用大页可以带来明显性能提升的场景. 但是多数情况下, 开发者并不知道自己是否需要 huge page. 这时候如果内核通过某种机制自动把普通的页面合并为 huge page, 或者在用户分配时就直接分配成 huge page, 让用户丝毫不感知 huge page 的存在, 但是却能享受到 huge page 带来的性能提升. 那自然是极好的. 鉴于此 THP(Transparent huge page) 孕育而生.

因为该过程不会被用户和应用感知, 所以被称为 "transparent", 也被称为动态大页. 与之相对的, 之前的 hugetlb 则被称为静态大页(static huge page), 那么自然. 一个是动态的, 一个是静态的. 两者的关系, 就类似于 CMA 和预留 DMA 的关系.

**2.6.38(2011 年 3 月发布)**

| LWN 归档 | Index entries for this article |
|:--------:|:------------------------------:|
| 0 | [Huge pages](https://lwn.net/Kernel/Index/#Huge_pages) |
| 1 | [Memory management/Huge pages](https://lwn.net/Kernel/Index/#Memory_management-Huge_pages) |

扩展阅读:

[卢钧轶: Huge Page 是否是拯救性能的万能良药？](https://www.cnblogs.com/cenalulu/p/4394695.html)

### 7.2.1 THP(Transparent Hugepage Support)
-------

2011 年 v2.6.38 期间 Andrea Arcangeli 为 linux 引入了 THP(Transparent huge page), 在应用需要 huge page 的时候, 可通过内存规整等操作, 为当前 VMA 分配一个 huge page, 因为该过程不会被应用感知到, 所以被称为 "transparent". 参见 [Transparent huge pages in 2.6.38](https://lwn.net/Articles/423584).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2010/12/15 | Andrea Arcangeli <aarcange@redhat.com> | [Transparent Hugepage Support #33](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=a664b2d8555c659127bf8fe049a58449d394a707) | 实现透明大页. | v33 ☑ [2.6.38-rc1](https://kernelnewbies.org/Linux_2_6_38#Transparent_huge_pages) | [LORE](https://lore.kernel.org/lkml/20101215051540.GP5638@random.random), [关键 COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=71e3aac0724ffe8918992d76acfe3aad7d872) |

1.      当发生 page fault 时, 内核将尝试使用 [do_huge_pmd_anonymous_page()](https://elixir.bootlin.com/linux/v2.6.38/source/mm/memory.c#L3305) 分配一个 huge page 来满足它. 如果分配成功, huge page 的页表就会被设置, 而该地址范围内的[任何现有的普通页面都将被释放](https://elixir.bootlin.com/linux/v2.6.38/source/mm/memory.c#L3313), 同时将 huge page 插入到进程 VMA 中. 如果没有 huge page 可用, 内核就会通过 [do_huge_pmd_wp_page_fallback()](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L910) 退回到普通页面, 应用程序永远不会感知到这些差异. [commit ("71e3aac0724f thp: transparent hugepage core")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=71e3aac0724ffe8918992d76acfe3aad7d872).

2.      但是分配时能够分配出 huge page 取决于物理上连续的大的内存块, 这是 Linux 内核程序员永远不能指望的. 因此在一个进程出现若干次小页面的 pge fault 之后, THP 补丁试图通过添加 "khugepaged" 内核线程来改善这种情况. 这个线程偶尔会尝试分配一个巨大的页面; 如果成功, 它会扫描内存, 寻找一个巨大的页面可以替代一堆较小的页面的位置. 因此, 可用的巨大页面应该快速放置到服务中, 最大限度地利用整个系统中的 hugepage. 内核还提供了一整套参数控制 khugepaged 线程的操作. 参见 [commit ("ba76149f47d8 thp: khugepaged")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ba76149f47d8).

3.      同时引入了 defrag 功能, 控制内核是否应该积极地使用内存规整 / 压缩(compaction) 以使更多的 hugepage 可用. 内存规整通过移动页面, 以创建空闲内存中大的、物理上连续的区域. 这些区域通常可以用来支持大量的分配, 对于创建 huge page 来说特别有用.


最早的实现只能用于匿名页面, 将 huge Huge Page(THP) SWAPpage 与 page cache 集成的工作尚未完成. 它也只处理一个页面大小为(2 MB) 的 PMD level 的大页. 当时 Mel Gorman 运行了一些[基准测试](https://lwn.net/Articles/423590), 在某些情况下可以提高 10% 左右. 一般来说, 结果不如 hugetlbfs 那样好, 但是 THP 更有可能被普遍的运用在生产环境中.


7.1 介绍的这种使用大页的方式看起来是挺怪异的, 需要用户程序做出很多修改. 而且, 内部实现中, 也需要系统预留一大部分内存. 基于此, 2.6.38 引入了一种叫[透明大页 Transparent huge pages](https://lwn.net/Articles/423584) 的实现. 如其名字所示, 这种大页的支持对用户程序来说是透明的.



它的实现原理如下. 在缺页中断中, 内核会 ** 尝试 ** 分配一个大页, 如果失败(比如找不到这么大一片连续物理页面), 就还是回退到之前的做法: 分配一个小页. 在系统内存紧张需要交换出页面时, 由于前面所说的根植内核的 4KB 页面大小的因, MM 会透明地把大页分割成小页再交换出去.



用户态程序现在可以完成无修改就使用大页支持了. 用户还可以通过 **madvice()** 系统调用给予内核指示, 优化内核对大页的使用. 比如, 指示内核告知其你希望进程空间的某部分要使用大页支持, 内核会尽可能地满足你.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/09/28 | Zi Yan <ziy@nvidia.com> | [1GB PUD THP support on x86_64](https://lkml.org/lkml/2020/9/28/973) | X86_64 支持 PUD 级别 (1G) 的匿名大页 | RFC,v2 ☐ | [2020/09/02 PatchWork RFC,00/16](https://patchwork.kernel.org/project/linux-mm/cover/20200902180628.4052244-1-zi.yan@sent.com)<br>*-*-*-*-*-*-*-* <br>[2020/09/28 PatchWork v2 00/30](https://patchwork.kernel.org/project/linux-mm/cover/20200928175428.4110504-1-zi.yan@sent.com) |
| 2021/05/10 | Muchun Song <songmuchun@bytedance.com> | [Overhaul multi-page lookups for THP](https://lore.kernel.org/patchwork/patch/1337675) | 提升大量页面查找时的效率 | v4 ☑ [5.12-rc1](https://kernelnewbies.org/Linux_5.12#Memory_management) | [PatchWork RFC](https://patchwork.kernel.org/project/linux-mm/cover/20201112212641.27837-1-willy@infradead.org) |
| 2021/05/10 | Ankur Arora <ankur.a.arora@oracle.com> | [Use uncached stores while clearing huge pages](https://patchwork.kernel.org/project/linux-mm/cover/20211020170305.376118-1-ankur.a.arora@oracle.com) | 本系列增加了对大页的非缓存页面清除的支持. [清除大页内存](https://patchwork.kernel.org/project/linux-mm/patch/20211020170305.376118-11-ankur.a.arora@oracle.com)时, 使用基于 [MOVNTI 指令](https://www.felixcloutier.com/x86/movnti) 的 [uncached clear page 接口](https://patchwork.kernel.org/project/linux-mm/patch/20211020170305.376118-4-ankur.a.arora@oracle.com).<br> 其动机是加快大型预分配虚拟机的创建, 并支持巨大的页面.<br> 支持非缓存页面清除有两种帮助:<br>1. 对于小于 LLC 大小的数据块, 未缓存的存储通常比缓存的存储慢, 而对于较大的数据块, 则更快. 2. 避免用无用的零替换潜在有用的缓存行.<br> 性能测试: 虚拟机创建 (对于预分配 2MB 后台页面的虚拟机) 在运行时有了显著的改进. | v2 ☐ | [PatchWork v2,00/14](https://patchwork.kernel.org/project/linux-mm/cover/20211020170305.376118-1-ankur.a.arora@oracle.com) |

THP 虽然实现了, 但是依旧存在着不少问题. 在 LSFMM 2015 进行了讨论, 参见 [Improving huge page handling](https://lwn.net/Articles/636162)

1.  Kirill Shutemov 首先描述了他提出的如何处理 THP 的引用计数的改变, 这个补丁集在 LWN 2014 年 11 月的文章 [Transparent huge page reference counting](https://lwn.net/Articles/619738) 中有详细的描述. 该补丁的关键部分是, 它允许在 PMD (巨大页面)和 PTE (常规页面)模式下同时映射一个巨大的页面. 正如作者所言, 这是一个大的补丁集, 而且仍然存在一些错误, 所以这组补丁并没有被合入主线. 参见 [THP 和 mapcount 之间的恩恩怨怨](https://richardweiyang-2.gitbook.io/kernel-exploring/00-index/02-thp_mapcount).

2.  剩下的一个问题是关于部分取消巨大页面的映射. 当一个进程取消映射一个 huge page 的一部分页面时, 有多种想法可以执行这种操作.

| 分割行为 | 描述 |
|:------:|:----:|
| 直接分割 | 将该页面拆分, 并将与释放的区域对应的单个页面返回给系统. |
| 延迟分割 | 也可以将这个 huge page 拆解成普通页面, 但是并不释放它们, 而是内核仍然维护组成这个 huge page 的普通页面, 将它们保持在一起. 这使得 huge page 在需要时允许它快速重新构建. 但是, 这也意味着没有任何内存实际上将被释放, 同时有必要将这个 huge page 及组成它的普通页面添加到一个特殊的列表中, 如果系统遇到内存压力, 可以在该列表中真正地进行分割. |


### 7.2.2 THP reference counting
-------

首先来看 THP 的引用计数的问题, 参见:

[Transparent huge page reference counting](https://lwn.net/Articles/619738)

[THP 和 mapcount 之间的恩恩怨怨](https://richardweiyang-2.gitbook.io/kernel-exploring/00-index/02-thp_mapcount).


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/10/06 | Kirill A. Shutemov"<kirill.shutemov@linux.intel.com> | [THP refcounting redesign](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=ddc58f27f9ee) | NA | v12 ☑ 4.5-rc1 | [LWN RFC 00/10](https://lwn.net/Articles/601781)<br>*-*-*-*-*-*-*-* <br>[LORE RFC v2,00/19](https://lore.kernel.org/lkml/1415198994-15252-1-git-send-email-kirill.shutemov@linux.intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE RFC v12,00/37](https://lore.kernel.org/lkml/1444145044-72349-1-git-send-email-kirill.shutemov@linux.intel.com) |
| 2016/05/06 | Andrea Arcangeli <aarcange@redhat.com> | [mm: thp: mapcount updates](https://lore.kernel.org/all/1462547040-1737-1-git-send-email-aarcange@redhat.com) | NA | v1 ☑ 4.6 | [LORE 0/3](https://lore.kernel.org/all/1462547040-1737-1-git-send-email-aarcange@redhat.com), [关键 COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6d0a07edd17c) |

### 7.2.3 THP allocations latencies
-------


LSFMM 2015 讨论了 THP 的诸多问题, 参见 [Improving huge page handling](https://lwn.net/Articles/636162)

最关键的问题在于, 我们分配出的 huge page 是否能带来性能的提升, 不管是处理 page fault 时分配出 huge page, 还是通过 khugepaged 合并出 huge page, 以及通过内存规整整出 huge page. 都是存在开销的. 首先在处理页面 page fault 上进行相当激进的 THP 分配尝试是否是一种良好的性能权衡. 参见 [LSF/MM TOPIC ATTEND](https://marc.info/?l=linux-mm&m=142056088232484&w=2)

    *   THP 分配增加了页面错误延迟, 因为高阶分配是出了名的昂贵. 页面分配 slowpath 现在会对 gfp_transshuge && !PF_KTHREAD 进行额外的检查, 以避免对用户页面错误进行更昂贵的同步压缩. 但即使是异步压缩也可能代价高昂.

    *   在 2MB 范围内的第一个 page fault 期间, 我们无法预测该范围内有多少实际将被访问, 理论上我们可以浪费[多达 511 个页面](https://www.spinics.net/lists/kernel/msg1928252.html). 或者, 范围内的页面可以从不同 NUMA 节点的 CPU 访问, 虽然基本页面可以都是本地的, 但 THP 可以远程到除一个 CPU 以外的所有 CPU. 由于这个错误的共享, 远程访问的成本将高于 huge page 省出来的 TLB 的开销.

因此很多时候, THP 的效果可能远不如想象中那么美好. 比如 SAP 建议为他们的应用程序[禁用 THPs](https://blogs.sap.com/2014/05/22/sap-iq-and-linux-hugepagestransparent-hugepages), 以提高性能.


#### 7.2.3.1 Improving huge page handling
-------

在 page fault 时分配 huge page 会带来较大的延迟.

Vlastimil 在 [LSFMM 2015 对 THP 的讨论中](https://lwn.net/Articles/636162) 提到, 由于不可能预测在 page fault 时提供 huge page 的好处, 所以最好少做一些. 相反, 应该主要在 khugepaged 守护进程中创建透明的巨大页面, 该守护进程可以在后台查看内存使用情况和折叠页面. 这样做需要重新设计 khugepaged, 这主要是为了在其他方法失败时填充巨大的页面. 但是它扫描速度很慢, 不能确定一个进程是否会从巨大的页面中受益; 特别是, 它不知道该进程是否会花费大量时间运行. 这可能是因为这个过程大多潜伏着等待外部事件的发生, 也可能是因为它即将退出.

Vlastimil 方法是通过将寻找 huge page 机会的扫描工作移动到进程上下文中来改进 khugepaged. 在某些时候, 例如从系统调用返回时, 每个进程都会扫描一部分内存, 或许还会将某些页折叠成 huge page. 它会部分基于成功率自动调整自身, 但也仅仅基于一个进程运行更频繁将做更多的扫描这一事实. 因为不涉及守护进程, 所以不会有额外的唤醒; 如果一个系统完全空闲, 就不会有页面扫描. 不过在 khugepaged 中折叠页面比在 page fault 时分配 huge page 要昂贵得多.

之前的想法太激进了, Andrea 等人有不同意见.

*   Andrea 认为在 khugepaged 中折叠页面比在 page fault 时分配 huge page 要昂贵得多. 要折叠一个页面, 内核必须将所有单独的小页面迁移 (复制) 到包含它们的新的巨大页面, 这需要花费较多的时间. 在需要之前, 进程上下文扫描可能有地方创建 huge page, 所以最好尽可能避免折叠页面.

*   Andi Kleen 认为, 在进程上下文中运行内存规整是一个糟糕的主意, 它会夺走并行性的机会. 规整扫描就应该在守护进程中完成, 这样它就可以在单独的核心上运行; 否则就会为受影响的进程创建过多的延迟.


Andrea 建议将工作放入工作队列中.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2014/10/22 |  Alex Thorlton | [Convert khugepaged to a task_work function](https://lwn.net/Articles/634384) | 将 huge page 的折叠扫描从 khugepaged 移动到 task_work 上下文. 在这个实现中, 扫描是由 `__khugepaged_enter()` 驱动的, 它在 vma_merge、fork 和 THP page fault 的尝试等事件中被调用, 例如不是完全周期性的事件. | RFC ☐ | [LKML 0/4](https://lkml.org/lkml/2014/10/22/931) |
| 2015/02/23 | Vlastimil Babka <vbabka@suse.cz> | [the big khugepaged redesign](https://lwn.net/Articles/634384) | 解决 huge page<br>1. 停止在 khugepaged 中预先分配大页面.<br>2. 只有在我们预期它们会成功的情况下才尝试错误分配.<br>3. 将折叠从 khugepaged 移动到 task_work 上下文.<br>4. thp 分配失败时唤醒 khugepaged. | RFC ☐ | [LORE RFC,0/6](https://lore.kernel.org/lkml/1424696322-21952-1-git-send-email-vbabka@suse.cz) |

但是上面的手段还是太激进了, 因此 Vlastimil 又进一步做了调整.

#### 7.2.3.2 Outsourcing THP allocations
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/05/11 | Vlastimil Babka <vbabka@suse.cz> | [Outsourcing page fault THP allocations to khugepaged](https://lwn.net/Articles/643891) | 由于 [the big khugepaged redesign](https://lwn.net/Articles/634384) 的改动过于激进, 社区建议被拆开重新适配, 将争议最小的一部分先提交到社区. 这组补丁不会将折叠扫描移动到 task_work 上下文, 而是专注于减少回收和压缩在 page fault 上下文中完成的工作, 通过将这一工作转移到 khugepaged. 这有两个好处:<br>1. 在 page fault 上下文中回收和规整会增加 page fault 的处理延迟, 这可能会抵消 THP 带来的收益, 特别是对于短期的分配, 这在 page fault 时都无法区分和甄别.<br>2. THP 分配在 page fault 仅使用异步规整 (asynchronous compaction), 这减少了延迟, 也提高了成功的概率, 失败不会导致延迟规整(deferred compaction). khugepaged 则使用更彻底的同步规整(synchronous compaction), 不会因为 need_resched() 而中途退出, 并将正确地配合延迟规整(deferred compaction) 机制. | RFC ☐ | [PatchWork RFC,0/4](https://lore.kernel.org/lkml/1431354940-30740-1-git-send-email-vbabka@suse.cz) |
| 2015/07/02 | Vlastimil Babka <vbabka@suse.cz> | [Outsourcing compaction for THP allocations to kcompactd](https://lwn.net/Articles/650051) | 本 RFC 系列是处理 THP 分配延迟尝试的另一个演进. 与前一个版本 [Outsourcing page fault THP allocations to khugepaged](https://lwn.net/Articles/643891) 的主要区别是借用了每个节点的 kcompactd. 试着把所有东西都放进 khugepaged 太笨拙了, 而 kcompactd 可以有更多的好处. 作者用 mmtests/thpscale 对它进行了简单的测试, 但是目前效果并不明显. | RFC ☐ | [PatchWork RFC v2,0/4](https://lore.kernel.org/lkml/1435826795-13777-1-git-send-email-vbabka@suse.cz) |



### 7.2.4 improve THP collapse rate
-------


*   khugepaged_max_ptes_none (`/sys/kernel/mm/transparent_hugepage/khugepaged/max_ptes_none`)

khugepaged 中如果发现当前连续的映射区间内有[超过 `khugepaged_max_ptes_none` 个页面是 pte_none 的](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L1643), 则将其合并为大页. 参见 [linux v2.6.38, mm/huge_memory.c#L1643](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L1643).


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/01/29 | Ebru Akagunduz <ebru.akagunduz@gmail.com> | [mm: incorporate read-only pages into transparent huge pages](https://lore.kernel.org/patchwork/patch/538503) | 允许 THP 转换只读 pte, 就像 do_swap_page 在读取错误后留下的那些 pte. 当在 2MB 范围内存在最多数量为 `khugepaged_max_ptes_none` 的 pte_none ptes 时, THP 可以将 4kB 的页面压缩为一个 THP. 这个补丁增加了对只读页面的大页支持. 该补丁使用一个程序进行测试, 该程序分配了 800MB 的内存, 对其进行写入, 然后休眠. 迫使系统通过触摸其他内存来交换除 190MB 外的所有程序. 然后, 测试程序对其内存进行混合的读写操作, 并将内存交换回来. 没有补丁的情况下, 只有没有被换出的内存保留在 THP 中, 这相当于程序内存的 24%, 这个百分比并没有随着时间的推移而增加. 有了这个补丁, 经过 5 分钟的等待, khugepageage 将 60% 的程序内存还原为 THP. | v4 ☑ [4.0-rc1](https://kernelnewbies.org/Linux_4.0#Memory_management) | [PatchWork v4](https://lore.kernel.org/patchwork/patch/538503), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=10359213d05acf804558bda7cc9b8422a828d1cd) |
| 2015/04/14 | Ebru Akagunduz <ebru.akagunduz@gmail.com> | [mm: incorporate zero pages into transparent huge pages](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ca0984caa8235762dc4e22c1c47ae6719dcc4064) | 通过大页允许零页面, 该补丁提高了 THP 的大页转换率. 目前, 当在 2MB 范围内存在最多数量为 khugepaged_max_ptes_none 的 pte_none ptes 时, THP 可以将 4kB 的页面压缩为一个 THP. 这个补丁支持了将零页映射为大页. 该补丁使用一个程序进行测试, 该程序分配了 800MB 的内存, 并执行交错读写操作, 其模式导致大约 2MB 的区域首先看到读访问, 从而导致影射了较多的零 pfn 映射. 没有补丁的情况下, 只有 50% 的程序被压缩成 THP, 并且百分比不会随着时间的推移而增加. 有了这个补丁, 等待 10 分钟后, khugepage 转换了 99% 的程序内存.  | v2 ☑ [4.1-rc1](https://kernelnewbies.org/Linux_4.1#Memory_management) | [LORE v2](https://lore.kernel.org/lkml/1423688635-4306-1-git-send-email-ebru.akagunduz@gmail.com), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ca0984caa8235762dc4e22c1c47ae6719dcc4064) |
| 2015/09/14 | Ebru Akagunduz <ebru.akagunduz@gmail.com> | [mm: make swapin readahead to gain more THP performance](https://lore.kernel.org/patchwork/patch/597392) | 支持在对匿名页 swapin 的时候 readahead 及逆行大页的转换.<br> 当 khugepaged 扫描页面时, 交换区中可能有一些页面. 有了这个补丁, 当 2MB 范围内 swap_ptes 的数目达到 max_ptes_swap 时, THP 可以将 4kB 页面转换成一个 THP 的大页.<br> 这个补丁用来处理那些在被调出内存后访问大部分 (但不是全部) 内存的程序的. 补丁合入后, 这些程序不会在内存换出 (swapout) 后将内存转换成到 THPs 中, 而会在内存从交换分区读入 (swapin) 时进行转换.<br> 测试使用了用一个测试程序, 该程序分配了 400B 的内存, 写入内存, 然后休眠. 然后强制将所有页面都换出. 之后, 测试程序通过对该内存区域进行写曹祖, 但是它在该区域的每 20 页中跳过一页.<br>1. 如果没有补丁, 系统就不能在 readahead 中交换. THP 率为程序内存的 65%, 不随时间变化.<br>2. 有了这个补丁, 经过 10 分钟的等待, khugepaged 已经崩溃了程序 99% 的内存. | v2 ☑ [4.8-rc1](https://kernelnewbies.org/Linux_4.8#Memory_management) | [PatchWork RFC,v5,0/3](https://lore.kernel.org/patchwork/patch/597392)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC,v5,0/3](https://lore.kernel.org/lkml/1442259105-4420-1-git-send-email-ebru.akagunduz@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LKML](https://lkml.org/lkml/2015/9/14/610) |
| 2022/05/10 | Yang Shi <shy828301@gmail.com> | [Make khugepaged collapse readonly FS THP more consistent](https://patchwork.kernel.org/project/linux-mm/cover/20220228235741.102941-1-shy828301@gmail.com/) | 618921 | v1 ☐☑ | [2022/02/28 LORE v1,0/8](https://lore.kernel.org/r/20220228235741.102941-1-shy828301@gmail.com)<br>*-*-*-*-*-*-*-* <br>[2022/05/10 LORE v4,0/8](https://lore.kernel.org/all/20220510203222.24246-1-shy828301@gmail.com) |

*   userspace hugepage collapse

khugepaged 默认速度较慢, 每 10 秒最多扫描 4096 页. 作为一个系统范围的设置, 这通常最通用的, 但是也可以有一些激进的方法, 可以让程序受益于. 除了为符合条件的内存范围添加优先级, 暂时加快整个系统的 khugepaged, 或者为属于某个进程的内存分片工作, 还有一种方法是允许用户空间进行 HugePage Collapse. 这种方法的好处是, 这是在进程上下文中完成的, 因此它的 CPU 被占用到执行 HugePage Collapse 的进程上. khugepaged 并不参与.

David Rientjes 率先提出了这种想法 [Hugepage collapse in process context](https://lore.kernel.org/all/d098c392-273a-36a4-1a29-59731cdf5d3d@google.com), 它的思路是允许用户空间通过新的 process_madvise() 调用来诱导 HugePage Collapse. 这允许我们为当前或另一个进程折叠大页. 当完成时, madvise() 将在正确的节点上分配一个大页, 并尝试在进程上下文中进行折叠.

对于已经使用 MADV_DONTNEED 将内存释放回系统并随后将内存故障的 malloc() 实现, 这将立即非常有用. 因为, 与其等待 khugepaged 在 30min 以后出现, 然后将这个内存折叠成一个大页(在一个非常大的系统上, 这可能会花费更长的时间), 还不如使用这个 process_madvise() 模式提前诱导这个动作. 换句话说," 我将这个内存返回给应用程序, 它将是热的, 所以现在返回一个大页面, 而不是等到以后 ". 它还可以用于用于文本段的只读文件支持映射. 还有一些额外的好处就是, khugepaged 要做的工作也减少了.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/02/16 | David Rientjes <rientjes@google.com> | [Hugepage collapse in process context](https://lore.kernel.org/all/d098c392-273a-36a4-1a29-59731cdf5d3d@google.com) | 允许用户空间通过新的 process_madvise() 调用来诱导 HugePage Collapse | RFC,v1 ☐ | [LORE RFC v1,00/14](https://lore.kernel.org/all/d098c392-273a-36a4-1a29-59731cdf5d3d@google.com) |
| 2022/06/04 | Zach O'Keefe <zokeefe@google.com> | [mm: userspace hugepage collapse](https://patchwork.kernel.org/project/linux-mm/cover/20220308213417.1407042-1-zokeefe@google.com/) | 这组补丁为用户空间提供了一种机制, 可以在进程上下文中将符合条件的内存范围压缩为 THP, 从而允许用户精细化地控制自己的 HugePages 使用策略. 借鉴了 David Rientjes 的想法. | v1 ☐☑ | [LORE v1,00/14](https://lore.kernel.org/r/20220308213417.1407042-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/12](https://lore.kernel.org/r/20220414180612.3844426-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/12](https://lore.kernel.org/r/20220426144412.742113-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/12](https://lore.kernel.org/r/20220502181714.3483177-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/12](https://lore.kernel.org/r/20220504214437.2850685-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/15](https://lore.kernel.org/r/20220604004004.954674-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v7,0/18](https://lore.kernel.org/r/20220706235936.2197195-1-zokeefe@google.com) |


*   其他

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/03/11 | maobibo <maobibo@loongson.cn> | [mm/khugepaged: sched to numa node when collapse huge page](https://patchwork.kernel.org/project/linux-mm/patch/20220311090119.2412738-1-maobibo@loongson.cn/) | 622550 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220311090119.2412738-1-maobibo@loongson.cn)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20220315040549.4122396-1-maobibo@loongson.cn) |
| 2022/05/27 | Jiaqi Yan <jiaqiyan@google.com> | [Memory poison recovery in khugepaged collapsing](https://patchwork.kernel.org/project/linux-mm/cover/20220527190731.322722-1-jiaqiyan@google.com/) | 645671 | v4 ☐☑ | [LORE v4,0/2](https://lore.kernel.org/r/20220527190731.322722-1-jiaqiyan@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/2](https://lore.kernel.org/r/20221107025359.2911028-1-jiaqiyan@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v7,0/2](https://lore.kernel.org/r/20221118013157.1333622-1-jiaqiyan@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v8,0/2](https://lore.kernel.org/r/20221201005931.3877608-1-jiaqiyan@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v9,0/2](https://lore.kernel.org/r/20221205234059.42971-1-jiaqiyan@google.com) |
| 2022/09/07 | Zach O'Keefe <zokeefe@google.com> | [mm: add file/shmem support to MADV_COLLAPSE](https://lore.kernel.org/all/20220907144521.3115321-1-zokeefe@google.com) | TODO | v3 ☐☑✓ | [2022/08/12 LORE 0/9](https://lore.kernel.org/linux-mm/20220812012843.3948330-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[2022/08/26 LORE v2,0/9](https://lore.kernel.org/linux-mm/20220826220329.1495407-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[2022/09/07 LORE v3,0/10](https://lore.kernel.org/all/20220907144521.3115321-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[2022/09/22 LORE v4,00/10](https://lore.kernel.org/all/20220922224046.1143204-1-zokeefe@google.com) |
| 2022/10/17 | Zach O'Keefe <zokeefe@google.com> | [Add MADV_COLLAPSE documentation](https://patchwork.kernel.org/project/linux-mm/cover/20221017175523.2048887-1-zokeefe@google.com/) | 685929 | v1 ☐☑ | [LORE v1,0/4](https://lore.kernel.org/r/20221017175523.2048887-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/4](https://lore.kernel.org/r/20221018235051.152548-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/4](https://lore.kernel.org/r/20221021223300.3675201-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/1](https://lore.kernel.org/r/20221031225500.3994542-1-zokeefe@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/1](https://lore.kernel.org/r/20221101150323.89743-1-zokeefe@google.com) |


### 7.2.5 THP splitting/reclaim/migration
-------

#### 7.2.5.1 splitting
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/07/31 | Yu Zhao <yuzhao@google.com> | [mm: optimize thp for reclaim and migration](https://lore.kernel.org/patchwork/patch/1470432) | 对 THP 的回收和迁移进行优化.<br> 使用配置 THP 为 always 的时候, 大量 THP 的内部碎片会造成不小的内存压力. 用户空间可以保留许多子页面不变, 但页面回收不能简单的根据 dirty bit 来识别他们.<brr> 但是, 仍然可以通过检查子页面的内容来确定它是否等同于干净. 当拆分 THP 用于回收或迁移时, 我们可以删除只包含 0 的子页面, 从而避免将它们写回或复制它们. | v1 ☐ | [PatchWork 0/3](https://patchwork.kernel.org/project/linux-mm/cover/20210731063938.1391602-1-yuzhao@google.com) |
| 2021/05/29 | Ning Zhang <ningzhang@linux.alibaba.com> | [mm, THP: introduce a controller to trigger THP reclaim](https://github.com/gatieme/linux/commit/57456d1625ba9036968fa0be70a6036b88f2b2f4) | 实现 MEMCG 的 THP 回收. [THP reclaim 功能](https://www.alibabacloud.com/help/zh/elastic-compute-service/latest/thp-reclaim). | v1 ☐ | [PatchWork 0/3](https://patchwork.kernel.org/project/linux-mm/cover/20210731063938.1391602-1-yuzhao@google.com) |
| 2018/01/03 | Michal Hocko <mhocko@kernel.org> | [unclutter thp migration](https://lore.kernel.org/patchwork/patch/869414) | THP 迁移以令人惊讶的语义侵入了通用迁移. 迁移分配回调应该检查 THP 是否可以立即迁移, 如果不是这样, 则分配一个简单的页面进行迁移. 取消映射和移动, 然后通过将 THP 拆分为小页面, 同时将标题页面移动到新分配的 order-0 页面来修复此问题. 剩余页面通过拆分页面移动到 LRU 列表. 如果 THP 分配失败, 也会发生同样的情况. 这真的很难看而且容易出错. | v1 ☐ | [PatchWork 0/3](https://lore.kernel.org/patchwork/patch/869414) |
| 2019/08/22 | Yang Shi <yang.shi@linux.alibaba.com> | [Make deferred split shrinker memcg aware](https://lore.kernel.org/patchwork/patch/869414) | 目前, THP 延迟分割收缩器不了解 memcg, 这可能会导致某些配置过早地处罚 OOM.<br> 通过引入每个 memcg 延迟拆分队列, 转换延迟拆分收缩器 memcg aware. THP 应位于每个节点或每个 memcg 延迟拆分队列上(如果它属于 memcg). 当页面迁移到另一个 memcg 时, 它也将迁移到目标 memcg 的延迟分割队列.<br> 对于每个 memcg 列表, 重复使用第二个尾页的延迟列表, 因为同一个 THP 不能位于多个延迟拆分队列上.<br> 使延迟分割收缩器不依赖于 memcg kmem, 因为它不是 slab. 过上述更改, 即使禁用了 memcg kmem(配置 cgroup.memory=nokmem), 测试也不会触发 OOM. | v6 ☑ 5.4-rc1 | [PatchWork v6,0/4](https://patchwork.kernel.org/project/linux-mm/cover/1566496227-84952-1-git-send-email-yang.shi@linux.alibaba.com) |
| 2018/01/03 | Michal Hocko <mhocko@kernel.org> | [unclutter thp migration](https://lore.kernel.org/patchwork/patch/869414) | THP 迁移以令人惊讶的语义侵入了通用迁移. 迁移分配回调应该检查 THP 是否可以立即迁移, 如果不是这样, 则分配一个简单的页面进行迁移. 取消映射和移动, 然后通过将 THP 拆分为小页面, 同时将标题页面移动到新分配的 order-0 页面来修复此问题. 剩余页面通过拆分页面移动到 LRU 列表. 如果 THP 分配失败, 也会发生同样的情况. 这真的很难看而且容易出错. | v1 ☐ | [PatchWork 0/3](https://lore.kernel.org/patchwork/patch/869414) |
| 2020/11/19 | Zi Yan <ziy@nvidia.com> | [Split huge pages to any lower order pages and selftests.](https://patchwork.kernel.org/project/linux-mm/cover/20201119160605.1272425-1-zi.yan@sent.com) | NA | v1 ☐ | [PatchWork 0/7](https://patchwork.kernel.org/project/linux-mm/cover/20201119160605.1272425-1-zi.yan@sent.com)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/5](https://lore.kernel.org/r/20220321142128.2471199-1-zi.yan@sent.com) |


#### 7.2.5.2 splitting test
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/01/15 | Kirill A. Shutemov <kirill.shutemov@linux.intel.com> | [thp: add debugfs handle to split all huge pages](https://www.spinics.net/lists/linux-mm/msg99353.html) | 将 1 写入 "split_huge_pages" 接口将尝试查找并拆分系统中的所有 THP. 这对于调试非常有用. | v1 ☑ 4.5-rc1 | [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=49071d436b51b58aeaf7abcd1877f38ca0146e31) |
| 2020/11/19 | Zi Yan <ziy@nvidia.com> | [mm: huge_memory: a new debugfs interface for splitting THP tests.](https://patchwork.kernel.org/project/linux-mm/cover/20201119160605.1272425-1-zi.yan@sent.com) | 提供了一个直接的用户接口来拆分 THP. 使用  `<debugfs>/split` 接受一个新命令来执行此操作. 通过将 "<pid>,<vaddr_start>,<vaddr_end>" 写入 "<debugfs>/split_huge_pages"将 分割具有给定 pid 的进程中给定虚拟地址范围内的 thp. 用于测试拆分页面功能. 此外, 还向 `tools/testing/selftests/vm` 添加了一个自检程序, 通过拆分 PMD THP 和 PTE 映射 THP 来利用该接口. 这不会改变旧的行为, 即向接口写入 1 仍然会拆分系统中的所有 THP. | v8 ☑ 5.13-rc1 | [PatchWork v8,1/2](https://patchwork.kernel.org/project/linux-mm/patch/20210331235309.332292-1-zi.yan@sent.com), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fa6c02315f745f00b62c634b220c3fb5c3310258) |



### 7.2.6 Reduce memory bloat
-------

首先是 HZP(huge zero page) 零页的支持.

[huge zero page vs FOLL_DUMP](https://linux-mm.kvack.narkive.com/Jrwys6ZE/huge-zero-page-vs-foll-dump)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2012/12/15 | Nitin Gupta <nitin.m.gupta@oracle.com> | [Introduce huge zero page](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=79da5407eeadc740fbf4b45d6df7d7f8e6adaf2c) | 参见 [Adding a huge zero page](https://lwn.net/Articles/517465). 在测试期间, 我注意到如果启用 THP, 某些工作负载 (例如 NPB 的 ft.A) 的内存消耗开销很大 (高达 2.5 倍). 造成这种巨大差异的主要原因是 THP 的情况下缺少零页面. 即使是读操作触发的 page-fault 也必须分配一个真正的页. 该补丁集通过引入巨页的零页面(hzp) 来解决这个问题, HZP 是一个不可移动的巨页(x86-64 上的 2M), 里面都是零.<br>1. 如果是读 fault 且 fault 的地址周围的区域适合使用 THP, 则在 do_huge_pmd_anonymous_page() 中设置它.<br>2. 如果设置 hzp (ENOMEM) 失败, 就回退到 handle_pte_fault(), 正常分配 THP.<br>3. 在对 hzp 的进行写操作触发 wp 时, 则为 THP 分配实际页面.<br>4. 如果是 ENOMEM, 则有一个优雅的回退: 创建一个新的 pmd 表, 并围绕故障地址将 pte 设置为新分配的正常 (4k) 页面. pmd 中的所有其他 pte 设置为正常零页.<br>5. 当前不能分割 hzp, 但可以分割指向它的 pmd. 在分割 pmd 时, 创建了一个所有 ptes 设置为普通零页的表. | v6 ☑ 3.8-rc1 | [PatchWork v6,00/12](https://lore.kernel.org/lkml/1353007622-18393-1-git-send-email-kirill.shutemov@linux.intel.com) |


当前, 如果启用 THP 的策略为 "always", 或模式为 "madvise" 且某个区域标记为 MADV_HUGEPAGE 时, 内核都会优先分配大页, 而即使 pud 或 pmd 为空. 这会产生最佳的 VA 翻译性能, 减少 TLB 冲突, 但是却增加了内存的消耗. 特别是但如果仅仅访问一些分散地零碎的小页面范围, 却仍然分配了大页, 造成了内存的浪费.


Oracle 的 Nitin Gupta 进行了最早的尝试, 参见 [mm: Reduce memory bloat with THP](https://lkml.org/lkml/2018/1/19/252). 这组补丁发到了 RFC v2 但是未被社区所认可.

Mel Gorman 提供了[另外一种新的思路](https://lkml.org/lkml/2018/1/25/571) :

1.  当在足够大的范围内出现第一个 page fault 时, 分配一个巨大的页面大小和对齐的基页块, 但是只映射与 page fault 地址对应的基本页, 并保留其余页.

2.  后续在这个范围内其他的页面出现 page fault 时, 则继续映射之前保留的页面, 同样只映射触发 page fault 的基本页.

3.  当映射了足够多的页面后, 将映射的页面和保留中的剩余页面升级为一个 THP.

4.  当内存压力较大时, 将 THP 拆分, 从而将未使用的页面从保留中释放出来.

Anthony Yznaga 接替了之前同事 Nitin Gupta 的工作, 并基于 Mel 的思路, 进行了尝试 [mm: THP: implement THP reservations for anonymous memory](https://lore.kernel.org/patchwork/patch/1009090).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/01/19 | Nitin Gupta <nitin.m.gupta@oracle.com> | [mm: Reduce memory bloat with THP](https://marc.info/?l=linux-mm&m=151631857310828&w=2) | NA | RFC,v2 ☐ | [PatchWork RFC,v2](https://lkml.org/lkml/2018/1/19/252) |
| 2018/11/09 | Anthony Yznaga <anthony.yznaga@oracle.com> | [mm: THP: implement THP reservations for anonymous memory](https://lore.kernel.org/patchwork/patch/1009090) | NA | RFC ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/1009090) |
| 2021/07/31 | Yu Zhao <yuzhao@google.com> | [mm: optimize thp for reclaim and migration](https://lore.kernel.org/patchwork/patch/1470432) | 对 THP 的回收和迁移进行优化.<br> 使用配置 THP 为 always 的时候, 大量 THP 的内部碎片会造成不小的内存压力. 用户空间可以保留许多子页面不变, 但页面回收不能简单的根据 dirty bit 来识别他们.<brr> 但是, 仍然可以通过检查子页面的内容来确定它是否等同于干净. 当拆分 THP 用于回收或迁移时, 我们可以删除只包含 0 的子页面, 从而避免将它们写回或复制它们. | v1 ☐ | [PatchWork 0/3](https://patchwork.kernel.org/project/linux-mm/cover/20210731063938.1391602-1-yuzhao@google.com) |
| 2021/10/28 | Ning Zhang <ningzhang@linux.alibaba.com> | [Reclaim zero subpages of thp to avoid memory bloat](https://patchwork.kernel.org/project/linux-mm/cover/1635422215-99394-1-git-send-email-ningzhang@linux.alibaba.com) | 这个补丁集引入了一种新的机制来分割没有子页面的大页并回收这些子页面. thp 可能会导致内存膨胀, 从而导致 OOM. 通过对一些应用程序的测试, 作者发现内存膨胀的原因是一个巨大的页面可能包含一些零子页面(可能被访问或没有). 而且大多数零子页面集中在几个大页面中.<br> 通过将匿名大页面添加到列表中, 以减少查找大页面的成本. 当内存回收被触发时, 列表将被遍历, 包含足够的零子页面的大页面可能会被回收. 同时, 用 ZERO_PAGE(0) 替换零子页面. 之前 Yu Zhao 已经做了一些类似的工作 [PatchWork 0/3](https://patchwork.kernel.org/project/linux-mm/cover/20210731063938.1391602-1-yuzhao@google.com), 当巨大的页面被交换或迁移来加速. 当我们在 swap 场景的正常内存收缩路径中这样做时, 以避免 OOM. 未来, 作者可能将尝试主动回收 "冷" 大页面. 为了尽可能在保持 thp 性能的同时. 使得开启了 thp 的内存使用量等于使用普通页的内存使用量. | RFC ☐ | [PatchWork RFC,0/6](https://patchwork.kernel.org/project/linux-mm/cover/1635422215-99394-1-git-send-email-ningzhang@linux.alibaba.com) |


### 7.2.7 THP Page Cache
-------

参见 [Linux 中的 Memory Compaction [三] - THP](https://zhuanlan.zhihu.com/p/117239320)

早期的 THP 还只支持匿名页(Anonymous Pages), 而不支持 Page Cache, 这是因为:

1. 匿名页 (Anonymous Pages) 通常只能通过 [mmap 映射](https://zhuanlan.zhihu.com/p/71517406)访问, 而 Page Cache 还可以通过直接调用 read() 和 write() 访问, huge page 区别于 normal page 的体现就是少了一级 (或者几级) 页表转换, 不通过映射访问的话, 对 huge page 的使用就比较困难.

2.  如果使用 THP 的话, 需要文件本身足够大, 才能充分利用 huge page 带来的好处, 而现实中大部分的文件都是比较小的, 参见 [Transparent huge pages in the page cache](https://lwn.net/Articles/686690).

不过在某些场景下, 让 THP 支持 page cache 的需求还是存在的.

[dhowells/linux-fs: fscache-thp](https://git.kernel.org/pub/scm/linux/kernel/git/dhowells/linux-fs.git/log/?h=fscache-thp)

[Transparent huge pages for filesystems](https://lwn.net/Articles/789159), 译文 [LWN 回顾: facebook 利用 transparent huge page 来优化代码执行性能](https://blog.csdn.net/Linux_Everything/article/details/103790440).

#### 7.2.7.1 THP RAMFS
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2013/05/12 | "Kirill A. Shutemov" <kirill.shutemov@linux.intel.com> | [Transparent huge page cache](https://lore.kernel.org/patchwork/patch/378803) | 这组补丁集为内核页面缓存增加了巨页的支持. 为了证明建议的更改是功能性的, 为最简单的文件系统 ramfs 启用了该特性. 每个 THP 在页面缓存基数树中由 HPAGE_PMD_NR(x86-64 上的 512)条目表示: 一个条目用于首页, HPAGE_PMD_NR-1 条目用于尾页. 可以通过三种方式将大型页面添加到页面缓存:<br>1. 向文件或页面写入;<br>2. 从稀疏文件中读取;<br>3. 稀疏文件上触发 page_fault.<br> 当然可能存在第三种方法可能是折叠小页面, 但它超出了最初的实现范围. | v4 ☐ | [PatchWork v4,00/39](https://lore.kernel.org/patchwork/patch/378803) |
| 2013/09/23 | "Kirill A. Shutemov" <kirill.shutemov@linux.intel.com> | [Transparent huge page cache: phase 1, everything but mmap()](https://lore.kernel.org/patchwork/patch/408000) | 这组补丁集为内核页面缓存增加了巨页的支持. 为了证明建议的更改是功能性的, 为最简单的文件系统 ramfs 启用了该特性, 但是 mmap 除外, 这将在后续的合入中修复. | v6 ☐ | [PatchWork v6,00/22](https://lore.kernel.org/patchwork/patch/408000) |


#### 7.2.7.2 THP TMPFS/SHMEM
-------


随后社区不少开发者在做 (TMPFS) Page Cache 的 THP 支持. 参见 [Two transparent huge page cache implementations](https://lwn.net/Articles/684300) 所述:

1.  一个解决方案是 Kirill Shutemov 提出的, 他完成了 THP 启用 tmpfs 补丁集. Kirill 的工作基于社区已有的[复合页面 compound pages](https://lwn.net/Articles/619514), 这是一种相当精细的机制, 可以将单个内存页面绑定到一个更大的页面上. 作者个人仓库 [`kas/linux.git`](https://git.kernel.org/pub/scm/linux/kernel/git/kas/linux.git).

2.  另一个竞争者是由 Hugh Dickins 实现的团队页面 (team pages) 方案. "团队页面" 可以被认为是一种新的、更轻量级的页面分组方式. 它的工作有一个优势, 据作者所言, 这个方案已经在谷歌的产品部署了一年多, 效果和稳定性都有保障.

社区开发者争相选择 tmpfs 入手是因为作为一个文件系统, 它很特殊, 从某种意义上说, 它不算一个 "货真价实" 的文件系统. 但它这种模棱两可的特性正好是一个绝佳的过渡, 实现了 THP 对 tmpfs 的支持之后, 可以进一步 [推广到 ext4 这种标准的磁盘文件系统](https://lwn.net/Articles/718102) 中去, 但......, 还很多问题要解决, 比如磁盘文件系统的 readahead 机制, [预读窗口通常是 128KB](https://zhuanlan.zhihu.com/p/71217136), 这远小于一个 huge page 的大小. 参见 LSFMM 2017 的报道 [Huge pages in the ext4 filesystem](https://lwn.net/Articles/718102)


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/02/20 | Hugh Dickins <hughd@google.com> | [huge tmpfs: an alternative approach to THPageCache](https://lwn.net/Articles/634334) | 为 tmpfs 文件系统增加了透明的巨大页面支持.<br> 这项工作包含了页面缓存中完全支持大页面所需的许多元素(这也是 Kirill 补丁的最终目标). 但是 Hugh 的方法相当不同, 这引起了用户社区的一些担忧; 最终, 这些补丁集中只有一个可能被合并.<br> 参见 [Improving huge page handling](https://lwn.net/Articles/636162) | v4 ☐ | [PatchWork 00/24](https://lore.kernel.org/lkml/alpine.LSU.2.11.1502201941340.14414@eggly.anvils) |
| 2016/02/20 | Hugh Dickins <hughd@google.com> | [huge tmpfs: THPagecache implemented by teams](https://lwn.net/Articles/682623) | 为 tmpfs 文件系统增加了透明的巨大页面支持.<br> 这项工作包含了页面缓存中完全支持大页面所需的许多元素(这也是 Kirill 补丁的最终目标). 但是 Hugh 的方法相当不同, 这引起了用户社区的一些担忧; 最终, 这些补丁集中只有一个可能被合并.<br> 参见 [Improving huge page handling](https://lwn.net/Articles/636162) | v4 ☐ | [PatchWork 00/24](https://lore.kernel.org/lkml/alpine.LSU.2.11.1502201941340.14414@eggly.anvils) |

经过多次改进和优化后, 最终 Kirill Shutemov 的基于复合页的方案 [THP-enabled tmpfs/shmem (using compound pages)](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=1b5946a84d6eb096158e535bdb9bda06e7cdd941) 被合入主线 [4.8](https://kernelnewbies.org/Linux_4.8#Support_for_using_Transparent_Huge_Pages_in_the_page_cache) 版本.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/06/15 | "Kirill A. Shutemov" <kirill.shutemov@linux.intel.com> | [THP-enabled tmpfs/shmem using compound pages](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=1b5946a84d6eb096158e535bdb9bda06e7cdd941) | 支持 tmpfs 和 shmem page cache 的透明大页支持.<br>1. 引入了 [CONFIG_TRANSPARENT_HUGE_PAGECACHE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e496cf3d782135c1cca0d154d4b924517ff58de0).<br>2. 实现了 [shmem page cache 大页](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=800d8c63b2e989c2e349632d1648119bf5862f01)的支持.<br>3. 完成了 khugepaged [对 tmpfs/shmem page cache 页面合并](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f3f0e1d2150b2b99da2cbdfaad000089efe9bf30)的支持. | v9 ☑ [4.8-rc1](https://kernelnewbies.org/Linux_4.8#Support_for_using_Transparent_Huge_Pages_in_the_page_cache) | [PatchWork v4,00/25](https://lore.kernel.org/patchwork/patch/658181)<br>*-*-*-*-*-*-*-* <br>[PatchWork v9,00/32](https://lore.kernel.org/lkml/1465222029-45942-1-git-send-email-kirill.shutemov@linux.intel.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v9-rebased2,00/37](https://lore.kernel.org/lkml/1466021202-61880-1-git-send-email-kirill.shutemov@linux.intel.com)  |


#### 7.2.7.3 THP EXT4
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/07/26 | "Kirill A. Shutemov" <kirill.shutemov@linux.intel.com> | [ext4: support of huge pages](https://lore.kernel.org/patchwork/patch/701302) | 这是我的补丁集的第一个版本, 它旨在将 THO 带到 ext4, 它还没有准备好应用或正式使用, 但足以展示该方法.<br> 基本原理与 tmpfs 相同, 并构建在它的基础上. 主要区别在于, 我们需要处理从备份存储器的读取和回写. | RFC,v1 ☐ | [PatchWork v1,RFC,00/33](https://lore.kernel.org/patchwork/patch/701302) |
| 2019/07/17 | Yang Shi <yang.shi@linux.alibaba.com> | [Fix false negative of shmem vma's THP eligibility](https://lore.kernel.org/patchwork/patch/1101422) | 提交 7635d9cbe832 ("mm, THP, proc: report THP qualified for each vma") 为进程的 map 引入了 THPeligible bit. 但是, 当检查 shmem vma 的资格时, `__transparent_hugepage_enabled()` 被调用以覆盖来自 shmem_huge_enabled() 的结果. 它可能导致匿名 vma 的 THP 标志覆盖 shmem 的.<br> 通过使用 transhuge_vma_suitable() 来检查 vma, 来修复此问题 | v4 ☑ 5.3-rc1 | [PatchWork v4,0/2](https://lore.kernel.org/patchwork/patch/1101422) |
| 2021/07/30 | Hugh Dickins <hughd@google.com> | [tmpfs: HUGEPAGE and MEM_LOCK fcntls and memfds](https://patchwork.kernel.org/project/linux-mm/cover/2862852d-badd-7486-3a8e-c5ea9666d6fb@google.com) | 一系列 HUGEPAGE 和 MEM_LOCK tmpfs fcntls 和 memfd_create 标志的清理和修复工作. | v1 ☐ | [PatchWork 00/16](https://patchwork.kernel.org/project/linux-mm/cover/2862852d-badd-7486-3a8e-c5ea9666d6fb@google.com) |

#### 7.2.7.4 non-tmpfs filesystems
-------

Matthew Wilcox 在这方面也做了很多工作.
代码仓库参见 : http://git.infradead.org/users/willy/pagecache.git

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/06/10 | Matthew Wilcox <willy@infradead.org> | [Large pages in the page cache](https://lore.kernel.org/patchwork/patch/1254710) | NA | RFC v6 ☐  | [PatchWork RFC,v6,00/51](https://patchwork.kernel.org/project/linux-mm/cover/20200610201345.13273-1-willy@infradead.org) |
| 2020/06/29 | Matthew Wilcox <willy@infradead.org> | [THP prep patches](https://patchwork.kernel.org/project/linux-mm/cover/20200629151959.15779-1-willy@infradead.org) | NA | v1 ☑ 5.9-rc1 | [PatchWork 0/7](https://patchwork.kernel.org/project/linux-mm/cover/20200629151959.15779-1-willy@infradead.org) |
| 2020/09/10 | Matthew Wilcox <willy@infradead.org> | [THP iomap patches for 5.10](https://lore.kernel.org/patchwork/patch/1304097) | NA | v2 ☑ 5.10-rc1 | [PatchWork v2,0/9](https://lore.kernel.org/patchwork/patch/1304097) |
| 2020/09/16 | Matthew Wilcox <willy@infradead.org> | [fs: Add a filesystem flag for THPs](https://lore.kernel.org/patchwork/patch/1306507) | NA | v1 ☑ 5.10-rc1 | [PatchWork 1/2](https://lore.kernel.org/patchwork/patch/1306507) |
| 2020/08/24 | Matthew Wilcox <willy@infradead.org> | [iomap/fs/block patches for 5.11](https://lore.kernel.org/patchwork/patch/1294511) | NA | v1 ☑ 5.11-rc1 | [PatchWork 00/11](https://lore.kernel.org/patchwork/patch/1294511) |
| 2020/10/26 | Matthew Wilcox <willy@infradead.org> | [Remove nrexceptional tracking](https://lore.kernel.org/patchwork/patch/1294511) | NA | v1 ☐ | [PatchWork 00/11](https://patchwork.kernel.org/project/linux-mm/cover/20201026151849.24232-1-willy@infradead.org) |
| 2020/11/12 | "Matthew Wilcox (Oracle)" <willy@infradead.org> | [Overhaul multi-page lookups for THP](https://lore.kernel.org/patchwork/patch/1337675) | 提升大量页面查找时的效率 | v4 ☐ | [PatchWork v4,00/16](https://patchwork.kernel.org/project/linux-mm/cover/20201112212641.27837-1-willy@infradead.org) |
| 2020/10/29 | "Matthew Wilcox (Oracle)" <willy@infradead.org> | [Transparent Hugepages for non-tmpfs filesystems](https://patchwork.kernel.org/project/linux-mm/cover/2862852d-badd-7486-3a8e-c5ea9666d6fb@google.com) | 一系列 HUGEPAGE 和 MEM_LOCK tmpfs fcntls 和 memfd_create 标志的清理和修复工作. | v1 ☐ | [PatchWork 00/16](https://patchwork.kernel.org/project/linux-mm/cover/20201029193405.29125-1-willy@infradead.org) |


### 7.2.8 Huge Page(THP) SWAP
-------


THP 和 hugetlb 看起来样子差不多, 但在 Linux 中的归属和行为却完全不同. 后者一体成型, 而前者更像是焊接起来的. THP 虽然勉强拼凑成了 huge page 的模样, 但骨子里还是 normal page, 还和 normal page 同处一个世界.

在它诞生之初, 面对这个庞然大物, 既有的内存管理子系统的机制还没做好充分的应对准备, 比如 THP 要 swap 的时候怎么办啊? 这个时候, 只能调用 split_huge_page(), 将 THP 重新打散成 normal page.

swap out 的时候打散, swap in 的时候可能又需要重新聚合回来, 这对性能的影响是不言而喻的. 一点一点地找到空闲的 pages, 然后辛辛苦苦地把它们组合起来, 现在到好, 一切都白费了(路修了又挖, 挖了又修……). 虽然动态地生成 huge page 确实能更充分利用物理内存, 但其带来的收益, 有时还真不见得能平衡掉这一来一去的损耗.

不过呢, 内核开发者也在积极努力, 希望能够实现 THP 作为一个整体被 swap out 和 swap in(参考这篇文章), 但这算是对 “牵一发而动全身” 的内存子系统的一次重大调整, 所以更多的 regression 测试还在进行中.

可见啊, THP 并没有想象的那么美好, 用还是不用, 怎么用, 就成了一个需要思考和选择的问题.

[The final step for huge-page swapping](https://lwn.net/Articles/758677)

#### 7.2.8.1 optimize the performance of Transparent
-------

后来随着存储设备的性能提升, 普通页面换入换出已无法使磁盘带宽达到饱和. 而另一方面内存容量不断增加, THP 的使用已经越来越广泛. 因此有必要对 THP 交换性能进行优化.


引入 THP SWAP 是有诸多好处的:

1. 批量处理 THP 的交换操作以减少锁的获取 / 释放, 包括分配 / 释放交换空间, 增加 / 删除交换缓存, 读写交换空间等. 这将有助于提高 THP 交换的性能.

2. THP 交换空间的读写最小也是 2M 的顺序 IO(sequential IO). 而 swap read 交换读取通常是 4k 随机 IO. 这也将提高 THP 交换的性能.

3. 使用 THP swap 也有助于减少内存碎片, 特别是对 THP 被大量使用的应用程序. 在 THP 被 swap out 之后, 至少可以释放 2M 的连续页面.

4. 使能 swap 也能有效地将提高系统的 THP 利用率. 因为 khugepaged 将普通页面折叠成 THP 的速度相当慢. 如果在换出时对 THP 进行拆分, 那么再次换入后普通页面需要相当长的时间才能再次折叠成 THP. 较高 THP 利用率也有助于提高基于页面的内存管理的效率.

但是引入 THP SWAP 后, 必然增大换入换出时读 / 写的 IO 大小, 这可能会给存储设备带来更多的开销. 因此为了解决这个问题, THP 交换应该只在必要的时候打开. 例如, 它可以通过 "always/never/madvise" 逻辑来选择, 全局打开, 全局关闭, 或者仅对 VMA 使用 MADV_HUGEPAGE 打开, 等等.


*   Delay splitting THP during swapping out


支持 THP SWAP 的第一步是逐步延迟拆分 THP, 最终避免在 THP 交换期间拆分 THP, 并在整个 THP 中进行交换. [LWN 717707: 页交换 (swap) 的改进计划](https://tinylab.org/lwn-717707).

[Linux 5.20 To Enable THP SWAP On 64-bit Arm For Better Swapping Performance](https://www.phoronix.com/news/Linux-5.20-THP-SWAP-ARM64)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/05/15 | "Huang, Ying" <ying.huang@intel.com> | [THP swap: Delay splitting THP during swapping out/STEP 1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=747552b1e71b400fa32a221e072be2c0b7661f14) | 在这个补丁集完成了延迟拆分 THP, 将拆分大页面的时间几乎从交换的第一步推迟到为 THP 分配交换空间并将 THP 添加到交换缓存之后. 这将减少交换缓存管理中使用的锁的获取 / 释放.<br> 在 8 个进程的 vm-scalability swap-w-seq 测试用例 (测试用例创建了 8 个进程, 它们依次分配和写入匿名页面, 直到 RAM 和交换设备的一部分用完) 中, 使用补丁集, 换出的吞吐量提高了 15.5%(从 3.73GB/s 提高到 4.31GB/s). | v11 ☑ 4.13-rc1 | [2016/08/09 LORE RFC,00/11](https://lore.kernel.org/lkml/1470760673-12420-1-git-send-email-ying.huang@intel.com)[2016/09/01 LORE v2,00/10](https://lore.kernel.org/lkml/1472743023-4116-1-git-send-email-ying.huang@intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,00/12](https://lore.kernel.org/all/20170724051840.2309-1-ying.huang@intel.com)<br>*-*-*-*-*-*-*-* <br>[2017/05/15 LORE v11,0/5](https://lore.kernel.org/lkml/20170515112522.32457-1-ying.huang@intel.com) |
| 2017/07/14 | "Huang, Ying" <ying.huang@intel.com> | [THP swap: Delay splitting THP during swapping out/STEP 2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=fe490cc0fe9e6ee48cc48bb5dc463bc5f0f1428f) | 这是 THP (透明大页) 交换优化的 STEP 2. 在 STEP 1, 拆分大页面的时机从交换的第一步延迟到为 THP 分配交换空间并将 THP 添加到交换缓存之后. 在 STEP 2 中, 拆分被进一步延迟到交换完成之后. 在这个补丁集中, 对匿名 THP 回收的更多操作(如 TLB 刷新、将 THP 写入交换设备、将 THP 从交换缓存中删除) 被批处理. 从而提高了匿名 THP 交换的性能. | v3 ☑ 4.14-rc1 | [2017/06/23 LORE v2,00/12](https://lore.kernel.org/lkml/20170623071303.13469-1-ying.huang@intel.com)<br>*-*-*-*-*-*-*-* <br>[2017/07/24 LORE v3,00/12](https://lore.kernel.org/all/20170724051840.2309-1-ying.huang@intel.com) |
| 2022/05/27 | Barry Song <21cnbao@gmail.com> | [[v2] arm64: enable THP_SWAP for arm64](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d0637c505f8a1d8c4088642f1f3e9e3b22da14f6) | 645555 | v2 ☐☑ | [LORE v2](https://lore.kernel.org/r/20220527100644.293717-1-21cnbao@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3](https://lore.kernel.org/r/20220706072707.114376-1-21cnbao@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/1](https://lore.kernel.org/r/20220720093737.133375-1-21cnbao@gmail.com) |


*   Swapout/swapin THP in one piece

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/12/14 | "Huang, Ying" <ying.huang@intel.com> | [swap: Swapout/swapin THP in one piece](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=fe490cc0fe9e6ee48cc48bb5dc463bc5f0f1428f) | NA | v9 ☐ 4.14-rc1 | [2018/05/09 LORE v2,00,21](https://lore.kernel.org/lkml/20180509083846.14823-3-ying.huang@intel.com)<br>*-*-*-*-*-*-*-* <br>[2018/12/14 LORE v9,00/21](https://lore.kernel.org/lkml/20181214062754.13723-1-ying.huang@intel.com), [PatchWork v9,00/21](https://patchwork.kernel.org/project/linux-mm/cover/20181214062754.13723-1-ying.huang@intel.com) |


### 7.2.9 THP khugepaged
-------

#### 7.2.9.1 khugepaged 概述
-------

khugepaged 是透明巨页的守护进程, 它会被定时唤醒, 并根据配置尝试将 page_size(比如 4K) 大小的普通 page 转成巨页, 减少 TLB 压力, 提高内存使用效率.

khugepaged 的处理过程中需要各种内存锁, 会在一定程度上影响系统性能.

khugepaged 主要的调用链如下:

```cpp
khugepaged
-=> khugepaged_do_scan()
    -=> khugepaged_scan_mm_slot()
        -=> khugepaged_scan_pmd()
            -=> collapse_huge_page()
                -=> __collapse_huge_page_copy()
```


khugepaged 控制参数, 位于 `/sys/kernel/mm/transparent_hugepage/khugepaged` 目录下:

| 节点 | 描述 |
|:---:|:----:|
| pages_collapsed | 记录了 khugepaged 已经[合并的大页的数目](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L1902) |
| pages_to_scan | 每次 khugepaged 尝试[处理的页面数目](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L2146), khugepage 扫描是有一定开销的, 因此对扫描的范围做一些限制 |
| full_scans | 完整扫描的轮数. 每次如果 khugepaged 如果完整的扫描了某个 mm_struct 的所有 VMA, 则[该计数增加 1](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L2118).  |
| scan_sleep_millisecs | 扫描间隔(毫秒) |
| alloc_sleep_millisecs | 当分配大内存失败等待多少毫秒来压制下一个大内存页的分配尝试. |

khugepaged 处理流程

| 关键流程 | 描述 |
|:-------:|:---:|
| khugepaged_do_scan | khugepaged 被唤醒后, 尝试处理 khugepaged_pages_to_scan 个页面. 其中每一次页面的循环这是通过调用 khugepaged_scan_mm_slot() 完成. 在 khugepaged_do_scan() 每次循环的最开始, 会尝试[调用 prealloc page 预分配一个巨页](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L2151), 需要注意. 这主要针对非 numa 系统, 对于 numa 系统, 这里不会真正分配. |
| khugepaged_scan_mm_slot | 扫描 struct mm_struct , 找到可以用巨页代替的普通页面. 这里需要扫描的 struct mm_struct 是放到 [全局变量 khugepaged_scan 中的 mm_slot](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L93). 而这些 mm_slot 中的这些 mm 是在 vma 结构体发生[合并 merge](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L1556) 和[扩展](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L678)的时候调用 [khugepaged_enter()](https://elixir.bootlin.com/linux/v2.6.38/source/include/linux/khugepaged.h#L38) 加入的. 这里找到需要扫描的 mm_struct 之后, 扫描 mm_struct 中包含的[各个 vma](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L2039), 注意, 扫描的范围是一个巨页包括的范围. |
| [khugepaged_scan_pmd](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L1915) | 完成普通页面搬运到巨页的工作. 具体来说, khugepaged_scan_pmd() 通过访问 vma 中的每一个页面, 然后判断该页面是否可以用巨页代替, 可以的话[调用 collapse_huge_page() 处理](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L1982). |
| collapse_huge_page | 首先[通过 `__collapse_huge_page_copy` 将普通页面拷贝到巨页中](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L1872), 然后释放普通页面. 不过释放之前会将原来页面的 tlb 刷出去, 然后将原来的页面从 LRU 链表移除, 最后拷贝到巨页中后, 还会更新内存相关信息. 如果是 numa 系统, 这里才会[真正分配巨页](https://elixir.bootlin.com/linux/v2.6.38/source/mm/huge_memory.c#L2151). |

#### 7.2.9.2 khugepaged 的引入和优化
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2011/01/13 | Andrea Arcangeli <aarcange@redhat.com> | [thp: khugepaged](https://lkml.org/lkml/2010/12/23/306) | 实现了一个 khugepaged 的内核线程, 用来完成将标准的小页面合并成大页的操作. | v1 ☑ 2.6.38-rc1 | [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ba76149f47d8c939efa0acc07a191237af900471) |
| 2015/09/14 | Ebru Akagunduz <ebru.akagunduz@gmail.com> | [mm: add tracepoint for scanning pages](https://lkml.org/lkml/2015/9/14/611) | [mm: make swapin readahead to gain more THP performance](https://lore.kernel.org/lkml/1442259105-4420-1-git-send-email-ebru.akagunduz@gmail.com) 系列的其中一个补丁, 为 khuagepaged 引入了 tracepoint 跟踪点. 用 scan_result 标记了 khugepaged 扫描的结果. | v5 ☑ 4.5-rc1 | [PatchWork RFC,v5,0/3](https://lore.kernel.org/lkml/1442259105-4420-2-git-send-email-ebru.akagunduz@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LKML](https://lkml.org/lkml/2015/9/14/611)<br>*-*-*-*-*-*-*-* <br>[commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7d2eba0557c18f7522b98befed98799990dd4fdb) |
| 2011/01/13 | Peter Xu <peterx@redhat.com> | [mm/khugepaged: Detecting uffd-wp vma more efficiently](https://patchwork.kernel.org/project/linux-mm/patch/20210922175156.130228-1-peterx@redhat.com) | 实现了一个 khugepaged 的内核线程, 用来完成将标准的小页面合并成大页的操作. | ☑ 2.6.38-rc1 | [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ba76149f47d8c939efa0acc07a191237af900471) |

*   Improve THP utilizationn

| 时间 | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:---:|:----:|:---:|:----:|:---------:|:----:|
| 2022/08/05 | alexlzhu@fb.com <alexlzhu@fb.com> | [mm: add thp_utilization metrics to /proc/thp_utilization](https://lore.kernel.org/all/20220805184016.2926168-1-alexlzhu@fb.com) | 由于性能的提高或降低取决于特定应用程序如何使用物理内存, THP 在历史上一直是针对每个应用程序启用的. 当 THP 被大量利用时, 由于 TLB 缓存失败的减少, 应用程序性能会得到改善. 长期以来, 人们一直怀疑启用 THP 时的性能下降是由于大量未充分利用的匿名 THP 造成的. 以前, 没有办法跟踪到底有多少 THP 被实际使用. 通过这个补丁, 帮助开发者了解 THP 的使用情况, 以便在分页方面做出更智能的决策. 这个更改引入了一个工具, 该工具扫描匿名 THP 的所有物理内存, 并根据使用率将它们分组到桶中. 它还包括一个位于 `/sys/kernel/debug/thp_utilization` 下的接口. THP 的利用率定义为 THP 中非零页面的百分比. 工作线程将扫描所有物理内存, 并获得所有匿名 THP 的利用率. 它将通过定期扫描所有物理内存来收集这些信息, 寻找匿名 THP, 根据利用率将它们分组到桶中, 并通过 `/sys/kernel/debug/thp_utilization` 下的 debugfs 报告利用率信息. | v3 ☐☑✓ | [LORE v2](https://lore.kernel.org/lkml/20220809014950.3616464-1-alexlzhu@fb.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3](https://lore.kernel.org/all/20220805184016.2926168-1-alexlzhu@fb.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/1](https://lore.kernel.org/r/20220818000112.2722201-1-alexlzhu@fb.com) |

*   Improve awareness for exiting processe

| 时间 | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:---:|:----:|:---:|:----:|:---------:|:----:|
| 2022/09/21 | Rongwei Wang <rongwei.wang@linux.alibaba.com> | [[RFC] mm/khugepaged: Improve awareness for exiting processes](https://patchwork.kernel.org/project/linux-mm/patch/20220921032759.41473-1-rongwei.wang@linux.alibaba.com/) | 678876 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220921032759.41473-1-rongwei.wang@linux.alibaba.com) |

#### 7.2.9.3 khugepaged 的其他优化
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/10/24 | Gautam Menghani <gautammenghani201@gmail.com> | [mm/khugepaged: add tracepoint to collapse_file()](https://patchwork.kernel.org/project/linux-mm/patch/20221024150922.129814-1-gautammenghani201@gmail.com/) | 688257 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221024150922.129814-1-gautammenghani201@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/1](https://lore.kernel.org/r/20221202201807.182829-1-gautammenghani201@gmail.com) |
| 2022/10/25 | Nathan Chancellor <nathan@kernel.org> | [mm/khugepaged: Initialize index and nr in collapse_file()](https://patchwork.kernel.org/project/linux-mm/patch/20221025173407.3423241-1-nathan@kernel.org/) | 688749 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221025173407.3423241-1-nathan@kernel.org) |
| 2022/10/26 | Johannes Weiner <hannes@cmpxchg.org> | [[v2] mm: vmscan: split khugepaged stats from direct reclaim stats](https://patchwork.kernel.org/project/linux-mm/patch/20221026180133.377671-1-hannes@cmpxchg.org/) | 689123 | v2 ☐☑ | [LORE v2,0/1](https://lore.kernel.org/r/20221026180133.377671-1-hannes@cmpxchg.org) |


## 7.3 复合页 Compound Page
-------

### 7.3.1 gigantic compounds pages
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2003/02/05 | Andrew Morton <akpm@digeo.com> | [Infrastructure for correct hugepage refcounting](https://github.com/gatieme/linux-history/commit/eefb08ee7da81e1548ffd5b664682dc5b229ddc2) | 实现了复合页(Compound Page), 如果用户的地址在一个大页面中, 设置页面的 refcount 是一个有歧义的事情, 我们应该设置的是组成这个大页的 4K 普通页, 而不是高阶分配单元的头页. 为了方便处理, 实现了一种处理高阶页面的通用方式:<br>1. 高阶页面称为 "复合页面", 复合页面的第一个 (控制) 4k 页面称为 "head" 页面, 剩余页面为尾页.<br>2. 所有复合页面都有 PG_compound 集合, 所有页面都有自己的 lru, 尾页面都指向头页面.<br>3. 紧接着就完成了 [convert hugetlb code to use compound pages](https://github.com/gatieme/linux-history/commit/b3a656b6d36622e628974bad5cf19c006c395efe) | v1 ☑ 2.5.60~33 | [COMMIT](https://github.com/gatieme/linux-history/commit/eefb08ee7da81e1548ffd5b664682dc5b229ddc2) |
| 2007/10/04 | Christoph Lameter <clameter@sgi.com> | [Virtual Compound Page Support V2](https://lore.kernel.org/patchwork/patch/93090) | NA | ☑ 4.11-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/93090) |
| 2008/10/23 | Andy Whitcroft <apw@shadowen.org> | [Fixes for gigantic compounds pages V3](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=18229df5b613ed0732a766fc37850de2e7988e43) | NA | v1 ☑✓ 2.6.28-rc4 | [LORE v1](https://lore.kernel.org/all/1223458499-12752-1-git-send-email-apw@shadowen.org)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/2](https://lore.kernel.org/all/1224771559-19363-1-git-send-email-apw@shadowen.org) |

### 7.3.2  Multiple consecutive page
-------

4k 太小, 2M 太大. 是否可以在中间做些什么. 这个系列展示了中间立场可能是什么样子. 它提供了 THP 的一些优点, 同时消除了一些缺点. Multiple Consecutive Page 使用 8K 到 2M 个基本页的 "多个连续页"(mcpages)进行匿名用户空间映射. 与 2M 映射相比, 这将导致更少的内部碎片, 从而减少内存消耗和浪费 CPU 时间归零内存, 而这些内存将永远不会被使用.

在实现中, 我们以 mcpage 的顺序分配高阶页面(例如, 16KB mcpage 的 2 阶). 这确保使用了物理连续内存, 并有利于顺序内存访问延迟.

然后拆分高阶页面. 通过这样做, mcpage 的子页面只是 4K 普通页面. 当前的内核页面管理应用于 "mc" 页面, 没有任何更改. mcpage 允许批处理页面错误, 并减少页面错误数.

mcpage 有成本. 除了 THP 没有带来 TLB 的好处之外, 与 4K 基本页相比, 它增加了内存消耗和分配页的延迟.

本系列是 mcpage 的第一步. 未来的工作可以为更多组件启用 mcpage, 如页面缓存、交换等. 最后, 系统中的大多数页面将按 mcpage 顺序分配 / 释放 / 回收.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2023/01/09 | Yin, Fengwei <fengwei.yin@intel.com> | [Multiple consecutive page for anonymous mapping](https://patchwork.kernel.org/project/linux-mm/cover/20230109072232.2398464-1-fengwei.yin@intel.com/) | 709978 | v1 ☐☑ | [LORE v1,0/4](https://lore.kernel.org/r/20230109072232.2398464-1-fengwei.yin@intel.com) |



# 8 进程虚拟地址空间(VMA)
-------

## 8.1 VMA
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2019/12/19 | Colin Cross | [mm: add a field to store names for private anonymous memory](https://lore.kernel.org/patchwork/patch/416962) | 在二进制中通过 xxx_sched_class 地址顺序标记调度类的优先级, 从而可以通过直接比较两个 xxx_sched_class 地址的方式, 优化调度器中两个热点函数 pick_next_task()和 check_preempt_curr(). | v2 ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/416962)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/patch/416962) |
| 2022/03/10 | Alex Sierra <alex.sierra@amd.com> | [split vm_normal_pages for LRU and non-LRU handling](https://patnux-mm/cover/20220310172633.9151-1-alex.sierra@amd.com/) | DEVICE_COHERENT 页面在 "普通" 页面可以被内核中的各种调用者使用的方式上引入了微妙的区别.<br> 为了在 CPU 页表中进行映射和 COW, 它们的行为就像普通页面一样. 但它们不支持 LRU 列表、NUMA 迁移或 THP. 因此, 我们将 vm_normal_page 拆分为两个函数 vm_normal_any_page 和 vm_normal_lru_page. 后者将只返回那些可以放在 LRU 列表中, 并且支持 NUMA 迁移、KSM 和 THP 的页面.<br> 在自测试中添加了 HMM 测试, 以使用设备连贯页面来执行这些更改. 名为 hmm_cow_in_device 的新测试将测试标记为 COW 的页面, 并在设备区域中分配. 此外, 还向 hmm_gup_test 中添加了更多的配置, 以测试基本的获取用户页面和设备区域页面中的获取用户页面的快速路径. | v1 ☐☑ | [LORE v1,0/3](https://lore.kernel.org/rsierra@amd.com) |
| 2022/05/27 | Jakub Matěna <matenajakub@gmail.com> | [Refactor of vma_merge and new merge call](https://patchwork.kernel.org/project/linux-mm/cover/20220527104810.24736-1-matenajakub@gmail.com/) | 645578 | v1 ☐☑ | [LORE v1,0/2](https://lore.kernel.org/r/20220527104810.24736-1-matenajakub@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/2](https://lore.kernel.org/r/20220527211708.839033-1-matenajakub@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/2](https://lore.kernel.org/r/20220602145642.16948-1-matenajakub@gmail.com) |


### 8.1.1 具名匿名页 Anonymous VMA naming
-------

用户空间进程通常有多个分配器, 每个分配器执行匿名 MMAP 以获取内存. 在整体检查单个进程或系统的内存使用情况时, 如果能够分解每个层分配的各种堆并检查它们的大小、RSS 和物理内存使用情况对分析内核的内存分配和管理非常有用.

因此内核开发者提出匿名 VMA 命名方案, 可以有效地对进程不同名称的 VMA 的进行统计, 比较和区分. 以前有很多匿名 VMA 命名的尝试.

最初 [Colin Cross](https://lore.kernel.org/linux-mm/1372901537-31033-1-git-send-email-ccross@android.com) 实现了一个重计数名称的字典, 以重用相同的名称字符串. 当时 Dave Hansen [建议改用用户空间指针](https://lore.kernel.org/linux-mm/51DDFA02.9040707@intel.com), 补丁也这样重写了.

后续 Sumit Semwal 发现这组补丁并没有被主线接受, 而一个非常相似的补丁已经存在 Android 中用于命名匿名 vma 很多年了, 参见 [ANDROID: mm: add a field to store names for private anonymous memory](https://github.com/aosp-mirror/kernel_common/commit/533e4ed309748abb89ae53548af7afe1649ac48a). 因此他 [接替了 Colin Cross 的工作, 并继续发到了 v7 版本](https://lore.kernel.org/linux-mm/20200901161459.11772-1-sumit.semwal@linaro.org), [Kees Cook](https://lore.kernel.org/linux-mm/5d0358ab-8c47-2f5f-8e43-23b89d6a8e95@intel.com/) 对这个补丁提出了担忧, 他注意到这个补丁缺少字符串清理功能, 并且使用了来自内核的用户空间指针. 社区建议 strndup_user() 来自 userspace 的字符串, 执行适当的检查并将副本存储为 vm_area_struct 成员. fork() 期间额外 strdup() 的性能影响应该通过分配大量(64k) 具有最长名称和定时 fork() 的 vma 来衡量.

接着, Suren Baghdasaryan 继续着这项工作. 实现了社区之间的 review 意见, 并优化了简单的重计数, 以避免在 fork() 期间 strdup() 名称所带来的性能开销. 最终于 v5.17 合入主线. 参见 LWN 报道 [Not-so-anonymous virtual memory areas](https://lwn.net/Articles/867818).

使用 CONFIG_ANON_VMA_NAME 开启这个功能. [用户空间](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9a10064f5625d5572c3626c1516e0bebc6c9fe9b)可以通过调用 [prctl(PR_SET_VMA, PR_SET_VMA_ANON_NAME, start, len, (unsigned long) name) 来设置内存区域的名称](https://elixir.bootlin.com/linux/v5.17/source/kernel/sys.c#L2294). 使用 struct anon_vma_name 记录具名匿名页的名字, 记录在 [struct vm_area_struct->anon_name](https://elixir.bootlin.com/linux/v5.17/source/include/linux/mm_types.h#L423) 中. [其次](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9a10064f5625d5572c3626c1516e0bebc6c9fe9b)在 `/proc/pid/maps` 和 `/proc/pid/smaps` 中添加一个[字段](https://elixir.bootlin.com/linux/v5.17/source/fs/proc/task_mmu.c#L333), 以显示用户空间为匿名 vmas 提供的名称. 已命名匿名 vmas 的名称显示为 `[anon:anon_name]`.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2013/07/03 | Suren Baghdasaryan <surenb@google.com> | [mm: add sys_madvise2 and MADV_NAME to name vmas](https://lore.kernel.org/linux-mm/1372901537-31033-1-git-send-email-ccross@android.com) | 这组通过添加新的 MADVESE2 系统调用, 可以使用 MADV_NAME 来将名称附加到现有的 VMA 上.<br>1. 向每个 vma 添加一个包含名称字符串的 vma_name 结构, 系统中每个具有相同名称的 vma 都保证指向相同的 vma_name 结构. 可以直接通过比较指针进行名称相等比较.<br>2. 匿名 VMA 的名称在 `/proc/pid/maps` 中显示为 `[anon:<name>]`. 所有命名 VMA 的名称显示在 `/proc/pid/smap` 中的 Name 字段中用于命名 VMA.<br>3. 此修补程序添加的未命名 vma 的唯一成本是检查 vm_name 指针. 对于命名 vma, 它会将 refcount 更新添加到拆分 / 合并 / 复制 vma 中, 如果命名 vma 是具有该名称的最后一个 vma, 则取消映射可能需要使用全局锁. | v2 ☐ | [PatchWork RFC](https://lore.kernel.org/linux-mm/1372901537-31033-1-git-send-email-ccross@android.com) |
| 2020/09/01 | Sumit Semwal <sumit.semwal@linaro.org> | [Anonymous VMA naming patches](https://lore.kernel.org/linux-mm/20200901161459.11772-1-sumit.semwal@linaro.org) | NA | v7 ☐ 5.9-rc3 | [PatchWork v7,0/3](https://lore.kernel.org/linux-mm/20200901161459.11772-1-sumit.semwal@linaro.org) |
| 2021/10/19 | Suren Baghdasaryan <surenb@google.com> | [Anonymous VMA naming patches](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=78db3412833dc9c479cd17412035f216cfd01a29) | NA | v11 ☑✓ 5.17-rc1 | [2021/08/27 PatchWork v8,0/3](https://patchwork.kernel.org/project/linux-mm/cover/20210827191858.2037087-1-surenb@google.com)<br>*-*-*-*-*-*-*-* <br>[2021/10/19 PatchWork v11,1/3](https://patchwork.kernel.org/project/linux-mm/patch/20211019215511.3771969-1-surenb@google.com) |
| 2022/11/05 | Pasha Tatashin <pasha.tatashin@soleen.com> | [mm: anonymous shared memory naming](https://lore.kernel.org/all/20221105025342.3130038-1-pasha.tatashin@soleen.com) | commit 9a10064f5625("mm: add a field to store names for private anonymous memory"), 可以设置私有匿名内存的名称, 但不能设置共享匿名. 但是, 命名共享匿名内存对于跟踪目的同样有用.<br> 扩展功能, 使其能够为共享匿名者设置名称. | v1 ☐☑✓ | [LORE](https://lore.kernel.org/all/20221105025342.3130038-1-pasha.tatashin@soleen.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2](https://lore.kernel.org/all/20221107184715.3950621-1-pasha.tatashin@soleen.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/1](https://lore.kernel.org/r/20221115020602.804224-1-pasha.tatashin@soleen.com) |


### 8.1.2 其他
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/05/16 | Jakub Matěna <matenajakub@gmail.com> | [Removing limitations of merging anonymous VMAs](https://patchwork.kernel.org/project/linux-mm/cover/20220218122019.130274-1-matenajakub@gmail.com/) | [Anonymous VMA merging improvements WIP](https://people.kernel.org/vbabka/anonymous-vma-merging-improvements-wip) | v1 ☐☑ | [2022/02/18 LORE v1,0/4](https://lore.kernel.org/r/20220218122019.130274-1-matenajakub@gmail.com)<br>*-*-*-*-*-*-*-* <br>[2022/05/16 LORE v3,0/6](https://lore.kernel.org/r/20220516125405.1675-1-matenajakub@gmail.com) |


## 8.2 Page Fault
-------

[linux 那些事之 gup_flags](https://zhikunhuo.blog.csdn.net/article/details/121712408) linux gup 子系统在处理各种 pin memory 中, 为了方便处理各种使用场景同时减少代码冗余, 使用了 gup_flags 标识位用于标记处理内存时各种场景.

[HZero 的内存管理专栏](https://blog.csdn.net/jasonactions/category_10652690.html) 其中有几篇详细讲解了 Page Fault 流程.

[Linux 内存管理 (10) 缺页中断处理](https://www.cnblogs.com/arnoldlu/p/8335475.html) 对各个处理函数有个简单的注解, 同时按照几个关键点 (比如页面是否在内存中, pte 内容是否存在, 有无 vm_ops) 区分 Page Fault 处理函数的流程. [Linux 内存管理：缺页异常 (一)](https://zhuanlan.zhihu.com/p/195580742) 则将这个框架流程绘制成了流程图的形式.

[ARM64 内存管理十五：do_page_fault 缺页中断](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/15/ARM64 内存管理十五 - do_page_fault 缺页中断)

[[原创](十四) Linux 内存管理之 page fault 处理](https://www.cnblogs.com/LoyenWang/p/12116570.html) 系统性地罗列了各个不同情形下 Page Fault 的处理框架.


| handler | 描述 |
|:-------:|:---:|
| do_anonymous_page | 用于 [处理匿名页的缺页](https://elixir.bootlin.com/linux/v5.15/source/mm/memory.c#L4566) 异常, 在以下情况下会触发:<br>1. malloc()/mmap() 分配了进程地址空间区域, 但是没有进行映射处理, 在首次访问时触发;<br>2. 用户栈不够的情况下, 进行栈区的扩大处理; |
| do_fault | 用于处理文件页异常, 包括以下三种情况:<br>1. [do_read_fault() 处理读文件页](https://elixir.bootlin.com/linux/v5.15/source/mm/memory.c#L4310)缺页 <br>2. [do_cow_fault() 处理写私有文件页](https://elixir.bootlin.com/linux/v5.15/source/mm/memory.c#L4312)缺页;<br>3. [do_share_faults() 写共享文件页](https://elixir.bootlin.com/linux/v5.15/source/mm/memory.c#L4314)缺页; |
| do_swap_page | 如果访问 swap 页面出错(页面不在内存中), 则从 swap cache 或 swap file 中读取该页面. |
| do_numa_page | Numa Balancing 处理时是通过 fault driven 的方式统计页面访问的 NUMA 和 TASK 信息的, 这时候就需要把进程 VMA 中的页面设置为 PTE_PROT_NONE, 从而触发 Page Fault; |
| do_wp_page | 用于[处理写时复制(COW/Copy On Write)](https://elixir.bootlin.com/linux/v5.15/source/mm/memory.c#L4586), 会在以下两种情况处理:<br>1. 创建子进程时, 父子进程会以只读方式共享私有的匿名页和文件页, 当试图写的时候, 触发页错误异常, 从而复制物理页, 并创建映射;<br>2. 进程创建私有文件映射, 读访问后触发异常, 将文件页读入到 page cache 中, 并以只读模式创建映射, 之后发生写访问后, 触发 COW; |



|  时间  | 作者 | 特性  | 描述  |  是否合入主线  | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2022/05/05 | Peter Xu <peterx@redhat.com> | [mm: Avoid unnecessary page fault retires on shared memory types](https://patchwork.kernel.org/project/linux-mm/patch/20220505211748.41127-1-peterx@redhat.com/) | 638888 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220505211748.41127-1-peterx@redhat.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20220506135205.46810-1-peterx@redhat.com) |


### 8.2.1 零页
-------

|  时间  | 作者 | 特性  | 描述  |  是否合入主线  | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2007/10/16 | Nick Piggin <npiggin@suse.de> | [remove ZERO_PAGE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=557ed1fa2620dc119adb86b34c614e152a629a80) | TODO | v1 ☑✓ 2.6.24-rc1 | [LORE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=557ed1fa2620dc119adb86b34c614e152a629a80) |
| 2008/06/20 | Linus Torvalds <torvalds@linux-foundation.org> | [Reinstate ZERO_PAGE optimization in'get_user_pages()' and fix XIP](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=89f5b7da2a6bad2e84670422ab8192382a5aeb9f) | TODO | v1 ☐☑✓ | [LORE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=89f5b7da2a6bad2e84670422ab8192382a5aeb9f) |
| 2008/06/23 | Linus Torvalds <torvalds@linux-foundation.org> | [Fix ZERO_PAGE breakage with vmware](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=672ca28e300c17bf8d792a2a7a8631193e580c74) | TODO | v1 ☐☑✓ | [LORE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=672ca28e300c17bf8d792a2a7a8631193e580c74) |
| 2009/09/21 | Hugh Dickins <hugh.dickins@tiscali.co.uk> | [mm: reinstate ZERO_PAGE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=03f6462a3ae78f36eb1f0ee8b4d5ae2f7859c1d5) | 主要清理 get_user_pages() 标志, 但修复 munlock 的 OOM, 整理"FOLL_ANON 优化 ", 并恢复零页面. <br>KAMEZAWA Hiroyuki 观察到早期内核的使能了 ZERO_PAGE, 但是引起了诸多错误, 因而在 2.6.24 时禁止了 do_anonymous_page() 中使用. 这组补丁恢复了 do_anonymous_page() 使用 ZERO_PAGE; 但是这一次不要用 (map) 计数更新来破坏它的结构页 cacheline. 让 vm_normal_page() 将其视为异常. | v1 ☑✓ 2.6.32-rc1 | [LORE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=03f6462a3ae78f36eb1f0ee8b4d5ae2f7859c1d5) |



### 8.2.2 COW(写时拷贝)
-------

在 fork 进程的时候, 并不会为子进程直接分配物理页面, 而是使用 COW 的机制. 在 fork 之后, 父子进程都共享原来父进程的页面, 同时将父子进程的 COW mapping 都设置为只读. 这样当这两个进程触发写操作的时候, 触发缺页中断, 在处理缺页的过程中再为该进程分配出一个新的页面出来.

```cpp
// https://www.cnblogs.com/pengdonglin137/p/16131785.html
sys_fork
    -> kernel_clone
        -> copy_process
            -> copy_mm
                -> dup_mm
                    -> dup_mmap
                        -> copy_page_range
                            -> copy_p4d_range
                                -> 如果时 PUD 巨型页：copy_huge_pud: 分别将父子的 PUD 页表项设置为写保护
                                    -> pudp_set_wrprotect(src_mm, addr, src_pud);
                                    -> set_pud_at(dst_mm, addr, dst_pud, pud_mkold(pud_wrprotect(pud)));
                                -> copy_pmd_range
                                    -> 如果是 PMD 巨型页：copy_huge_pmd: 分别将父子的 PMD 页表项设置为写保护
                                        -> pmdp_set_wrprotect(src_mm, addr, src_pmd);
                                        -> set_pmd_at(dst_mm, addr, dst_pmd, pmd_mkold(pmd_wrprotect(pmd)));
                                    -> copy_pte_range
                                        -> copy_present_pte
                                            -> 如果：is_cow_mapping(vm_flags) && pte_write(pte)
                                                -> ptep_set_wrprotect(src_mm, addr, src_pte);
                                                -> set_pte_at(dst_vma->vm_mm, addr, dst_pte, pte_wrprotect(pte));
```

执行 fork 完毕后, 原先父进程中可写的区域在父子进程中都被设置了 CoW, 即 pte 中可写的属性被清除. 那么下次对此地址进行写操作时(不管是父进程还是子进程), 都会触发 PageFault. 然后新分配一个页面出来公其使用, 同时原来 pte 的可写属性也会被重新设置.



#### 8.2.2.1 匿名页的写时拷贝
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/09/27 | David Hildenbrand <david@redhat.com> | [selftests/vm: test COW handling of anonymous memory](https://patchwork.kernel.org/project/linux-mm/cover/20220927110120.106906-1-david@redhat.com/) | 680969 | v1 ☐☑ | [LORE v1,0/7](https://lore.kernel.org/r/20220927110120.106906-1-david@redhat.com) |
| 2023/01/04 | David Hildenbrand <david@redhat.com> | [[mm-unstable,v1] selftests/vm: cow: Add COW tests for collapsing of PTE-mapped anon THP](https://patchwork.kernel.org/project/linux-mm/patch/20230104144905.460075-1-david@redhat.com/) | 708828 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20230104144905.460075-1-david@redhat.com) |

#### 8.2.2.2 文件页的写时拷贝
-------

#### 8.2.2.3 页表的写时拷贝
-------

[Introduce Copy-On-Write to Page Table](https://patchwork.kernel.org/project/linux-mm/cover/20220927162957.270460-1-shiyn.lin@gmail.com) 这组补丁集为 PTE 级页表引入了写入时拷贝(COW). 参见 [Memory-management short topics: page-table sharing and working sets](https://lwn.net/Articles/919143)

在用户需要程序副本才能在隔离环境中运行的情况下, COW PTE 提高了性能. 基于反馈的模糊器 (例如, AFL) 和微服务框架是两个主要的例子. 例如, COW PTE 在 fuzzer(AFL) 上运行 SQLite 时, 吞吐量增加了 9.3 倍.

由于 COW PTE 只在特定场景下能获得性能提升, 因此作者添加了一个新的 sysctl vm.cow_pte, 带有输入进程 ID(PID), 允许用户为特定进程启用 cow pte.

1. 为了处理具有共享 PTE 表的每个进程的页表状态, 该补丁引入了 COW PTE 表所有权的概念. 这个实现使用 PMD 索引的地址来跟踪 PTE 表的所有权. 这有助于维护 COW PTE 表的状态, 例如 RSS 和 pgtable_bytes. 一些 PTE 表 (例如, 驻留在表中的固定页面) 仍然需要立即复制, 以与当前的 COW 逻辑保持一致. 结果, 一个标志 COW_PTE_OWNER_EXCLUSIVE 被添加到表的所有者指针上, 它指示一个 PTE 表是否是独占的(即, 一次只有一个任务拥有它). 每次在 fork 期间复制 PTE 表时, 将检查所有者指针(以及独占标志), 以确定 PTE 表是否可以跨进程共享.

2. 使用一个引用计数来跟踪共享页表的生命周期. 使用 COW PTE 调用 fork 将增加引用计数. refcount=1 表示页表目前没有与其他进程共享, 但可能会被共享. 并且, 当有人写入共享 PTE 表时, 会导致写入故障中断 COW PTE, 如果共享 PTE 表的 refcount 为 1, 触发该故障的进程将重用共享 PTE 表. 否则, 进程将减少引用计数, 将信息复制到一个新的 PTE 表, 或取消引用所有信息, 并更改所有者(如果他们拥有共享 PTE 表).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/09/27 | Chih-En Lin <shiyn.lin@gmail.com> | [Introduce Copy-On-Write to Page Table](https://patchwork.kernel.org/project/linux-mm/cover/20220927162957.270460-1-shiyn.lin@gmail.com/) | 目前, 写入时复制仅用于映射内存; 在分叉期间, 子进程仍然需要从父进程复制整个页表. 当父进程分配了一个大页表时, 父进程可能需要花费大量时间和内存来复制页表. 这组补丁集为 PTE 级页表引入了写入时拷贝(COW). | v2 ☐☑ | [LORE v2,0/9](https://lore.kernel.org/r/20220927162957.270460-1-shiyn.lin@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/14](https://lore.kernel.org/r/20221220072743.3039060-1-shiyn.lin@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/14](https://lore.kernel.org/r/20230207035139.272707-1-shiyn.lin@gmail.com) |


#### 8.2.2.X 写时拷贝的问题
-------


Dirty COW(CVE-2016-5195) 是近几年影响比较严重的问题, 参见 [Dirty COW and clean commit messages](https://lwn.net/Articles/704231). 也让内核开发者开始关注 COW 机制可能引起的安全问题.

随后在 2019 年 Andrew Baumann, Jonathan Appavoo, Orran Krieger 和 Timothy Roscoe 在 Microsoft Research 网站上发表的一篇研究论文 [A fork() in the road](https://www.microsoft.com/en-us/research/uploads/prod/2019/04/fork-hotos19.pdf), 认为 fork() 系统调用是一个基本的设计错误. 他们认为 fork 作为一流的 OS 原语的持续存在阻碍了系统研究, 应该弃用它. 我们应该把 fork 当作历史文物来研究, 而不是作为进程的第一道工序创造机制. [LWN 上当时也对此进行了讨论](https://lwn.net/Articles/785430). 以及知乎上一些 [(赵俊民) 大佬的分析](https://zhuanlan.zhihu.com/p/272675052). 当时掀起了轩然大波, 讨论了很多方面, 包括性能, 安全等. 甚至有人质疑这篇论文是微软对 linux 发起的攻击.

2021 年 Redhat 的开发者 David Hildenbrand 总结了上游社区中存在的 COW 问题. 并发布在邮件列表中. [Summary of COW (Copy On Write) Related Issues in Upstream Linux](https://lore.kernel.org/all/3ae33b08-d9ef-f846-56fb-645e3b9b4c66@redhat.com). 随后在某个阳光明媚的周三进行的 linux-mm alignment session 中对这些问题进行了讨论:

| [CVE-2020-29374](https://nvd.nist.gov/vuln/detail/CVE-2020-29374)  | Observing Memory Modifications of Private Pages From A Child Process |
|:---------------:|:--------------------------------------------------------------------:|
| 参考资料 | 摘要: [Patching until the COWs come home (part 1)](https://lwn.net/Articles/849638).<br> 详情: [Patching until the COWs come home (part 2)](https://lwn.net/Articles/849876). |
| 影响 | 一旦 fork(), 进程私有内存可能不像你想的那样私有, 子进程仍然可以观察到父进程中私有内存区域的连续修改, 例如, 通过使用 vmsplice() + munmap(). |
| 核心问题 | 将可读页面固定在子进程中(比如通过 vmsplice 系统调用), 可能会导致子进程观察父进程所做的内存修改, 而子进程不应该观察父进程. |

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/01/26 | David Hildenbrand <david@redhat.com> | [mm: COW fixes part 1: fix the COW security issue for THP and hugetlb](https://patchwork.kernel.org/project/linux-mm/cover/20211217113049.23850-1-david@redhat.com) | NA | v1 ☐ | [PatchWork v1,00/11](https://patchwork.kernel.org/project/linux-mm/cover/20211217113049.23850-1-david@redhat.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2,0/9](https://lore.kernel.org/r/20220126095557.32392-1-david@redhat.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v3,0/9](https://lore.kernel.org/r/20220131162940.210846-1-david@redhat.com) |
| 2022/03/15 | David Hildenbrand <david@redhat.com> | [mm: COW fixes part 2: reliable GUP pins of anonymous pages](https://patchwork.kernel.org/project/linux-mm/cover/20220224122614.94921-1-david@redhat.com/) | 617540 | v1 ☐☑ | [2022/02/24 LORE v1,00/13](https://lore.kernel.org/r/20220224122614.94921-1-david@redhat.com))<br>*-*-*-*-*-*-*-* <br>[2022/03/08 LORE v1,00/15](https://lore.kernel.org/r/20220308141437.144919-1-david@redhat.com)<br>*-*-*-*-*-*-*-* <br>[2022/03/15 LORE v2,00/15](https://lore.kernel.org/r/20220315104741.63071-1-david@redhat.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,00/16](https://lore.kernel.org/r/20220329160440.193848-1-david@redhat.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,00/17](https://lore.kernel.org/r/20220428083441.37290-1-david@redhat.com) |
| 2022/03/15 | David Hildenbrand <david@redhat.com> | [mm: COW fixes part 3: reliable GUP R/W FOLL_GET of anonymous pages](https://patchwork.kernel.org/project/linux-mm/cover/20220315141837.137118-1-david@redhat.com/) | 623540 | v1 ☐☑ | [LORE v1,0/7](https://lore.kernel.org/r/20220315141837.137118-1-david@redhat.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/8](https://lore.kernel.org/r/20220329164329.208407-1-david@redhat.com) |


| [CVE-2020-29374](https://nvd.nist.gov/vuln/detail/CVE-2020-29374)  | Intra Process Memory Corruptions due to Wrong COW (FOLL_GET) |
|:---------------:|:--------------------------------------------------------------------:|
| 参考资料 | NA |
| 影响 | NA |
| 根本问题 | NA |



### 8.2.3 enhance
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/09/10 | Catalin Marinas <catalin.marinas@arm.com> | [arm64: Add support for hardware updates of the access and dirty pte bits](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1436545468-1549-1-git-send-email-catalin.marinas@arm.com) | ARMv8.1 体系结构扩展引入了对页表项中访问和脏信息的硬件更新的支持, 用于支持硬件自动原子地完成页表更新("读 - 修改 - 回写").<br>TCR_EL1.HA 为 1, 则使能硬件的自动更新访问位. 当处理器访问内存地址时, 硬件自动设置 PTE_AF 位, 而不是再触发访问位标志错误.<br>TCR_EL1.HD 为 1, 则使能硬件的脏位管理.  | v1 ☑ [4.3-rc1](https://kernelnewbies.org/Linux_4.3#Architectures) | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/344775)<br>*-*-*-*-*-*-*-* <br>[PatchWork](https://lore.kernel.org/patchwork/patch/344816), [commit 2f4b829c625e](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2f4b829c625ec36c2d80bef6395c7b74cea8aac0) |
| 2017/07/25 | Catalin Marinas <catalin.marinas@arm.com> | [arm64: Fix potential race with hardware DBM in ptep_set_access_flags()](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20170725135308.18173-2-catalin.marinas@arm.com) | 修复前面支持 TCP_EL1.HA/HD 引入的硬件和软件的竞争问题 | RFC ☐ | [PatchWork RFC](https://patchwork.kernel.org/project/linux-mm/cover/20200224203057.162467-1-walken@google.com) |
| 2022/03/15 | Bibo Mao <maobibo@loongson.cn> | [[2/2] mm: add access/dirty bit on numa page fault](https://patchwork.kernel.org/project/linux-mm/patch/20220315092323.620610-1-maobibo@loongson.cn/) | 在 x86/arm 等支持 hw 页面行走的平台上, access 和 dirty bit 由 hw 设置, 而在一些没有这些 hw 功能的平台上, access 和 dirty bit 由 next trap 中的软件设置.<br> 在 numa 页面故障期间, 如果写入故障时迁移失败, 可以为旧 pte 添加脏位. 如果迁移成功, 可以为迁移后的新 pte 增加访问位, 也可以为写错误增加脏位. | v1 ☐☑ | [LORE v1,0/2](https://lore.kernel.org/r/20220315092323.620610-1-maobibo@loongson.cn)<br>*-*-*-*-*-*-*-* <br>[LORE v2](https://lore.kernel.org/r/20220317065033.2635123-1-maobibo@loongson.cn) |

### 8.2.4 userfaultfd
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/06/19 | Nadav Amit <nadav.amit@gmail.com> | [userfaultfd: support access/write hints](https://patchwork.kernel.org/project/linux-mm/cover/20220619233449.181323-1-namit@vmware.com/) | 651866 | v2 ☐☑ | [LORE v2,0/5](https://lore.kernel.org/r/20220619233449.181323-1-namit@vmware.com)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/5](https://lore.kernel.org/r/20220622185038.71740-1-namit@vmware.com) |
| 2023/02/13 | Muhammad Usama Anjum <usama.anjum@collabora.com> | [[v2,1/2] mm/userfaultfd: Support WP on multiple VMAs](https://patchwork.kernel.org/project/linux-mm/patch/20230213163124.2850816-1-usama.anjum@collabora.com/) | 721349 | v2 ☐☑ | [LORE v2,0/2](https://lore.kernel.org/r/20230213163124.2850816-1-usama.anjum@collabora.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/2](https://lore.kernel.org/r/20230216064155.1500545-1-usama.anjum@collabora.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/1](https://lore.kernel.org/r/20230217105558.832710-1-usama.anjum@collabora.com) |
| 2023/02/14 | Axel Rasmussen <axelrasmussen@google.com> | [mm: userfaultfd: add UFFDIO_CONTINUE_MODE_WP to install WP PTEs](https://patchwork.kernel.org/project/linux-mm/patch/20230214215046.1187635-1-axelrasmussen@google.com/) | 721865 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20230214215046.1187635-1-axelrasmussen@google.com) |
| 2023/02/15 | Peter Xu <peterx@redhat.com> | [mm/uffd: UFFD_FEATURE_WP_ZEROPAGE](https://patchwork.kernel.org/project/linux-mm/patch/20230215210257.224243-1-peterx@redhat.com/) | 722253 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20230215210257.224243-1-peterx@redhat.com) |


### 8.2.5 MMAP Locking Scalability
-------

当用户空间通过 malloc()/mmap() 等使用内存时, 只是分配了虚拟内存, 即只在进程地址空间中建立了 VMA, 并没有分配物理内存, 因此自然也没有建立虚拟内存与物理内存的关联. 当进程首次访问时会触发缺页异常处理.

当进程发生缺页异常时, 由于需要对进程空间 VMA 进行操作, 开始缺页前会先持进程的 mmap_sem 锁, 缺页完成后释放 mmap_sem.

进程缺页处理时需要持有 mmap_sem 的 reader 锁. mmap_sem 锁是进程为了保护自身虚拟地址空间不受多线程并发访问影响而设计的. 而频繁发生缺页时, mmap_sem 锁的竞争将会非常激烈.

由于 map_sem 锁的存在, 多线程程序访问内存方面的并发能力严重受到该锁的竞争激烈程度的制约. 要改善这个问题, 毫无疑问是需要减少 mmap_sem 锁的竞争.





| 时间 | 讨论 |
|:----:|:----:|
| 2013 年 | [LSFMM: Problems with mmap_sem](https://lwn.net/Articles/548098) |
| 2014 年 | [LWN: Memory management locking](https://lwn.net/Articles/591978) |
| 2015 年 | [Topics of interest from the MM summit](https://events.static.linuxfound.org/sites/events/files/slides/mm.pdf) |
| 2017 年 | [Another attempt at speculative page-fault handling](https://lwn.net/Articles/730531) |
| 2018 年 | [Zone-lock and mmap_sem scalability](https://lwn.net/Articles/753269), [The LRU lock and mmap_sem](https://lwn.net/Articles/753058) |
| 2019 年 | [Splitting the mmap_sem](https://lore.kernel.org/all/20191203222147.GV20752@bombadil.infradead.org) |
| 2019 年 | [How to get rid of mmap_sem](https://lwn.net/Articles/787629) |
| 2021 年 | [Introducing maple trees](https://lwn.net/Articles/845507) |
| 2021 年 | [LSF/MM TOPIC] mmap locking topics](https://www.spinics.net/lists/linux-mm/msg258803.html) |
| 2022 年 | [The ongoing search for mmap_lock scalability](https://lwn.net/Articles/893906)<br>LPC-2022 [Scalability solutions for the mmap_lock - Maple Tree and per-VMA locks](https://lpc.events/event/16/contributions/1271) |

#### 8.2.5.1 SPF(Speculative page faults)
-------

[投机性缺页异常 (SPF) 原理分析](https://zhuanlan.zhihu.com/p/567855662)

[投机性缺页异常处理](https://developer.aliyun.com/article/767293)

[spf_test](https://github.com/surenbaghdasaryan/spf_test/blob/main/spf_test.c)

为了解决这个问题, 内核研发人员 Peter Zijlstra 提出了 投机性缺页 SPF(Speculative page-fault handling) 优化来实现对进程 VMA 的无锁读访问, 并在 v4.17(2009 年)开发窗口期间发出了第一版的补丁集. 它的基本思路是通过避免在缺页异常处理中使用 mmapsem, 无锁遍历 VMA, 从而提高内存访问的性能.

投机性缺页就像一场赌博, 它在赌访问 VMA 时 VMA 没有被修改, 如果 VMA 没有被修改, 它就赌赢了, 这个过程确实不需要持锁; 如果 VMA 被修改了, 它就赌输了, SPF 做的工作就没有任何意义, 还是需要继续传统的缺页处理. 当然, 如果系统整体情况大部分时候 SPF 赌赢了, 它其实就赚了.

但是我们不得不说, 设计和使用 mmap_sem 就是为了解决一些不好解决的同步问题, 如果想要无锁化, 那这些问题就得想其他的办法去解决.

| 问题 1 | SPF 优化策略 | SPF 实现 |
|:------:|:----------:|:--------:|
| 无锁处理 PageFault 的区间越大, SPF 赌输的概率越大 | 缩小临界区的大小 | 尽可能把与 VMA 状态无关的工作都先做掉, 然后在直接改变进程地址空间之前再检查一下 VMA 是否发生了改变. 举例来说, 当我们从磁盘读数据到内存的时候, 我们可以先分配一个内存页, 将数据读取出来, 这些阶段都是不需要 mmap_sem 的, 而当我们把这个页加入到进程地址空间的时候我们需要一个一致的 VMA, 所以这个时候是需要拿 mmap_sem 的 |
| 写冲突. 无锁处理 PageFault 期间, 对应的 VMA 描述中的区域可能发生改变. | 通过 seqlock 保护 VMA 修改 | 在 VMA 中增加顺序锁 seqlock, 所有修改 VMA 的地方都会增加 sequence count 计数, 在缺页时工作前先获取一次 sequence count, 然后不持 mmap_sem 锁情况下进行缺页处理, 处理完成后, 需要再获取一次 sequence count, 检查有计数有没有发生改变, 如果发生了改变, 说明 VMA 在这个期间被修改过了, 如果没有发生改变, 说明 VMA 在这个期间没有被修改过. |
| 释放冲突. 无锁处理 PageFault 期间, 对应的 VMA 可能被释放. | 最早 Peter 的版本建议通过 [SRCU(RCU 的一种可睡眠的变体)来串行化 VMA](https://lore.kernel.org/lkml/20100104182813.479668508@chello.nl/) 的更新. 这可以保证在处理缺页异常的时候, VMA 结构是存在的. Laurent 接手后 v10 期间发现它引起了性能的下降. v11 基于 SRCU 保护 VMA 释放的灵感通过 [mm_rb_lock 来实现](https://lore.kernel.org/linux-mm/1526555193-7242-19-git-send-email-ldufour@linux.vnet.ibm.com). mm_rb_lock 锁机引入了 vma_get() 和 vma_put() 使得得在访问 RB Tree 时使用不持有 mmap_sem. v12 又改成 [sequence lock](https://lore.kernel.org/lkml/20190416134522.17540-20-ldufour@linux.ibm.com). 不过后来 Michel Lespinasse 接手的版本使用了 [rcu safe vma freeing](https://lore.kernel.org/all/20220128131006.67712-12-michel@lespinasse.org). |
| TLB 失效. 很多行为, 例如 unmapping 一个内存区域, 都会导致 TLB 的失效. 失效 TLB 的过程是发送处理期间中断 (IPI) 来告诉每个 CPU 失效它自己的 TLB. unmap 的调用路径可能会在锁住特定的页表项期间进行 TLB 失效操作. 此时, Speculative-fault-handling 可能在关中断的情况下尝试获取页表锁, 如果这种尝试的页表项被锁在 unmap 的路径上, 处理器会在关中断的情况下自旋, 因此永远收不到 TLB 失效的 IPI, 这将导致死锁. | 优先尝试 SPF 无锁访问 | 在 speculative 路径使用 trylock 操作获取锁, 如果获取锁失败, 则立即 fall back 到传统 page fault 处理流程上. |

| 时间线 | 对应版本 | 描述 |
|:-----:|:-------:|:---:|
| 2009 | NA | speculative page fault 相关的 patch 由 Hiroyuki Kamezawa 在 2009 年发布 |
| 2010 | NA | 接下来 Peter Zijlstra 组织了内核社区的讨论并开发了他自己的实现, 他使用了 RCU 来完成无锁读 VMA. 不过 Peter 的实现也有一些问题, 所以没能被合并进主干. |
| 2014 | NA | 由于很多之前导致他的 patch 不能工作的问题都已经解决了, 所以 Peter Zijlstra 重启了这个想法. 然而, 讨论再次无疾而终. |
| 2019 | NA | Dufourt 在最新内核移植了 PeterZ 的 patch, 同时也添加了自己的一些 patch, 然后重新发送到了邮件列表. 不过 Dufour 提到, 他的 patch 集里仍然存在 TLB 失效的问题. |
| 2022 | v5.12~v5.17 | Michel 继续了这项工作 |


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2009/12/18 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [speculative page fault](https://lore.kernel.org/all/20091218093849.8ba69ad9.kamezawa.hiroyu@jp.fujitsu.com) | TODO | v1 ☐☑✓ | [LORE v1,0/11](https://lore.kernel.org/all/20091218093849.8ba69ad9.kamezawa.hiroyu@jp.fujitsu.com) |
| 2009/12/24 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [asynchronous page fault](https://lkml.org/lkml/2009/12/24/153) | TODO | v1 ☐☑✓ | [LORE v1,0/11](https://lkml.org/lkml/2009/12/24/153) |
| 2010/01/04 | Peter Zijlstra <a.p.zijlstra@chello.nl> | [Speculative pagefault -v3](https://lore.kernel.org/lkml/20100104182429.833180340@chello.nl) | SPF | RFC v3 ☐  | [PatchWork](https://lore.kernel.org/lkml/20100104182429.833180340@chello.nl) |
| 2019/04/16 | Laurent Dufour <ldufour@linux.vnet.ibm.com> | [Speculative page faults](http://lore.kernel.org/patchwork/patch/1062659) | SPF | v12 ☐  | [LORE v11,00/26](https://lore.kernel.org/linux-mm/1526555193-7242-1-git-send-email-ldufour@linux.vnet.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v12,00/31](https://lore.kernel.org/lkml/20190416134522.17540-1-ldufour@linux.ibm.com) |
| 2022/01/28 | Michel Lespinasse <michel@lespinasse.org> | [Speculative page faults (anon vmas only)](http://lore.kernel.org/patchwork/patch/1420569) | SPF | v11 ☐  | [LORE 00/29](https://lore.kernel.org/lkml/20210430195232.30491-1-michel@lespinasse.org)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC,00/37](https://lore.kernel.org/patchwork/patch/1408784)<br>*-*-*-*-*-*-*-* <br>[PatchWork v1](https://lore.kernel.org/patchwork/patch/1420569)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2,00/35](https://patchwork.kernel.org/project/linux-mm/cover/20220128131006.67712-1-michel@lespinasse.org) |


#### 8.2.5.2 mmap_sem Scalability
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/03/21 | Yang Shi <yang.shi@linux.alibaba.com> | [Drop mmap_sem during unmapping large map](https://lore.kernel.org/all/1521581486-99134-1-git-send-email-yang.shi@linux.alibaba.com) | 1521581486-99134-1-git-send-email-yang.shi@linux.alibaba.com | v1 ☐☑✓ | [LORE v1,0/8](https://lore.kernel.org/all/1521581486-99134-1-git-send-email-yang.shi@linux.alibaba.com) |



#### 8.2.5.3 Fine grained MM locking
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2013/01/31 | Michel Lespinasse <walken@google.com> | [Mapping range lock](https://lore.kernel.org/patchwork/patch/356467) | 文件映射的 mapping lock | RFC ☐  | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/356467) |
| 2020/02/24 | Michel Lespinasse <walken@google.com> | [Fine grained MM locking](https://patchwork.kernel.org/project/linux-mm/cover/20200224203057.162467-1-walken@google.com) | 细粒度 MM MMAP lock | RFC ☐  | [PatchWork RFC](https://patchwork.kernel.org/project/linux-mm/cover/20200224203057.162467-1-walken@google.com), [fine_grained_mm.pdf](https://linuxplumbersconf.org/event/4/contributions/556/attachments/304/509/fine_grained_mm.pdf) |

#### 8.2.5.4 Per-VMA locks
-------

参见 LWN 报道 [Concurrent page-fault handling with per-VMA locks](https://lwn.net/Articles/906852)

2022 年, LSF/MM 在 [SPF](https://lore.kernel.org/all/20220128131006.67712-1-michel@lespinasse.org) 讨论中讨论了 Per-VMA locks 的想法, 该想法的结论是: 可以 rw_semaphore 放入 VMA 本身; 这将产生使用 VMA 作为一种锁范围的效果.

> 其基本实现思路是:
>
> 在处理页面错误时, 我们查找包含 RCU 保护下的错误页面的 VMA, 并尝试获取其锁. 如果失败了, 我们将返回使用 mmap_lock, 类似于 SPF 处理这种情况的方式.

这里面同样存在不少问题, 比如 VMA 的读锁定方式(the way VMAs are read-locked). 在某些 MM 更新期间, 需要锁定多个 VMA, 直到更新结束(例如 vma_imerge、split_vma 等). 这需要

1. 跟踪所有锁定的 VMA, 避免递归锁, 确定何时可以安全地解锁先前锁定的 VMAs, 这将使代码更加复杂. 因此, 与通常的锁定/解锁模式不同, 所提出的解决方案将 VMA 标记为锁定, 并提供了一种有效的方式: ①. 识别锁定的 VMA. ② 批量解锁所有锁定的 VMA.

2. 将解锁锁定的 VMA 推迟到更新结束时, 即执行 mmap_write_unlock. 这可能会使 VMA 锁定的时间超过绝对必要的时间, 但会大大降低代码复杂性.

VMA 的读锁定是使用两个序列号完成的: 一个在 vm_area_struct 中, 一个在 mm_struct. 当这些序列号相等时, VMA 被视为读锁定.

1. 为了读取 VMA 的锁, 将 vm_area_struct 中的序列号设置为等于 mm_struct 的序列号.

2. 为了解锁所有 VMA, 增加 mm_struct 的序列号. 这允许一种有效的方法来跟踪锁定的 VMA, 并在更新结束时删除所有 VMA 上的锁定.

当前实现仅对未交换的匿名页面实现每个 VMA 锁定, 并避免用户错误, 因为它们的实现更复杂. 可以增量添加对文件背面错误、交换页面和用户页面的额外支持. 性能基准显示出与 SPF 贴片相似但略小的好处(约为 SPF 收益的 75%). 尽管如此, 由于复杂性较低, 这种方法可能更可取.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/08/29 | Suren Baghdasaryan <surenb@google.com> | [per-VMA locks proposal](https://lore.kernel.org/all/20220829212531.3184856-1-surenb@google.com) | TODO | v1 ☐☑✓ | [2022/08/29 LORE v1,0/28](https://lore.kernel.org/all/20220829212531.3184856-1-surenb@google.com)<br>*-*-*-*-*-*-*-* <br>[2022/09/01 LORE v1,0/28](https://lore.kernel.org/all/20220901173516.702122-1-surenb@google.com) |
| 2023/01/09 | Suren Baghdasaryan <surenb@google.com> | [Per-VMA locks](https://patchwork.kernel.org/project/linux-mm/cover/20230109205336.3665937-1-surenb@google.com/) | 710245 | v1 ☐☑ | [LORE v1,0/41](https://lore.kernel.org/r/20230109205336.3665937-1-surenb@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/33](https://lore.kernel.org/r/20230127194110.533103-1-surenb@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,00/35](https://lore.kernel.org/all/20230216051750.3125598-1-surenb@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/33](https://lore.kernel.org/all/20230227173632.3292573-1-surenb@google.com) |


#### 8.2.5.5 Maple Tree
-------

[Maple Tree"RFC"Patches Sent Out As New Data Structure To Help With Linux Performance](https://www.phoronix.com/scan.php?page=news_item&px=Maple-Tree-Linux-RFC)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/08/22 | Liam Howlett <liam.howlett@oracle.com> | [Introducing the Maple Tree](https://lore.kernel.org/patchwork/patch/1477973) | Maple Tree 是一种基于 RCU 安全范围的 B 树, 旨在高效使用现代处理器缓存. 在内核中有许多地方, 基于范围的非重叠树是有益的, 尤其是具有简单接口的树. Maple Tree 的第一个用户是 vm_area_struct, 当前替换了三个结构: 增强 rbtree、vma 缓存和 mm_struct 中的 vma linked 链表. 长期目标是减少或消除 mmap_sem 争用. | v9 ☐ | [2021/08/17 PatchWork v2,00/61](https://patchwork.kernel.org/project/linux-mm/cover/20210817154651.1570984-1-Liam.Howlett@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2021/10/05 PatchWork v3](https://patchwork.kernel.org/project/linux-mm/cover/20211005012959.1110504-1-Liam.Howlett@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2021/12/01 PatchWork v4,00/66](https://patchwork.kernel.org/project/linux-mm/cover/20211201142918.921493-1-Liam.Howlett@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2022/02/02 PatchWork v5,00/70](https://patchwork.kernel.org/project/linux-mm/cover/20220202024137.2516438-1-Liam.Howlett@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2022/04/04 LORE v7,00/70](https://lore.kernel.org/r/20220404143501.2016403-1-Liam.Howlett@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2022/04/26 LORE v8,0/70](https://lore.kernel.org/r/20220426150616.3937571-1-Liam.Howlett@oracle.com)<br>*-*-*-*-*-*-*-* <br>[LORE v9,0/69](https://lore.kernel.org/r/20220504010716.661115-1-Liam.Howlett@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2022/06/21 LORE v10,0/69](https://lore.kernel.org/all/20220621204632.3370049-1-Liam.Howlett@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2022/07/17 LORE v11,0/69](https://lore.kernel.org/r/20220717024615.2106835-1-Liam.Howlett@oracle.com)<br>*-*-*-*-*-*-*-* <br>[2022/08/22 ORE v13,0/70](https://lore.kernel.org/r/20220822150128.1562046-1-Liam.Howlett@oracle.com) |
| 2022/05/04 | Liam Howlett <Liam.Howlett@Oracle.com> | [Prepare for maple tree](https://patchwork.kernel.org/project/linux-mm/cover/20220504002554.654642-1-Liam.Howlett@oracle.com/) | 638130 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220504002554.654642-1-Liam.Howlett@oracle.com) |
| 2022/10/11 | Liam Howlett <Liam.Howlett@Oracle.com> | [mm/mmap: Preallocate maple nodes for brk vma expansion](https://patchwork.kernel.org/project/linux-mm/patch/20221011160624.1253454-1-Liam.Howlett@oracle.com/) | 684552 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221011160624.1253454-1-Liam.Howlett@oracle.com) |
| 2022/10/28 | Liam Howlett <Liam.Howlett@Oracle.com> | [[v2] maple_tree: Reorganize testing to restore module testing](https://patchwork.kernel.org/project/linux-mm/patch/20221028180415.3074673-1-Liam.Howlett@oracle.com/) | 689987 | v2 ☐☑ | [LORE v2,0/1](https://lore.kernel.org/r/20221028180415.3074673-1-Liam.Howlett@oracle.com) |


## 8.3 反向映射 RMAP(Reverse Mapping)
-------

[郭健:  Linux 内存逆向映射(reverse mapping) 技术的前世今生](https://blog.51cto.com/u_15015138/2557286)

[Reverse-Mapping VM Performance](https://www.usenix.org/conference/2003-linux-kernel-developers-summit/presentation/reverse-mapping-vm-performance)

[Virtual Memory Management Techniques in 2.6 Linux kernel and challenges](https://www.researchgate.net/publication/348620478_Virtual_Memory_Management_Techniques_in_26_Linux_kernel_and_challenges)

[OLS2003](https://www.kernel.org/doc/ols/2003)

[linux 内存管理 - 反向映射](https://blog.csdn.net/faxiang1230/article/details/106609834)

[如何知道物理内存中的某个页帧属于某个进程, 或者说进程的某个页表项?](https://www.zhihu.com/question/446137543)

RMAP 反向映射是一种物理地址反向映射虚拟地址的方法.

正向映射是通过虚拟地址根据页表找到物理内存, 而反向映射就是通过物理地址或者物理页面找到哪些进程或者哪些虚拟地址使用它.

那内核在哪些路径下需要进行反向映射呢?

反向映射的典型应用场景:

1.  kswapd 进行页面回收时, 需要断开所有映射了该匿名页面的 PTE 表项;

2.  页面迁移时, 需要断开所有映射了该匿名页面的 PTE 表项;

即一个物理页面是可以被多个进程映射到自己的虚拟地址空间. 在整个内存管理的过程中, 有些页面可能被迁移, 有些则可能要被释放、写回到磁盘或者 交换到 SWAP 空间(磁盘) 上, 那么这时候经常要找到哪些进程映射并使用了这个页面.

*   2.4 的 reverse mapping


在 Linux 2.4 的时代, 为了确定映射了某个页面的所以进程, 我们不得不遍历每个进程的页表, 这是一项费时费力的工作.

*   PTE-based reverse mapping

于是在 2002 年 [Linux 2.5.27](https://kernelnewbies.org/LinuxVersions) 内核开发者实现了一种低开销的[基于 PTE 的的反向映射方方案(low overhead pte-based reverse mapping scheme)](http://lastweek.io/notes/rmap) 来解决这一问题. 经过了不断的发展, 到 2.5.33 趋于完善.

这个版本的实现相当直接, 在每个物理页(包括匿名页和文件映射页)(的通过结构体类型 struct page) 中维护了一个名为 [pte_chain 的链表](https://elixir.bootlin.com/linux/v2.5.27/source/include/linux/mm.h#L161), 里面包含所有引用该页的页表项, 链表的每一项都直接指向映射该物理页面的页表项 PTE 的指针.

该机制在大多数情况下都很高效, 但其自身也存在一些问题. 链表中保存反向映射信息的节点很多, 占用了大量内存, 并且 pte_chain 链表本身的维护的工作量也十分繁重, 并且处理速度随着物理页面的增多而变得缓慢. 以 fork() 系统调用为例, 由于需要为进程地址空间中的每个页面都添加新的反向映射项, 所以处理较慢, 影响了进程创建的速度. 关于这个方案的详细实现可以参考 [PATCH 2.5.26-rmap-1-core](http://loke.as.arizona.edu/~ckulesa/kernel/rmap-vm/2.5.26/2.5.26-rmap-1-core) 或者 [CODE mm/rmap.c, v2.5.27](https://elixir.bootlin.com/linux/v2.5.27/source/mm/rmap.c). 论文参见 [Object-based Reverse Mapping](https://landley.net/kdocs/ols/2004/ols2004v2-pages-71-74.pdf)

社区一直在努力试图优化 (PTE-based) RMAP 的整体效率.

*   object-based reverse mapping(objrmap)

在 2003 年, IBM 的 Dave McCracken 提出了一种新的解决方法 [** 基于对象的反向映射 **("object-based reverse mapping")](https://lwn.net/Articles/23732), 简称 objrmap. 虽然这组补丁当时未合入主线, 但是向社区证实了, 从 struct page 找到映射该物理页的页表项 PTE. 该方法虽然存在一些问题, 但是测试证明可以显著解决 RMAP 在部分场景的巨大开销问题. 参见 [Performance of partial object-based rmap](https://lkml.org/lkml/2003/2/19/235).

用户空间进程的页面主要有两种, 一种是 file mapped page, 另外一种是 anonymous mapped page. Dave McCracken 的 objrmap 方案虽然性能不错(保证逆向映射功能的基础上, 同时又能修正 rmap 带来的各种问题). 但是只是适用于 file mapped page, 对于匿名映射页面, 这个方案无能为力.

因此 Hugh Dickins 在 [2003 年](https://lore.kernel.org/patchwork/patch/14844) ~ [2004 年](https://lore.kernel.org/patchwork/patch/22938)的提交了[一系列补丁](https://lore.kernel.org/patchwork/patch/23565), 试图在 Dave McCracken 补丁的基础上, 进一步优化 RMAP 在匿名映射页面上的问题.

与此同时, SUSE 的 Andrea Arcangeli 当时正在做的工作就是让 32-bit 的 Linux 运行在配置超过 32G 内存的公司服务器上. 在这些服务器上往往启动大量的进程, 共享了大量的物理页帧, 消耗了大量的内存. 对于 Andrea Arcangeli 来说, 内存消耗的罪魁祸首是非常明确的 : 反向映射模块, 这个模块消耗了太多的 low memory, 从而导致了系统的各种 crash. 为了让自己的工作继续推进, 他必须解决 rmap 引入的内存扩展性 (memory scalability) 问题, 最直接有效的思路同样是在 Dave McCracken's objrmap 的基础上, 为匿名映射页面设计一种基于对象的逆向映射机制. 最后 Andrea Arcangeli [设计了 full objrmap 方案](https://lwn.net/Articles/77106). 并在与 Hugh Dickins 类似方案的 PK 中胜出. 并最终在 [v2.6.7](https://elixir.bootlin.com/linux/v2.6.7/source/mm/rmap.c#322) 合入主线.

> 小故事:
>
> 关于 32-bit 的 Linux 内核是否支持 4G 以上的 memory. 在 1999 年, Linus 的决定是 :
>
> 1999 年, linus 说: 32 位 linux 永远不会, 也不需要支持大于 2GB 的内存, 没啥好商量的.
>
> 不过历史的洪流不可阻挡, 处理器厂商设计了扩展模块以便寻址更多的内存, 高端的服务器也配置了越来越多的内存.
>
> 这也迫使 Linus 改变之前的思路, 让 Linux 内核支持更大的内存.


通过新增结构 anon_vma, 类似于重用 address_space 的想法, 即拥有一个数据结构 trampoline.
一个 VMA 中的所有页面只共享一个 anon_vma. vma->anon_vma 表示 vma 是否映射了页面. 相关的处理在 do_anonymous_fault()-=>anon_vma_prepare().

相对于基本的基于 PTE-chain 的解决方案, 基于对象的 RMAP :

| 比较 | 描述 |
|:---:|:----:|
| 优势 |  在页面错误期间, 我们只需要设置 page->映射为指向 anon_vma 结构, 而不是分配一个新结构并插入.
| 不足 | 在 rmap 遍历期间, 我们需要额外的计算来遍历每个 VMA 的页表, 以确保该页实际上被映射到这个特定的 VMA 中. |


*    anon_vma_chain(AVC-per process anon_vma)

旧的机制下, 多个进程共享一个 AV(anon_vma), 这可能导致大量分支工作负载的可伸缩性问题. 具体来说, 由于每个 anon_vma 将在父进程和它的所有子进程之间共享.

1.  这样在进程 fork 子进程的场景下, 如果进行 COW, 父子进程原本共享的 page frame 已经不再共享, 然而, 这两个 page 却仍然指向同一个 anon_vma.

2.  即使一开始就没有在父子进程之间共享的页面, 当首次访问的时候(无论是父进程还是子进程), 通过 do_anonymous_page 函数分配的 page frame 也是同样的指向一个 anon_vma.


在有些网路服务器中, 系统非常依赖 fork, 某个服务程序可能会 fork 巨大数量的子进程来处理服务请求, 在这种情况下, 系统性能严重下降. Rik van Riel 给出了一个具体的示例: 系统中有 1000 进程, 都是通过 fork 生成的, 每个进程的 VMA 有 1000 个匿名页. 根据目前的软件架构, anon_vma 链表中会有 1000 个 vma 的节点, 而系统中有一百万个匿名页面属于同一个 anon_vma.


这可能导致这样的系统 : 一个 CPU 遍历 page_referenced_one 中的 1000 个进程的页表, 而其他所有 CPU 都被 anon_vma 锁卡住. 这将导致像 AIM7 这样的基准测试出现灾难性的故障, 在那里进程的总数可能达到数万个. 实际工作量仍然比 AIM7 少 10 倍, 但它们正在迎头赶上.

这个补丁改变了 anon_vmas 和 VMA 的链接方式, 它允许我们将多个 anon_vmas 与一个 VMA 关联起来. 在分叉时, 每个子进程都获得自己的 anon_vmas, 它的 COW 页面将在其中实例化. 父类的 anon_vma 也链接到 VMA, 因为非 cowed 页面可以出现在任何子类中. 这将 1000 个子进程的页面的 RMAP 扫描复杂度降低到 O(1), 系统中最多 1/N 个页面的 RMAP 扫描复杂度为 O(N). 这将把繁重的工作负载从 O(N)减少到很小的平均扫描成本.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2002/07/20 | Rik van Riel <riel@redhat.com> | [VM with reverse mappings](https://www.cs.helsinki.fi/linux/linux-kernel/2002-46/0281.html) | 反向映射 RMAP 机制 | ☑ 2.5.27 | [Patch Archive](http://loke.as.arizona.edu/~ckulesa/kernel/rmap-vm/2.5.31/readme.html), [LWN](https://lwn.net/Articles/2925) |
| 2003/02/20 | Dave McCracken <dmccr@us.ibm.com> | [The object-based reverse-mapping VM](https://lwn.net/Articles/23732) | 基于对象的反向映射技术 | ☐ 2.5.62 in Andrew Morton's tree | [LWN](https://lwn.net/Articles/23584) |
| 2003/04/02 | Dave McCracken <dmccr@us.ibm.com> | [Optimizaction for object-based rmap](https://lore.kernel.org/patchwork/patch/15415) | 优化基于对象的反向映射 | ☐ 2.5.66 | [PatchWork](https://lore.kernel.org/patchwork/patch/15415) |
| 2003/03/20 | Hugh Dickins <hugh@veritas.com> | [anobjrmap 0/6](https://lore.kernel.org/patchwork/patch/14844) | 扩展了 Dave McCracken 的 objrmap 来处理匿名内存 | ☐ 2.5.66 | [PatchWork for 6 patches 2003/03/20](https://lore.kernel.org/patchwork/patch/14844)<br>*-*-*-*-*-*-*-* <br>[PatchWork for 2.6.5-rc1 8 patches 2004/03/18](https://lore.kernel.org/patchwork/patch/22938)<br>*-*-*-*-*-*-*-* <br>[PatchWork for 2.6.5-mc2 40 patches 2004/04/08](https://lore.kernel.org/patchwork/patch/23565) |
| 2004/03/10 | Andrea Arcangeli <andrea@suse.de> | [Virtual Memory II: full objrmap](https://lwn.net/Articles/75198) | 基于对象的反向映射技术的有一次尝试 | v1 ☑ [2.6.7](https://elixir.bootlin.com/linux/v2.6.7/source/mm/rmap.c#322) | [230-objrmap fixes for 2.6.3-mjb2](https://lore.kernel.org/patchwork/patch/22524)<br>*-*-*-*-*-*-*-* <br>[objrmap-core-1](https://lore.kernel.org/patchwork/patch/22623)<br>*-*-*-*-*-*-*-* <br>[RFC anon_vma previous (i.e. full objrmap)](https://lore.kernel.org/patchwork/patch/22674)<br>*-*-*-*-*-*-*-* <br>[anon_vma RFC2](https://lore.kernel.org/patchwork/patch/22694)<br>*-*-*-*-*-*-*-* <br>[anon-vma](https://lore.kernel.org/patchwork/patch/23307) |
| 2009/11/24 | Hugh Dickins <hugh.dickins@tiscali.co.uk> | [ksm: swapping](https://lore.kernel.org/patchwork/patch/179157) | 内核 Samepage 归并 (KSM) 是 Linux 2.6.32 中合并的一个特性, 它对虚拟客户机的内存进行重复数据删除. 但是, 该实现不允许交换共享的页面. 这个版本带来了对 KSM 页面的交换支持.<br> 这个对 RMAP 的影响是, RMAP 支持 KASM 页面. 同时用 rmap_walk 的方式 | v1 ☑ [2.6.33-rc1](https://kernelnewbies.org/Linux_2_6_33#Swappable_KSM_pages) | [PatchWork v1](https://lore.kernel.org/patchwork/patch/179157) |
| 2010/01/14 | Rik van Riel <riel@redhat.com> | [change anon_vma linking to fix multi-process server scalability issue](https://lwn.net/Articles/75198) | 引入 AVC, 实现了 per process anon_vma, 每一个进程的 page 都指向自己特有的 anon_vma 对象. | v1 ☑ [2.6.34](https://kernelnewbies.org/Linux_2_6_34#Various_core_changes) | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/185210)<br>*-*-*-*-*-*-*-* <br>[PatchWork v1](https://lore.kernel.org/patchwork/patch/186566) |
| 2013/12/4 | Rik van Riel <riel@redhat.com> | [mm/rmap: unify rmap traversing functions through rmap_walk](https://lwn.net/Articles/424318) | 引入 AVC, 实现了 per process anon_vma, 每一个进程的 page 都指向自己特有的 anon_vma 对象. | v1 ☑ 3.4-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/patch/424318) |
| 2012/12/01 | Davidlohr Bueso <davidlohr@hp.com> | [mm/rmap: Convert the struct anon_vma::mutex to an rwsem](https://lore.kernel.org/patchwork/patch/344816) | 将 struct anon_vma::mutex 转换为 rwsem, 这将有效地改善页面迁移过程中对 VMA 的访问的竞争问题. | v1 ☑ 3.8-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/344775)<br>*-*-*-*-*-*-*-* <br>[PatchWork](https://lore.kernel.org/patchwork/patch/344816) |
| 2014/05/23 | Davidlohr Bueso <davidlohr@hp.com> | [mm: i_mmap_mutex to rwsem](https://lore.kernel.org/patchwork/patch/466974) | 2012 年 Ingo 将 anon-vma lock 从 mutex 转换到了 rwsem 锁. i_mmap_mutex 与 anon-vma 有类似的职责, 保护文件支持的页面. 因此, 我们可以使用类似的锁定技术 : 将互斥锁转换为 rwsem, 并在可能的情况下共享锁. | RFC ☐ | [PatchWork](https://lore.kernel.org/patchwork/patch/466974) |
| 2014/05/23 | Davidlohr Bueso <davidlohr@hp.com> | [mm/rmap.c: Avoid double faults migrating device private pages](https://patchwork.kernel.org/project/linux-mm/patch/20211018045247.3128058-1-apopple@nvidia.com) | 在迁移过程中, 将为要迁移的每个页面安装特殊的页面表条目. 这些条目存储 pfn 和映射被混合页面的 PTE 的相关权限.<br> 设备专用页使用特殊的交换 pte 项来区分只读页和可写页, 迁移代码在创建迁移项时会检查这些页. 通常, 这遵循 migrate_vma_collect_pmd() 中的快速路径, 该路径在将页面迁移回 CPU 时将设备专用页面的权限正确复制到迁移条目.<br> 但是, 慢的路径是使用 try_to_migrate(), 它无条件地为设备专用页创建只读迁移条目. 这会导致 CPU 上出现不必要的双重错误, 因为新页面始终以只读方式映射, 即使它们可以被映射为可写. 通过在 try_to_migrate_one() 中正确复制设备私有权限来修复此问题. | v1 ☐ | [PatchWork](https://patchwork.kernel.org/project/linux-mm/patch/20211018045247.3128058-1-apopple@nvidia.com) |


## 8.4 mremap 重新映射虚拟地址
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| NA | Linus Torvalds <torvalds@linuxfoundation.org> | [Import 1.3.78](https://lore.kernel.org/patchwork/patch/1324435) | filemap 的批量内存分配和拷贝. | v4 ☑ 1.3.78 | [COMMIT HISTOR](http://github.com/gatieme/linux-history/commit/24747a70c81a376046a1ad00422142a6bb8ed7e9) |
| 2020/11/08 | Linus Torvalds <torvalds@linuxfoundation.org> | [Add support for fast mremap](https://patchwork.kernel.org/project/linux-mm/cover/20181108181201.88826-1-joelaf@google.com) | mremap PMD 映射优化. 在内存管理相关的操作中, Android 需要 mremap 大区域的内存. 如果不启用 THP, mremap 系统调用会非常慢. 瓶颈是 move_page_tables, 它一次复制每个 pte, 在一个大的映射中可能非常慢. 开启 THP 可能不是一个可行的选择, 也不适合我们. 因此实现了 fast remap 通过尽可能在 PMD 级别进行复制来加快非 THP 系统的性能. 速度在 x86 上是提升了一个数量级(~20 倍). 在 1GB 的 mremap 上, mremap 完成时间从 3.4-3.6 毫秒下降到 144-160 微秒. | v4 ☑ 1.3.78 | [PatchWork v1,1/2](https://patchwork.kernel.org/project/linux-mm/patch/20181012013756.11285-1-joel@joelfernandes.org)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2,0/3](https://patchwork.kernel.org/project/linux-mm/cover/20181013013200.206928-1-joel@joelfernandes.org)<br>*-*-*-*-*-*-*-* <br>[PatchWork v5,0/3](https://patchwork.kernel.org/project/linux-mm/cover/20181108181201.88826-1-joelaf@google.com)<br>*-*-*-*-*-*-*-* <br>[COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2c91bd4a4e2e530582d6fd643ea7b86b27907151) |
| 2020/10/14 | Kalesh Singh <kaleshsingh@google.com> | [Speed up mremap on large regions](https://lore.kernel.org/patchwork/patch/1321018) | mremap PUD 映射优化, 可以加速映射大块地址时的速度. 如果源地址和目的地址是 PMD/PUD 对齐和 PMD/PUD 大小, mremap 的耗时时间可以通过移动 PMD/PUD 级别的表项来优化. 允许在 arm64 和 x86 上的 PMD 和 PUD 级别移动. 其他支持这种类型移动且已知安全的体系结构也可以通过启用 `HAVE_MOVE_PMD` 和 `HAVE_MOVE_PUD` 来选择这些优化. | v4 ☑ | [PatchWork v4,0/5](https://patchwork.kernel.org/project/linux-mm/cover/20201014005320.2233162-1-kaleshsingh@google.com) |
| 2020/10/14 | Kalesh Singh <kaleshsingh@google.com> | [Speedup mremap on ppc64](https://patchwork.kernel.org/project/linux-mm/cover/20210616045735.374532-1-aneesh.kumar@linux.ibm.com) | NA | v8 ☑ 5.14-rc1 | [PatchWork v8,0/3](https://patchwork.kernel.org/project/linux-mm/cover/20201014005320.2233162-1-kaleshsingh@google.com) |


## 8.5 ioremap
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2006/08/10 | Haavard Skinnemoen <hskinnemoen@atmel.com> | [Generic ioremap_page_range: introduction](https://lore.kernel.org/patchwork/patch/62430) | 基于 i386 实现的 ioremap_page_range() 的通用实现, 将 I/O 地址空间映射到内核虚拟地址空间. | v1 ☑ 2.6.19-rc1 | [PatchWork 0/14](https://lore.kernel.org/patchwork/patch/62430) |
| 2015/03/03 | Toshi Kani <toshi.kani@hp.com> | [Kernel huge I/O mapping support](https://lore.kernel.org/patchwork/patch/547056) | ioremap() 支持透明大页. 扩展了 ioremap() 接口, 尽可能透明地创建具有大页面的 I/O 映射. 当一个大页面不能满足请求范围时, ioremap() 继续使用 4KB 的普通页面映射. 使用 ioremap() 不需要改变驱动程序. 但是, 为了使用巨大的页面映射, 请求的物理地址必须以巨面大小 (x86 上为 2MB 或 1GB) 对齐. 内核巨页的 I/O 映射将提高 NVME 和其他具有大内存的设备的性能, 并减少创建它们映射的时间. | v3 ☑ 4.1-rc1 | [PatchWork v3,0/6](https://lore.kernel.org/patchwork/patch/547056) |
| 2015/05/15 | Haavard Skinnemoen <hskinnemoen@atmel.com> | [mtrr, mm, x86: Enhance MTRR checks for huge I/O mapping](https://lore.kernel.org/patchwork/patch/943736) | 增强了对巨页 I/O 映射的 MTRR 检查.<br>1. 允许 pud_set_huge() 和 pmd_set_huge() 创建一个巨页映射, 当范围被任何内存类型的单个 MTRR 条目覆盖时. <br>2. 当指定的 PMD 映射范围超过一个 MTRR 条目时, 记录 pr_warn_once() 消息. 当这个范围被 MTRR 覆盖时, 驱动程序应该发出一个与单个 MTRR 条目对齐的映射请求. | v5 ☐ | [PatchWork v5,0/6](https://lore.kernel.org/patchwork/patch/943736) |




# 9 内存控制组 MEMCG (Memory Cgroup)支持
-------

[KS2012: The memcg/mm minisummit](https://lwn.net/Articles/516439)

[Controlling memory use in containers](https://lwn.net/Articles/243795)

**2.6.25(2008 年 4 月发布)**

在 Linux 轻量级虚拟化的实现 container 中 (比如现在挺火的 Docker, 就是基于 container), 一个重要的功能就是做资源隔离. Linux 在 2.6.24 中引入了 cgroup(control group, 控制组) 的资源隔离基础框架(将在最后一个部分详述), 提供了资源隔离的基础.

在 2.6.25 中, 内核在此基础上支持了内存资源隔离, 叫内存控制组. 它使用可以在不同的控制组中, 实施内存资源控制, 如分配, 内存用量, 交换等方面的控制.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/08/24 | Balbir Singh <balbir@linux.vnet.ibm.com> | [Memory controller containers setup (v7)](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=e1a1cd590e3fcb0d2e230128daf2337ea55387dc) | 实现 MEMCG | v7 ☑ [2.6.25-rc1](https://kernelnewbies.org/Linux_2_6_25) | [PatchWork v7](https://lore.kernel.org/lkml/20070824152009.16582.78904.sendpatchset@balbir-laptop) |
| 2010/09/01 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg: towards I/O aware memcg v7.](https://lore.kernel.org/patchwork/patch/213968) | IO 感知的 MEMCG. | v7 ☐ | [PatchWork v7](https://lore.kernel.org/patchwork/patch/213968) |
| 2009/09/25 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg updates v5](https://lore.kernel.org/patchwork/patch/129608) | IO 感知的 MEMCG. | v7 ☐ | [PatchWork 0/12](https://lore.kernel.org/patchwork/patch/129608) |
| 2009/06/15 | Balbir Singh <balbir@linux.vnet.ibm.com>| [Remove the overhead associated with the root cgroup](https://lore.kernel.org/patchwork/patch/160500) | 通过删除与 root cgroup 有关的开销来降低 mem cgroup 的开销 <br>1. 删除了与计算 root cgroup 中所有页面相关的开销. 作为一个副作用, 我们不能再在 root cgroup 中设置内存硬限制.<br>2. 添加了一个新的标记 PCG_ACCT_LRU, 用于跟踪页面是否已被计入. page_cgroup 的标记现在被原子地设置, pcg_default_flags 现在已经过时并被删除. | v5 ☑ 2.6.32-rc1 | [PatchWork v5](https://lore.kernel.org/patchwork/patch/160500) |
| 2009/07/10 | Balbir Singh <balbir@linux.vnet.ibm.com>| [Memory controller soft limit patches (v9)](https://lore.kernel.org/patchwork/patch/163652) | 实现内存资源控制器的软限制.<br> 软限制是内存资源控制器的一个新特性, 类似的东西已经以共享的形式存在于组调度程序中. CPU 控制器对共享的解释是非常不同的. 对于管理员希望过度使用系统的环境, 软限制是最有用的特性, 这样只有在内存争用时, 限制才会生效. 当前的软限制实现为内存控制器提供了 soft_limit_in_bytes 接口, 而不是为内存 + 交换控制器提供的接口. 该实现维护一个 RB-Tree, 其中的组超过了其软限制, 并开始从超出该限制的组中回收最大数量的组. | v9 ☑ 2.6.32-rc1 | [PatchWork RFC,0/5](https://lore.kernel.org/patchwork/patch/163652) |
| 2022/03/31 | zhaoyang.huang <zhaoyang.huang@unisoc.com> | [[RFC] cgroup: introduce dynamic protection for memcg](https://patchwork.kernel.org/project/linux-mm/patch/1648713656-24254-1-git-send-email-zhaoyang.huang@unisoc.com) | 动态限制的 MEMCG.<br> 对于某些类型的 MEMCG, 使用情况因场景而异. 例如, 多媒体应用的使用范围可能在 50MB 到 500MB 之间, 这是通过在其虚拟地址空间中加载特殊算法生成的, 如果没有用户空间的交互, 很难保护扩展的使用. 此外, 固定的 mem.low 有点违背了它的软保护作用, 因为它会以同样的方式响应任何系统的内存压力. 在此基础上, 提出了一种于组水印和系统内存压力的动态保护方案. 目标是: 1. 动态保护, 无固定设置限值. 2. 正确的内存保护值. 3. 基于时间的衰变保护 4. 记忆压力相关保护.<br> 引入 elow, emin<br>1. 在保护方面, elow 是根据水印的比例在适当的范围内.<br>2. 经过时间通过 decayed_watermark 对 elow 有积极的影响.<br>3. 内存压力对 low 有负面影响, 当系统压力较小时, low 可以保持更多的使用. | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/all/1648713656-24254-1-git-send-email-zhaoyang.huang@unisoc.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/all/1649040193-10073-1-git-send-email-zhaoyang.huang@unisoc.com) |



## 9.1 PageCacheLimit
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/01/15 | Roy Huang <royhuang9@gmail.com> | [Provide an interface to limit total page cache.](https://lore.kernel.org/patchwork/patch/72078) | 限制 page cache 的内存占用. | RFC ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/72078) |
| 2007/01/24 | Christoph Lameter <clameter@sgi.com> | [Limit the size of the pagecache](https://lore.kernel.org/patchwork/patch/72581) | 限制 page cache 的内存占用. | RFC ☐ | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/72581) |
| 2014/06/16 | Xishi Qiu <qiuxishi@huawei.com> | [mm: add page cache limit and reclaim feature](https://lore.kernel.org/patchwork/patch/473535) | openEuler 上限制 page cache 的内存占用. | v1 ☐ | [PatchWork](https://lore.kernel.org/patchwork/patch/416962)<br>*-*-*-*-*-*-*-* <br>[openEuler 4.19](https://gitee.com/openeuler/kernel/commit/6174ecb523613c8ed8dcdc889d46f4c02f65b9e4) |
| 2019/02/23 | Chunguang Xu <brookxu@tencent.com> | [pagecachelimit: limit the pagecache ratio of totalram](http://github.com/tencent/TencentOS-kernel/commit/6711b34671bc658c3a395d99aafedd04a4ebbd41) | TencentOS 限制 page cache 的内存占用. | NA |  [pagecachelimit: limit the pagecache ratio of totalram](http://github.com/tencent/TencentOS-kernel/commit/6711b34671bc658c3a395d99aafedd04a4ebbd41) |

## 9.2 Cgroup-Aware OOM killer
-------

[Taming the OOM killer](https://lwn.net/Articles/317814)

[Teaching the OOM killer about control groups](https://lwn.net/Articles/761118)


*   MEMCG Priority

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/03/17 | Roman Gushchin <guro@fb.com> | [add support for reclaiming priorities per mem cgroup](https://lore.kernel.org/all/20170317231636.142311-1-timmurray@google.com) | 设置 CGROUP 的内存优先级, Kswapd 等优先扫描低优先级 CGROUP 来回收内存. 用于优化 Android 上的内存回收和 OOM 等机制 | v13 ☐ | [PatchWork RFC v1](https://lore.kernel.org/all/20170317231636.142311-1-timmurray@google.com) |
| 2021/04/14 | Yulei Zhang <yuleixzhang@tencent.com> | [introduce new attribute"priority"to control group](https://lore.kernel.org/patchwork/patch/828043) | Cgroup 引入优先级. | v13 ☐ | [PatchWork v1](https://lwn.net/Articles/851649)<br>*-*-*-*-*-*-*-* <br>[LWN](https://lwn.net/Articles/852378) |

*   OOM Killer

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/09/04 | Tim Murray <timmurray@google.com> | [cgroup-aware OOM killer](https://lore.kernel.org/patchwork/patch/828043) | Cgroup 感知的 OOM, 通过优先级限定 OOM 时杀进程的次序 | v13 ☐ | [PatchWork v7](https://lore.kernel.org/patchwork/patch/828043) 带 oom_priority<br>*-*-*-*-*-*-*-* <br>[PatchWork v13](https://lore.kernel.org/patchwork/patch/828043) |
| 2018/03/16 | David Rientjes <rientjes@google.com> | [rewrite cgroup aware oom killer for general use](https://lore.kernel.org/patchwork/patch/828043) | Cgroup 感知的 OOM, 通过优先级限定 OOM 时杀进程的次序 | v13 ☐ | [PatchWork v1](https://lore.kernel.org/patchwork/patch/934536) |
| 2021/03/25 | Ybrookxu | [bfq: introduce bfq.ioprio for cgroup](https://lore.kernel.org/patchwork/patch/828043) | Cgroup 感知的 bfq.ioprio | v3 ☐ | [LKML](https://lkml.org/lkml/2021/3/25/93) |
| 2022/04/21 | Nico Pache <npache@redhat.com> | [Slight improvements for OOM/Futex](https://patchwork.kernel.org/project/linux-mm/cover/20220421190533.1601879-1-npache@redhat.com/) | 634339 | v1 ☐☑ | [LORE v1,0/3](https://lore.kernel.org/r/20220421190533.1601879-1-npache@redhat.com) |


*   Killed Threads

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/03/25 | Tetsuo Handa <penguin-kernel@I-love.SAKURA.ne.jp> | [memcg: killed threads should not invoke memcg OOM killer](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7775face207922ea62a4e96b9cd45abfdc7b9840) | 如果当前线程在等待 oom_lock 时被杀死, 我们不需要调用 memcg OOM Killer | v3 ☐ | [PatchWork](https://patchwork.kernel.org/project/linux-mm/patch/01370f70-e1f6-ebe4-b95e-0df21a0bc15e@i-love.sakura.ne.jp), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7775face207922ea62a4e96b9cd45abfdc7b9840) |
| 2021/10/23 | Vasily Averin <vvs@virtuozzo.com> | [memcg: prohibit unconditional exceeding the limit of dying tasks](https://patchwork.kernel.org/project/linux-mm/patch/b89715b5-6df7-34a3-f7b9-efa8e0eefd3e@virtuozzo.com) | 内存 cgroup charge 的过程中允许杀死或退出的任务超过硬限制. 假设这些任务占用的内存数量是绑定的, 并且在任务退出时, 大部分内存将被释放. 这类似于任务访问内存储备时的全局 OOM 情况的启发式策略. 在 memcg 水平上没有全局内存短缺, 所以 memcg 启发式比较轻松.<br> 不过, 上述假设过于乐观. 例如:<br>1. vmalloc 可以扩展到真正大的请求, 启发式将允许这一点. 我们曾经在 vmalloc 分配器中有一个被杀死的任务的早期中断, 但这已经被提交 b8c8a338f75e ("Revert vmalloc: back off when the current task is killed"). 所恢复. 可能还有其他类似的代码路径在分配和充电循环中不检查致命信号.<br>2. 还有一些内核对象统计到给 memcg, 但是它们没有绑定到进程生命周期. 据观察, 触发这些旁路并造成全球 OOM 情况并非真的很难.<br> 解决这些问题的一种潜在方法是限制过剩的数量(类似于有限房间储备的全局 OOM). 这当然是可能的, 但我们并不清楚需要多少额外的内存, 并且仍然可以防止全局 oom, 因为这将不得不考虑整个 memcg 配置.<br> 这个补丁通过完全删除启发式的策略来解决这个问题. 绕过只允许请求, 要么不能失败, 或失败是不可取的, 而超出应该仍然受到限制. 实现一个被杀死或即将死亡的任务, 如果它已经通过 OOM 杀手阶段, 就不能再被 charge. | v3 ☐ | [2021/10/20 PatchWork RFC,0/2](https://patchwork.kernel.org/project/linux-mm/cover/fe1d45a1-276d-b0f4-fb71-8f5c1a9e8872@virtuozzo.com/)<br>*-*-*-*-*-*-*-* <br>[2021/10/05 PatchWork v2,0/2](https://patchwork.kernel.org/project/linux-mm/cover/21d79b86-476c-a8f2-e950-2339606f1253@virtuozzo.com)<br>*-*-*-*-*-*-*-* <br>[2021/10/20 PatchWork v3,0/2](https://patchwork.kernel.org/project/linux-mm/cover/20e50917-3589-bcb7-0174-b6fccfd15c66@virtuozzo.com) |

*   Cpuset OOM

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/09/21 | Gang Li <ligang.bdlg@bytedance.com> | [[RFC,v1] mm: oom: introduce cpuset oom](https://patchwork.kernel.org/project/linux-mm/patch/20220921064710.89663-1-ligang.bdlg@bytedance.com/) | 678906 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220921064710.89663-1-ligang.bdlg@bytedance.com) |


## 9.3 KMEMCG
-------

[memcg kernel memory tracking](https://lwn.net/Articles/482777)

```cpp
git://git.kernel.org/pub/scm/linux/kernel/git/glommer/memcg.git kmemcg-stac
```

2012-04-20 [slab+slub accounting for memcg 00/23](https://lore.kernel.org/patchwork/patch/298625)
2012-05-11 [kmem limitation for memcg (v2,00/29)](https://lore.kernel.org/patchwork/patch/302291)
2012-05-25 [kmem limitation for memcg (v3,00/28)](https://lore.kernel.org/patchwork/patch/304994)
2012-06-18 [kmem limitation for memcg (v4,00/25)](https://lore.kernel.org/patchwork/patch/308973)
2012-06-25 [kmem controller for memcg: stripped down version](https://lore.kernel.org/patchwork/patch/310278)
2012-09-18 [kmem controller for memcg (v3,00/13)](https://lore.kernel.org/patchwork/patch/326975)
2012-10-08 [kmem controller for memcg (v4,00/14)](https://lore.kernel.org/patchwork/patch/331578)
2012-10-16 [kmem controller for memcg (v5,00/14](https://lore.kernel.org/patchwork/patch/333535)

```cpp
git://github.com/glommer/linux.git kmemcg-slab
```

2021-07-25 [memcg kmem limitation - slab.](https://lore.kernel.org/patchwork/patch/315827)
2012-08-09 [Request for Inclusion: kmem controller for memcg (v2, 00/11)](https://lore.kernel.org/patchwork/patch/318976)
2012-09-18 [slab accounting for memcg (v3,00/16)](https://lore.kernel.org/patchwork/patch/326976)
2012-10-19 [slab accounting for memcg (v5,00/18)](https://lore.kernel.org/patchwork/patch/334479)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2012/03/09 |  Suleiman Souhlal <ssouhlal@FreeBSD.org> | [Memcg Kernel Memory Tracking](https://lwn.net/Articles/485593) | memcg 支持对内核 STACK 进行统计 | v2 ☐ | [PatchWork v2](https://lore.kernel.org/patchwork/patch/291219) |
| 2012/10/16 | Glauber Costa <glommer@parallels.com> | [kmemcg-stack](https://lwn.net/Articles/485593) | memcg 支持对内核 STACK 进行统计 | v5 ☐ | [PatchWork v5](https://lore.kernel.org/patchwork/patch/333535) |
| 2012/10/19 | Glauber Costa <glommer@parallels.com> | [kmemcg-stack](https://lwn.net/Articles/485593) | memcg 支持对内核 SLAB 进行统计 | v5 ☐ | [PatchWork v5](https://lore.kernel.org/patchwork/patch/334479) |
| 2015/11/10 | Vladimir Davydov <vdavydov@virtuozzo.com> | [memcg/kmem: switch to white list policy](https://lore.kernel.org/patchwork/patch/616606/) | NA | v2 ☑ 4.5-rc1 | [PatchWork v2,0/6](https://lore.kernel.org/patchwork/patch/616606) |
| 2020/06/23 | Glauber Costa <glommer@parallels.com> | [kmem controller for memcg](https://lwn.net/Articles/516529) | memcg 支持对内核内存 (kmem) 进行统计, memcg 开始支持统计两种类型的内核内存使用 : 内核栈 和 slab. 这些限制对于防止 fork 炸弹 (bombs) 等事件很有用. | v6 ☑ 3.8-rc1 | [PatchWork v5 kmem(stack) controller for memcg](https://lore.kernel.org/patchwork/patch/333535), [PatchWork v5 slab accounting for memcg](https://lore.kernel.org/patchwork/patch/334479)<br>*-*-*-*-*-*-*-* <br>[PatchWork v6](https://lore.kernel.org/patchwork/patch/337780) |
| 2020/06/23 |  Roman Gushchin <guro@fb.com> | [The new cgroup slab memory controller](https://lore.kernel.org/patchwork/patch/1261793) | 将 SLAB 的统计跟踪统计从页面级别更改为到对象级别. 它允许在 memory cgroup 之间共享 SLAB 页. 这一变化消除了每个 memory cgroup 每个重复的每个 cpu 和每个节点 slab 缓存集, 并为所有内存控制组建立了一个共同的每个 cpu 和每个节点 slab 缓存集. 这将显著提高 SLAB 利用率(最高可达 45%), 并相应降低总的内核内存占用. 测试发现不可移动页面数量的减少也对内存碎片产生积极的影响. | v7 ☑ 5.9-rc1 | [PatchWork v7](https://lore.kernel.org/patchwork/patch/1261793) |
| 2015/11/10 | Roman Gushchin <guro@fb.com> | [memcg/kmem: switch to white list policy](https://lore.kernel.org/patchwork/patch/616606) | 所有的 kmem 分配 (即每次 kmem_cache_alloc、kmalloc、alloc_kmem_pages 调用) 都会自动计入内存 cgroup. 如果出于某些原因, 呼叫者必须明确选择退出. 这样的设计决策会导致以下许多个问题, 因此该补丁切换为白名单策略. 现在 kmalloc 用户必须通过传递 __GFP_ACCOUNT 标志来显式标记. | v7 ☑ 5.9-rc1 | [PatchWork v7](https://lore.kernel.org/patchwork/patch/616606) |
| 2021/05/06 |  Waiman Long <longman@redhat.com> | [The new cgroup slab memory controller](https://lore.kernel.org/patchwork/patch/1422112) | [The new cgroup slab memory controller](https://lore.kernel.org/patchwork/patch/1261793) 合入后, 不再需要为每个 MEMCG 使用单独的 kmemcache, 从而减少了总体的内核内存使用. 但是, 我们还为每次调用 kmem_cache_alloc() 和 kmem_cache_free() 增加了额外的内存开销. 参见 [10befea91b:  hackbench.throughput -62.4% regression](https://lore.kernel.org/lkml/20210114025151.GA22932@xsang-OptiPlex-9020) 和 [memcg: performance degradation since v5.9](https://lore.kernel.org/linux-mm/20210408193948.vfktg3azh2wrt56t@gabell/T/#u). 这组补丁通过降低了 kmemcache 统计的开销. 缓解了这个问题. | v7 ☑ 5.9-rc1 | [PatchWork v7](https://lore.kernel.org/patchwork/patch/1422112) |
| 2020/12/20 | Shakeel Butt <shakeelb@google.com> | [inotify, memcg: account inotify instances to kmemcg](https://lore.kernel.org/patchwork/patch/1355497) | 目前 sysctl inotify/max_user_instances 用于限制系统的 inotify 实例数量. 对于运行多个工作负载的系统, 每个用户的命名空间 sysctl max_inotify_instances 可以用于进一步划分 inotify 实例. 但是, 没有简单的方法可以对 inotify 实例设置合理的系统级别最大限制, 并在工作负载之间对其进行进一步分区. 该补丁通过将 inotify 实例分配给 memcg, 管理员可以简单地将 max_user_instances 设置为 INT_MAX, 并让作业的 memcg 限制它们的 inotify 实例. | v2 ☑ [5.12-rc1](https://kernelnewbies.org/Linux_5.12#Memory_management) | [PatchWork v7](https://lore.kernel.org/patchwork/patch/1355497), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ac7b79fd190b02e7151bc7d2b9da692f537657f3) |
| 2014/02/15 | Vladimir Davydov <vdavydov@parallels.com> | [kmemcg shrinkers](https://lore.kernel.org/patchwork/patch/438717) | NA | v15 ☐ | [PatchWork -mm,v15,00/13](https://lore.kernel.org/patchwork/patch/438717) |
| 2013/04/08 | Glauber Costa <glommer@parallels.com> | [memcg-aware slab shrinking with lasers and numbers](https://lore.kernel.org/all/1365429659-22108-1-git-send-email-glommer@parallels.com) | SHRINKER_MEMCG_AWARE 的一次尝试. | v3 ☐☑✓ | [LORE v3,0/32](https://lore.kernel.org/all/1365429659-22108-1-git-send-email-glommer@parallels.com) |
| 2015/01/08 | Vladimir Davydov <vdavydov@parallels.com> | [Per memcg slab shrinkers](https://lore.kernel.org/patchwork/patch/438717) | 实现 SHRINKER_MEMCG_AWARE, memcg 的 kmem 统计现在无法使用, 因为它缺少 slab 的 shrink 支持. 这意味着当达到极限时, 将获得 ENOMEM, 而没有任何恢复的机会. 然后我们应该做的是调用 shrink_slab(), 它将从这个 cgroup 中回收旧的 inode/dentry 缓存, 这组补丁就完成了这个功能. 基本上, 它做两件事.<br>1. 首先, 引入了 per memcg slab shrinkers. 按 cgroup 回收对象的 shrinker 应将自身标记为 MEMCG AWARE. 接着在 shrink_control->memcg 中递归地对 memcg 进行扫描. 迭代目标 cgroup 下的整个 cgroup 子树, 并为每个 kmem 活动 memcg 调用 shrinker.<br>2. 其次, 该补丁集为每个 memcg 生成  list_lru. 这对 list_lru 是透明的, 他们所要做的一切就是告诉 list_lru_init() 他们想要有 memcg aware 的 list_lru. 然后, list_lru 将根据对象所属的 cgroup 自动在每个 memcg list 中分配对象. 为了让 FS 收缩器 (icache、dcache) 能够识别 memcg, 我们只需要让它们使用 memcg 识别 list_lru.<br> 与以前一样, 此修补程序集仅在压力来自 memory.limit 而不是 memory.kmem.limit 时启用 per-memcg kmem reclaim. 由于 GFP_NOFS 分配, 处理 memory.kmem.limit 是一个棘手的问题, 目前还不清楚我们是否会在统一的层次结构中使用这个旋钮. | v3 ☑ 4.0-rc1 | [PatchWork -mm,v3,0/9](https://lore.kernel.org/all/cover.1420711973.git.vdavydov@parallels.com), [关键 commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cb731d6c62bbc2f890b08ea3d0386d5dad887326) |
| 2022/02/11 | Shakeel Butt <shakeelb@google.com> | [memcg: robust enforcement of memory.high](https://patchwork.kernel.org/project/linux-mm/cover/20220210081437.1884008-1-shakeelb@google.com) | 612932 | v1 ☐ | [PatchWork v1,0/4](https://lore.kernel.org/all/20220210081437.1884008-1-shakeelb@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2,0/4](https://lore.kernel.org/r/20220211064917.2028469-1-shakeelb@google.com) |


弃用 kmem.limit_in_bytes

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/10/19 | Shakeel Butt <shakeelb@google.com> | [memcg, kmem: further deprecate kmem.limit_in_bytes](https://patchwork.kernel.org/project/linux-mm/patch/20211019153408.2916808-1-shakeelb@google.com) | kmem.limit_in_bytes 的弃用过程始于提交 commit 0158115f702("memcg, kmem:deprecate kmem.limit_in_bytes"), 这也详细解释了弃用背后的动机. 总而言之, 这是达到 kmem 极限时的意外行为. 此修补程序通过不允许设置 kmem 限制, 将弃用过程移动到下一阶段. 将来我们可能会完全删除 kmem.limit_in_bytes 文件. | v3 ☐ | [PatchWork v3](https://patchwork.kernel.org/project/linux-mm/patch/20211019153408.2916808-1-shakeelb@google.com) |

其他

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/02/01 | Yosry Ahmed <yosryahmed@google.com> | [memcg: add per-memcg total kernel memory stat](https://patchwork.kernel.org/project/linux-mm/patch/20220201200823.3283171-1-yosryahmed@google.com/) | 610482 | v1 ☐☑ | [PatchWork v1,0/1](https://lore.kernel.org/r/20220201200823.3283171-1-yosryahmed@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2,0/1](https://lore.kernel.org/r/20220203193856.972500-1-yosryahmed@google.com) |
| 2022/05/21 | Vasily Averin <vvs@openvz.org> | [memcg: accounting for objects allocated by mkdir cgroup](https://lore.kernel.org/all/06505918-3b8a-0ad5-5951-89ecb510138e@openvz.org) | TODO | v2 ☐☑✓ | [LORE v2,0/9](https://lore.kernel.org/all/06505918-3b8a-0ad5-5951-89ecb510138e@openvz.org) |
| 2022/06/28 | Yosry Ahmed <yosryahmed@google.com> | [KVM: mm: count KVM mmu usage in memory stats](https://patchwork.kernel.org/project/linux-mm/cover/20220628220938.3657876-1-yosryahmed@google.com/) | 654790 | v6 ☐☑ | [LORE v6,0/4](https://lore.kernel.org/r/20220628220938.3657876-1-yosryahmed@google.com) |


## 9.4 MEMCG LRU
-------

### 9.4.1 双 LRU 方案(per-memcg 的 per-zone LRU + 全局 LRU)
-------

自 2008 年 v2.6.25-rc1, Balbir Singh 实现 MEMCG 时, 就支持了 per-memcg 的 LRU,  [Memory controller: add per cgroup LRU and reclaim](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=66e1707bc34609f626e2e7b4fe7e454c9748bad5). 不过当时每个 MEMECG 只有一个唯一的 LRU, 且区分 active_list 和 inactive_list(Per cgroup active and inactive list). 引入了 [page_cgroup](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8cdea7c05454260c0d4d83503949c358eb131d17) 维护了 mem_cgroup 和 lru 的联系,. 并标记了 [TODO: Consider making these lists per zone](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=78fb74669e80883323391090e4d26d17fe29488f). 此时, per-memcg 的 lru_lock 也是 mem_cgroup 级别的.

而当时正是 per-Zone LRU 的时代, 随即 KAMEZAWA Hiroyuki 就为 per-memcg 的唯一 LRU 也完成了 per-Zone 的支持, 参见 [per-zone and reclaim enhancements for memory controller take 3](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=072c56c13e1302fcdc39961dc64e76485731ad6). 为了完成这个功能, 引入了 mem_cgroup_per_zone 等结构, 通过 `__mem_cgroup_add_list()` 和 `__mem_cgroup_remove_list()` 将每个 page_cgroup.lru 节点添加或者移除其对应 [mem_cgroup_per_zone 的 active_list 和 inactive_list](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1ecaab2bd221251a3fd148abb08e8b877f1e93c8), 同样完成了 [mem_cgroup_per_zone 的 lru_lock](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=072c56c13e1302fcdc39961dc64e76485731ad67).

这个阶段, root memcg 的 LRU 其实并没有用处, 因此可以不用把页面添加进去, [memcg: remove the overhead associated with the root cgroup](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4b3bde4c983de36c59e6c1a24701f6fe816f9f55).

至此内核为 per-memcg 实现了 per-Zone 的 LRU. 不过这时候的 LRU 是 双 LRU 方案(double-LRU scheme). 即除了 per-memcg 的 LRU 以外, 系统中还有全局的 LRU, 而且他们都是 Per-Zone 的.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/08/24 | Balbir Singh <balbir@linux.vnet.ibm.com> | [Memory controller: add per cgroup LRU and reclaim](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=66e1707bc34609f626e2e7b4fe7e454c9748bad5) | 实现 [Memory controller containers setup (v7)](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=e1a1cd590e3fcb0d2e230128daf2337ea55387dc) 时引入了 [per cgroup 的 LRU 和 页面 reclaim](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=66e1707bc34609f626e2e7b4fe7e454c9748bad5) 和  | v7 ☑ [2.6.25-rc1](https://kernelnewbies.org/Linux_2_6_25) | [PatchWork v7](https://lore.kernel.org/lkml/20070824152009.16582.78904.sendpatchset@balbir-laptop), [关注 commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=66e1707bc34609f626e2e7b4fe7e454c9748bad5). |
| 2007/11/26 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [per-zone and reclaim enhancements for memory controller take 3](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=072c56c13e1302fcdc39961dc64e76485731ad6) | per-zone LRU for memcg, per-zone 的页面回收感知 MEMCG. 其中引入了 mem_cgroup_per_zone, mem_cgroup_per_node 等结构. | v3 ☑ 2.6.25-rc1 | [LORE v3,00/10](https://lore.kernel.org/lkml/20071127115525.e9779108.kamezawa.hiroyu@jp.fujitsu.com) |


### 9.4.2 per-memecg 的 LRU lock 回退到全局的 zone lru_lock
-------

v2.6.29-rc1 [memcg: synchronized LRU](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=08e552c69c6930d64722de3ec18c51844d06ee28) 发现了当前 memcg per-Zone LRU 的诸多问题. 当前通过 page_cgroup.lru 将 mem_cgroup 链接到自己 per-Zone LRU 的 active_list 和 inactive_list 上, 但是

1.  page_cgroup 的 LRU 与 global LRU 不同步.

2.  page 和 page_cgroup 一对一, 静态分配.

3.  要找到 page_cgroup(pc) 是什么 LRU, 你必须检查 pc->mem_cgroup as - lru = page_cgroup_zoneinfo(pc, nid_of_pc, zid_of_pc);

4.  复杂的 mem_cgroup_per_zone lru_lock 和 zone lru_lock 的嵌套处理, 非常容易导致死锁.

因此为了简化处理

1.  接口层次上实现了 mem_cgroup_add_lru_list(), mem_cgroup_del_lru_list(), mem_cgroup_rotate_lru_list(), mem_cgroup_del_lru(), mem_cgroup_move_lists() 来辅助 mem_cgroup_move_lists() 完成工作.

2.  移除了 mem_cgroup_per_zone 的 lru(即 MEMCG 的 Per-Zone LRU) lock, 全部采用 zone->lru_lock 来处理. 这样 MEMCG 的 Per-Zone LRU lock 就又变成了全局的 zone->lru_lock. 简而言之, 就是这个补丁合入后, 内核没有了 per memcg lru lock, 依旧使用旧的全局 lru lock 来管理全部 memcg lru lists. 这就造成了本来可以自治的 MEMCG, 却要等待其他 MEMCG 释放使用的 lru lock, 在一些场景往往会造成严重的性能问题. 由此而引发了 memcg lru lock 漫长的合入, 参见 [memcg lru lock 血泪史](https://blog.csdn.net/bjchenxu/article/details/112504932).


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/11/21 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg updates (14/Nov/2008)](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=08e552c69c6930d64722de3ec18c51844d06ee28) | 关键 [commit 08e552c69c69 ("memcg: synchronized LRU")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=08e552c69c6930d64722de3ec18c51844d06ee28) 移除了 mem_cgroup_per_zone 的 lru(即 MEMCG 的 Per-Zone LRU) lock, 再次回退到 MEMCG 也使用 zone->lru_lock 来处理的时代. | v1 ☐☑✓ | [LORE v1,0/9](https://lkml.kernel.org/lkml/20081114191246.4f69ff31.kamezawa.hiroyu@jp.fujitsu.com) |


### 9.4.3 归一化的 per-memcg LRU
-------

双 LRU 方案(double-LRU scheme) 方案管理混乱, 运行复杂, 因此 v3.3 的时候, Johannes Weiner 终于看不下去了, 决定对两个 LRU 统一管理, 提出 [memcg naturalization -rc5,00/10](https://lore.kernel.org/lkml/1320787408-22866-1-git-send-email-jweiner@redhat.com). 它使传统的页面回收能够从每个 memcg LRU 列表中查找页面, 从而摆脱了双 LRU 方案和系统中每个页面所需的额外列表头. 参见 [Integrating memory control groups](https://lwn.net/Articles/443241). 归一后, 全局 LRU 不再存在, 所有的页面仅存在于每个组的 LRU 列表中. 未记入特定控制组的页面进入层次结构顶部 root 分组的 LRU 列表. 从本质上讲, 每组的页面回收接管旧的全局回收代码, 即使是禁用控制组的系统也被视为只有一个控制组包含所有正在运行的进程的系统.

实现上:

1.  MEMCG 页面回收上, 引入了 [struct mem_cgroup_zone](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f16015fbf2f6ac45505d6ad21455ff9f6c14473d), 它包含内存 cgroup 和被扫描区域的组合, 作为 [shrink 回收页面时的参数传递](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5660048ccac8735d9bc0a46325a02e6a6518b5b2). 使用 mem_cgroup_reclaim() 和 mem_cgroup_soft_reclaim() 替代了之前 mem_cgroup_hierarchical_reclaim() 来接管 MEMCG 的页面回收.

2.  MEMCG LRU 管理上, 引入了 [lruvec 封装了 LRU list](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6290df545814990ca2663baf6e894669132d5f73), 所有的 LRU 都[被转换为在 per memcg 的 LRU](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b95a2f2d486d0d768a92879c023a03757b9c7e58), 因此[没有理由再保留双 LRU 方案](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=925b7673cce39116ce61e7a06683a4a0dad1e72a). 并且清理了 memcg LRU 操作的接口, 提供了 mem_cgroup_zone_lruvec(), mem_cgroup_lru_add_list(), mem_cgroup_lru_del_list(), mem_cgroup_lru_del(), mem_cgroup_lru_move_lists() 来操作 memcg LRU.

核心算法上:

1.  现在在控件组层次结构中执行深度优先遍历, 试图从每个层次结构中回收一些页. 不存在页面的全局老化问题.

2.  不管其他组中发生了什么, 每个组都有其最老的页面被考虑回收. 当然, 在设定回收目标时, 每个分组的硬性和软性限制都会被考虑.

最终的结果是, 全局回收自然地将痛苦传播到所有控制组, 并在过程中实现每个组的策略. 控制组软限制的实施与该机制相结合, 使软限制的实施更加公平地分布在系统中的所有控制组中.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2011/12/08 | Johannes Weiner <jweiner@redhat.com> | [memcg naturalization -rc5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=6b208e3f6e35aa76d254c395bdcd984b17c6b626) | 参见 [LWN: Integrating memory control groups](https://lwn.net/Articles/443241). 引入 per-memcg lru, 消除重复的 LRU 列表, [全局 LRU 不再存在, page 只存在于 per-memcg LRU list 中](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=925b7673cce39116ce61e7a06683a4a0dad1e72a).<br> 它使传统的页面回收能够从每个 memcg LRU 列表中查找页面, 从而消除了双 LRU 模式 (除了每个 memcg 区域外, 每个全局区域) 和系统中每个页面所需的额外列表头. <br> 该补丁引入了 lruvec 结构. | v5 ☑ [3.3-rc1](https://kernelnewbies.org/Linux_3.3#Memory_management) | [LORE v5,00/10](https://lore.kernel.org/lkml/1320787408-22866-1-git-send-email-jweiner@redhat.com) |


### 9.4.4 per-memcg LRU lock 的血泪史
-------

zone->lru_锁是一个竞争激烈的锁, 因此 2012 年左右 Konstantin Khlebnikov 希望在多核系统上, 通过[在多个 MEMCG 之间拆分它来减少冲突](https://lore.kernel.org/lkml/20120223133728.12988.5432.stgit@zurg).

这引起了 Google 开发人员 Hugh Dickins 的关注, 很快他就发布了 [mm/memcg: per-memcg per-zone lru locking v1, 00/10](https://lore.kernel.org/lkml/alpine.LSU.2.00.1202201518560.23274@eggly.anvils), 通过重新引入 per memcg 的 LRU lock 来解决问题. 但是当时引起的关注比较少, 也缺乏 benchmark 来展示补丁的效果, 所以很快被社区遗忘了. 不过 Google 内部则一直在维护这组补丁, 随他们内核版本升级.

随后 2020 年, 阿里的大佬 Alex Shi 重新设计了 [per memcg lru lock v21,00/19](https://lore.kernel.org/all/1604566549-62481-1-git-send-email-alex.shi@linux.alibaba.com), 并于 [v5.11](https://kernelnewbies.org/Linux_5.11#Memory_management) 版本合入主线. 参见 [memcg lru lock 血泪史](https://blog.csdn.net/bjchenxu/article/details/112504932).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2012/02/16 | Konstantin Khlebnikov <khlebnikov@openvz.org> | [mm: lru_lock splitting](https://lore.kernel.org/all/20120215224221.22050.80605.stgit@zurg) | 20120215224221.22050.80605.stgit@zurg | v1 ☐☑✓ | [LORE v1,0/15](https://lore.kernel.org/all/20120215224221.22050.80605.stgit@zurg)<br>*-*-*-*-*-*-*-* <br>[LORE v2,00/22](https://lore.kernel.org/lkml/20120220171138.22196.65847.stgit@zurg)<br>*-*-*-*-*-*-*-* <br>[LORE v3,00/21](https://lore.kernel.org/lkml/20120223133728.12988.5432.stgit@zurg) |
| 2012/02/20 | Hugh Dickins <hughd@google.com> | [mm/memcg: per-memcg per-zone lru locking](https://lore.kernel.org/patchwork/patch/288055) | per-memcg lru lock | v1 ☐ 3.4 | [PatchWork v1, 00/10](https://lore.kernel.org/lkml/alpine.LSU.2.00.1202201518560.23274@eggly.anvils) |
| 2018/01/31 | Daniel Jordan <daniel.m.jordan@oracle.com> | [lru_lock scalability](https://lore.kernel.org/all/20180131230413.27653-1-daniel.m.jordan@oracle.com) | lru_lock 是内核中争抢非常严重的一把锁, 它保护 per-node lru list. 提高 lru_lock 可伸缩性的一种方法就是使用细粒度锁, 引入一系列锁, 每个锁保护特定批次的 lru 页面. | v1 ☐☑✓ | [LORE v1,0/13](https://lore.kernel.org/all/20180131230413.27653-1-daniel.m.jordan@oracle.com) |
| 2020/12/05 | Alex Shi <alex.shi@linux.alibaba.com> | [per memcg lru lock](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=15b447361794271f4d03c04d82276a841fe06328) | per memcg LRU lock | v21 ☑ [5.11](https://kernelnewbies.org/Linux_5.11#Memory_management) | [LORE v4,0/9](https://lore.kernel.org/lkml/1574166203-151975-1-git-send-email-alex.shi@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[LORE v21,00/19](https://lore.kernel.org/all/1604566549-62481-1-git-send-email-alex.shi@linux.alibaba.com) |


### 9.4.5 MEMCG Reclaim
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2011/05/26 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg async reclaim](https://lore.kernel.org/patchwork/patch/251835) | 实现 MEMCG 的异步回收机制 (Asynchronous memory reclaim)<br>1. 当使用 memc g 时, 应用程序可以看到由 memcg 限制引起的内存回收延迟. 一般来说, 这是不可避免的. 有一类应用程序, 它使用许多干净的文件缓存并执行一些交互式工作.<br>2. 如果内核能够帮助后台回收内存, 那么应用程序的延迟就会在一定程度上被隐藏(这取决于应用程序的睡眠方式). 这组补丁程序添加了控制开关 memory.async_control 启用异步回收. 采用动态计算的方法计算了边缘的大小. 该值被确定为减少应用程序达到限制的机会.<br> 使用了新引入的 WQ_IDLEPRI 类型(使用 SCHED_IDLE 调度策略) 的 kworker(memcg_async_shrinker) 来完整回收的操作. 通过使用 SCHED_IDLE, 系统繁忙的时候异步内存回收只能消耗 0.3% 的 CPU, 但如果 cpu 空闲, 可以使用很多 CPU. | v3 ☐ | [LORE FRC,0/7](https://lkml.org/lkml/2011/5/10/204)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC,v3,0/10](https://lore.kernel.org/lkml/20110526141047.dc828124.kamezawa.hiroyu@jp.fujitsu.com) |
| 2017/05/30 | Johannes Weiner <hannes@cmpxchg.org> | [mm: per-lruvec slab stats](https://lore.kernel.org/patchwork/patch/793422) | Josef 正在研究一种平衡 slab 缓存和 page cache 的新方法. 为此, 他需要 lruvec 级别的 slab 缓存统计信息. 这些补丁通过添加基础设施来实现这一点, 该基础设施允许每个 lruvec 更新和读取通用 VM 统计项, 然后将一些现有 VM 记帐站点 (包括 slab 记帐站点) 切换到这个新的 cgroup 感知 API. | v1 ☑ 4.13-rc1 | [PatchWork 0/6](https://lore.kernel.org/patchwork/patch/793422) |
| 2021/02/17 | Yang Shi <shy828301@gmail.com> | [Make shrinker's nr_deferred memcg aware](https://lore.kernel.org/patchwork/patch/1393744) | 最近, 在一些 vfs 元数据繁重的工作负载上看到了大量的 one-off slab drop, 造成收缩器了大量累积的 nr_deferred 对象.<br> 这个是由多种原因导致的.<br> 这组补丁集使 nr_deferred 变成 per-memcg 来解决问题. | v10 ☑ 5.13-rc1 | [PatchWork v10,00/13](https://patchwork.kernel.org/project/linux-mm/cover/20210217001322.2226796-1-shy828301@gmail.com) |
| 2021/10/19 | Huangzhaoyang <huangzhaoyang@gmail.com> | [mm: skip current when memcg reclaim](https://patchwork.kernel.org/project/linux-mm/patch/1634628576-27448-1-git-send-email-huangzhaoyang@gmail.com) | NA | v2 ☐ | [PatchWork v1](https://patchwork.kernel.org/project/linux-mm/patch/1634278529-16983-1-git-send-email-huangzhaoyang@gmail.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://patchwork.kernel.org/project/linux-mm/patch/1634628576-27448-1-git-send-email-huangzhaoyang@gmail.com) |
| 2022/04/07 | Yosry Ahmed <yosryahmed@google.com> | [memcg: introduce per-memcg proactive reclaim](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=eae3cb2e87ff84547e66211b81301a8f9122840f) | MEMCG 中引入 memory.reclaim 使得用户空间可以通过持续探测 MEMCG 并触发主动回收以回收少量内存. 随着 LRU 不断排序, 这将提供更准确和最新的工作集估计, 并可能提供更具确定性的内存过度使用行为. 内存超分配控制器可以对正在运行的应用程序不断变化的行为提供更主动的响应, 而不是被动响应. 在这种情况下, 用户空间回收器的目的不是完全替代 KSWAPD 或直接回收, 而是主动识别内存节约机会, 回收策略设置的一些冷页, 以释放内存, 用于要求更高的作业或安排新作业.<br> 谷歌数据中心使用了此用户空间主动回收器. | v2 ☑ 5.19-rc1 | [LORE v1,0/1](https://lore.kernel.org/all/20220331084151.2600229-1-yosryahmed@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/4](https://lore.kernel.org/r/20220407224244.1374102-1-yosryahmed@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/4](https://lore.kernel.org/r/20220408045743.1432968-1-yosryahmed@google.com))<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/4](https://lore.kernel.org/r/20220421234426.3494842-1-yosryahmed@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/4](https://lore.kernel.org/r/20220425190040.2475377-1-yosryahmed@google.com) |




## 9.5 MEMCG DRITY Page
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2011/04/17 | Greg Thelen <gthelen@google.com> | [memcg: per cgroup dirty page limiting](https://lore.kernel.org/patchwork/patch/263368) | per-memcg 的脏页带宽限制 | v9 ☐ | [PatchWork v9](https://lore.kernel.org/patchwork/patch/263368) |
| 2015/12/30 | Tejun Heo <tj@kernel.org> | [memcg: add per cgroup dirty page accounting](https://lore.kernel.org/patchwork/patch/558382) | madvise 支持页面延迟回收 (MADV_FREE) 的早期尝试  | v5 ☑ [4.2-rc1](https://kernelnewbies.org/Linux_4.2#Memory_management) | [PatchWork v1](https://lore.kernel.org/patchwork/patch/558382), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c4843a7593a9df3ff5b1806084cefdfa81dd7c79) |


## 9.6 MEMCG Swapin
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/07/04 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg: shmem swap cache](https://lore.kernel.org/patchwork/patch/121475) | memcg swapin 的延迟统计, 对 memcg 进行了修改, 使其在交换时直接对交换页统计, 而不是在出错时统计, 这可能要晚得多, 或者根本不会发生. | v1 ☑ 2.6.26-rc8-mm1 | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/121270)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/patch/121475)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/patch/https://lore.kernel.org/patchwork/patch/121475), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d13d144309d2e5a3e6ad978b16c1d0226ddc9231) |
| 2008/07/14 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg: handle tmpfs' swapcache](https://lore.kernel.org/patchwork/patch/122556) | memcg swapin 的延迟统计, 对 memcg 进行了修改, 使其在交换时直接对交换页统计, 而不是在出错时统计, 这可能要晚得多, 或者根本不会发生. | v1 ☐ | [PatchWork v2](https://lore.kernel.org/patchwork/patch/122556)) |
| 2008/11/14 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg : add swap controller](https://lore.kernel.org/patchwork/patch/134928) | 实现 CONFIG_MEMCG_SWAP | v1 ☑ 2.6.29-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/patch/134928)) |
| 2008/12/01 | KOSAKI Motohiro <kosaki.motohiro@jp.fujitsu.com> | [memcg: split-lru feature for memcg take2](https://lkml.org/lkml/2008/12/1/99) | NA | v2 ☑ [2.6.29-rc1](https://kernelnewbies.org/Linux_2_6_29#Memory_controller_swap_management_and_other_improvements) | [LKML](https://lkml.org/lkml/2008/12/1/99), [PatchWork](https://lore.kernel.org/patchwork/patch/136809) |
| 2008/12/02 | KOSAKI Motohiro <kosaki.motohiro@jp.fujitsu.com> | [memcg: swappiness](https://lkml.org/lkml/2008/12/2/21) | 引入 per-memcg 的 swappiness, 可以用来对 per-memcg 进行精确控制. | v2 ☑ [2.6.29-rc1](https://kernelnewbies.org/Linux_2_6_29#Memory_controller_swap_management_and_other_improvements) | [LKML](https://lkml.org/lkml/2008/12/2/21), [PatchWork](https://lore.kernel.org/patchwork/patch/136809) |
| 2009/06/02 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [memcg fix swap accounting](https://lore.kernel.org/patchwork/patch/158520) | NA | v1 ☑ 2.6.29-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/patch/157997)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://lore.kernel.org/patchwork/patch/158520)) |
| 2015/12/17 | Vladimir Davydov <vdavydov@virtuozzo.com> | [Add swap accounting to cgroup2](https://lore.kernel.org/patchwork/patch/628754) | NA | v2 ☑ 2.6.29-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/patch/628754)) |
| 2020/08/20 | Johannes Weiner <hannes@cmpxchg.org> | [mm: memcontrol: charge swapin pages on instantiation](https://lore.kernel.org/patchwork/patch/1239175) | memcg swapin 的延迟统计, 对 memcg 进行了修改, 使其在交换时直接对交换页统计, 而不是在出错时统计, 这可能要晚得多, 或者根本不会发生. | v2 ☑ [5.8-rc1](https://kernelnewbies.org/Linux_5.8#Memory_management) | [PatchWork v1](https://lore.kernel.org/patchwork/patch/1227833), [PatchWork v2](https://lore.kernel.org/patchwork/patch/1239175) |


## 9.7 MEMCG THP
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/05/05 | CGEL <cgel.zte@gmail.com> | [mm/memcg: support control THP behaviour in cgroup](https://patchwork.kernel.org/project/linux-mm/patch/20220505033814.103256-1-xu.xin16@zte.com.cn) | 使用 THP 可以提高内存的性能, 但会增加内存占用. 应用程序可能会使用 madvise 来减少占用空间, 但并非所有应用程序都支持使用 madvise, 而且重新编码所有应用程序需要花费大量成本. 但是当前容器越来越流行于管理一组任务. 因此, 添加对 cgroup 的支持以控制 THP 行为将提供很大的便利, 管理员可能只对重要容器启用 THP, 而对其他容器禁用它. 这样我们就可以享受 THP 的高性能, 同时最大限度地减少内存占用, 而无需重新编写任何应用程序. | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220505033814.103256-1-xu.xin16@zte.com.cn)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20220506031804.437642-1-yang.yang29@zte.com.cn) |
| 2022/06/22 | Baolin Wang <baolin.wang@linux.alibaba.com> | [Add PUD and kernel PTE level pagetable account](https://patchwork.kernel.org/project/linux-mm/cover/cover.1655887440.git.baolin.wang@linux.alibaba.com/) | 652684 | v2 ☐☑ | [LORE v2,0/3](https://lore.kernel.org/r/cover.1655887440.git.baolin.wang@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/3](https://lore.kernel.org/r/cover.1656586863.git.baolin.wang@linux.alibaba.com) |

## 9.8 Memory vmpressure
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2022/06/23 | Yosry Ahmed <yosryahmed@google.com> | [mm: vmpressure: don't count userspace-induced reclaim as memory pressure](https://patchwork.kernel.org/project/linux-mm/patch/20220623000530.1194226-1-yosryahmed@google.com/) | 652966 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220623000530.1194226-1-yosryahmed@google.com) |


## 9.9 Dying MEMCG
-------

第 17 届中国 Linux 内核开发者大会 (CLK-2022) ["内存管理与异构计算" 分论坛](https://live.csdn.net/v/247710) 的第一个议题 "Iprovement of Dying Memory Cgroup" 来自 ByteDance 的开发者宋牧春就对 Dying MEMCG 问题进行了深入的讨论.

| 时间 | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:---:|:----:|:---:|:----:|:---------:|:----:|
| 2020/06/23 | Roman Gushchin <guro@fb.com> | [The new cgroup slab memory controller](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=fbc1ac9d09d70859eee24131d667e01e3986e368) | 引入了 obj_cgroup, 接管了 slab memory 的 charge/uncharge 操作. | v7 ☑✓ 5.9-rc1 | [LORE v7,0/19](https://lore.kernel.org/all/20200623174037.3951353-1-guro@fb.com) |
| 2021/03/20 | Muchun Song <songmuchun@bytedance.com> | [Use obj_cgroup APIs to charge kmem pages](hhttps://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=bd290e1e75d8a8b2d87031b63db56) | 使用 obj_cgroup 接管 kmem 的 charge/uncharge 操作. | v5 ☑✓ 5.13-rc1 | [LORE v5,0/7](https://lore.kernel.org/all/20210319163821.20704-1-songmuchun@bytedance.com) |
| 2022/04/21 | Waiman Long <longman@redhat.com> | [mm/memcg: Free percpu stats memory of dying memcg's](https://lore.kernel.org/all/20220421145845.1044652-1-longman@redhat.com) | TODO | v1 ☐☑✓ | [LORE](https://lore.kernel.org/all/20220421145845.1044652-1-longman@redhat.com) |
| 2022/06/21 | Muchun Song <songmuchun@bytedance.com> | [Use obj_cgroup APIs to charge the LRU pages](https://lore.kernel.org/all/20220621125658.64935-1-songmuchun@bytedance.com) | 使用 obj_cgroup 接管 LRU 页面的 charge/uncharge. | v6 ☐☑✓ | [LORE v6,0/11](https://lore.kernel.org/all/20220621125658.64935-1-songmuchun@bytedance.com) |



# 10 内存热插拔支持
-------




内存热插拔, 也许对于第一次听说的人来说, 觉得不可思议: 一个系统的核心组件, 为何要支持热插拔? 用处有以下几点.



> 1\. 大规模集群中, 动态的物理容量增减, 可以实现更好地支持资源集约和均衡.
> 2\. 大规模集群中, 物理内存出错的机会大大增多, 内存热插拔技术对提高高可用性至关重要.
> 3\. 在虚拟化环境中, 客户机 (Guest OS) 间的高效内存使用也对热插拔技术提出要求



当然, 作为一个核心组件, 内存的热插拔对从系统固件, 到软件 (操作系统) 的要求, 跟普通的外设热插拔的要求, 不可同日而语. 这也是为什么 Linux 内核对内存热插拔的完全支持一直到近两年才基本完成.



总的来说, 内存热插拔分为两个阶段, 即 ** 物理热插拔阶段 ** 和 ** 逻辑热插拔阶段:**

> ** 物理热插拔阶段:** 这一阶段是内存条插入 / 拔出主板的过程. 这一过程必须要涉及到固件的支持(如 ACPI 的支持), 以及内核的相关支持, 如为新插入的内存分配管理元数据进行管理. 我们可以把这一阶段分别称为 hot-add / hot-remove.
> ** 逻辑热插拔阶段:** 这一阶段是从使用者视角, 启用 / 关闭这部分内存. 这部分的主要从内存分配器方面做的一些准备工作. 我们可以把这一阶段分别称为 online / offline.



逻辑上来讲, 内存插和拔是一个互为逆操作, 内核应该做的事是对称的, 但是, 拔的过程需要关注的技术难点却比插的过程多, 因为, 从无到有容易, 从有到无麻烦: 在使用的内存页应该被妥善安置, 就如同安置拆迁户一样, 这是一件棘手的事情. 所以内核对 hot-remove 的完全支持一直推迟到 2013 年.



## 10.1 内存热插入支持
-------

**2.6.15(2006 年 1 月发布)**


这提供了最基本的内存热插入支持(包括物理 / 逻辑阶段). 注意, ** 此时热拔除还不支持.**



## 10.2 初步的内存逻辑热拔除支持
-------

**2.6.24(2008 年 1 月发布)**




此版本提供了 ** 部分的逻辑热拔除阶段的支持, 即 offline.** Offline 时, 内核会把相关的部分内存隔离开来, 使得该部分内存不可被其他任何人使用, 然后再把此部分内存页, 用前面章节说过的内存页迁移功能转移到别的内存上. 之所以说 ** 部分支持 **, 是因为该工作只提供了一个 offline 的功能. 但是, 不是所有的内存页都可以迁移的. 考虑 "迁移" 二字的含义, 这意味着 ** 物理内存地址 ** 会变化, 而内存热拔除应该是对使用者透明的, 这意味着, 用户见到的 ** 虚拟内存地址 ** 不能变化, 所以这中间必须存在一种机制, 可以修改这种因迁移而引起的映射变化. 所有通过页表访问的内存页就没问题了, 只要修改页表映射即可; 但是, 内核自己用的内存, 由于是绕过寻常的逐级页表机制, 采用直接映射(提高了效率), 内核内存页的虚拟地址会随着物理内存地址变动, 因此, 这部分内存页是无法轻易迁移的. 所以说, 此版本的逻辑热拔除功能只是部分完成.



注意, 此版本中, ** 物理热拔除是还完全未实现. 一句话, 此版本的热拔除功能还不能用.**





## 10.3 完善的内存逻辑热拔除支持
-------

**3.8(2013 年 2 月发布)**




针对 9.2 中的问题, 此版本引入了一个解决方案. 9.2 中的核心问题在于 ** 不可迁移的页会导致内存无法被拔除.** 解决问题的思路是使可能被热拔除的内存不包含这种不可迁移的页. 这种信息应该在内存初始化 / 内存插入时被传达, 所以, 此版本中, 引入一个 movable\_node 的概念. 在此概念中, 一个被 movable\_node 节点的所有内存, 在初始化 / 插入后, 内核确保它们之上不会被分配有不可迁移的页, 所以当热拔除需求到来时, 上面的内存页都可以被迁移, 从而使内存可以被拔除.





## 10.4 物理热拔除的支持
-------


**3.9(2013 年 4 月支持)**

此版本支持了 ** 物理热拔除,** 这包括对内存管理元数据的删除, 跟固件(如 ACPI) 相关功能的实现等比较底层琐碎的细节, 不详谈.



在完成这一步之后, 内核已经可以提供基本的内存热插拔支持. 值得一提的是, 内存热插拔的工作, Fujitsu 中国这边的内核开放者贡献了很多 patch. 谢谢他们!


## 10.5 improving in-kernel auto-online support
-------

*   虚拟机内存热缩优化


新上线的页面通常会被放置在 freelist 的前面, 这意味着它们很快会在下一次内存分配时就被使用, 这将造成添加的内存通常立即无法删除.

使用 virtio-mem 和 dimm 可以经常观察到这个现象: 当通过 `add_memory*()` 添加单独的内存块并立即联机使用它们时, 内核通常会把下一个块的元数据 (特别是 memmap 的数据) 被放置到刚刚添加并上线的内存块上. 这是一个不可移动的内存分配链, 如果最后一个内存块不能脱机并 remove(), 那么所有依赖的内存块也会脱机.

其实新页面通常是冷的, 尝试分配其他页面 (可能是热的) 本身是一件非常自然的事情. 因此一个非常简单有效的尝试是: 新上线的内存其空闲页面不要放置到 freelist 的前面, 尝试放置到 freelist 的最后即可.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/09/16 | David Hildenbrand <david@redhat.com> | [mm: place pages to the freelist tail when onling and undoing isolation](https://patchwork.kernel.org/project/linux-mm/cover/20200916183411.64756-1-david@redhat.com) | 新上线的内存其空闲页面或者隔离页放置到 freelist 的最后 | RFC,v1 ☑ 5.10-rc1 | [PatchWork RFC,0/4](https://patchwork.kernel.org/project/linux-mm/cover/20200916183411.64756-1-david@redhat.com) |

*   "auto-movable" online policy


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/06/02 | David Hildenbrand <david@redhat.com> | [virtio-mem: prioritize unplug from ZONE_MOVABLE](https://patchwork.kernel.org/project/linux-mm/cover/20210602185720.31821-1-david@redhat.com) | NA | v1  ☑ 5.14-rc1 | [PatchWork v1,0/7](https://patchwork.kernel.org/project/linux-mm/cover/20210602185720.31821-1-david@redhat.com) |
| 2021/08/06 | David Hildenbrand <david@redhat.com> | [mm/memory_hotplug:"auto-movable"online policy and memory groups](https://patchwork.kernel.org/project/linux-mm/cover/20210806124715.17090-1-david@redhat.com) | NA | v3 ☑ 5.15-rc1 | [PatchWork v3,0/9](https://patchwork.kernel.org/project/linux-mm/cover/20210806124715.17090-1-david@redhat.com) |

# 11 超然内存 (Transcendent Memory) 支持
-------


## 11.1 tmem 简介
-------

[超然内存 (Transcendent Memory)](https://lwn.net/Articles/338098), 对很多第一次听见这个概念的人来说, 是如此的奇怪和陌生. 这个概念是在 Linux 内核开发者社区中首次被提出的. 超然内存(后文一律简称为[**tmem**](https://lore.kernel.org/patchwork/patch/161314)) 之于普通内存, 不同之处在于以下几点:


1.  tmem 的大小对内核来说是未知的, 它甚至是可变的; 与之相比, 普通内存在系统初始化就可被内核探测并枚举, 并且大小是固定的(不考虑热插拔情形).

2.  tmem 可以是稳定的, 也可以是不稳定的, 对于后者, 这意味着, 放入其中的数据, 在之后的访问中可能发现不见了; 与之相比, 普通内存在系统加电状态下一直是稳定的, 你不用担心写入的数据之后访问不到(不考虑内存硬件故障问题).

3.  基于以上两个原因, tmem 无法被内核直接访问, 必须通过定义良好的 API 来访问 tmem 中的内存; 与之相比, 普通内存可以通过内存地址被内核直接访问.


初看之下, [tmem](https://lore.kernel.org/patchwork/patch/163284) 这三点奇异的特性, 似乎增加了不必要的复杂性, 尤其是第二条, 更是诡异无用处.

计算机界有句名言: 计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决. tmem 增加的这些间接特性正是为了解决某些问题.


考虑虚拟化环境下, 虚拟机管理器 (hypervisor) 需要管理维护各个虚拟客户机(Guest OS) 的内存使用. 在常见的使用环境中, 我们新建一台虚拟机时, 总要事先配置其可用的内存大小. 然而这并不十分明智, 我们很难事先确切知道每台虚拟机的内存需求情况, 这难免造成有些虚拟机内存富余, 而有些则捉襟见肘. 如果能平衡这些资源就好了. 事实上, 存在这样的技术, hypervisor 维护一个内存池子, 其中存放的就是这些所谓的 tmem, 它能动态地平衡各个虚拟机的盈余内存, 更高效地使用. 这是 tmem 概念提出的最初来由.



再考虑另一种情形, 当一台机器内存使用紧张时, 它会把一些精心选择的内存页面写到交换外设中以腾出空间. 然而外设与内存存在着显著的读写速度差异, 这对性能是一个不利的影响. 如果在一个超高速网络相连的集群中, 网络访问速度比外设访问速度快, 可不可以有别的想法? tmem 又可以扮演一个重要的中间角色, 它可以把集群中所有节点的内存管理在一个池子里, 从而动态平衡各节点的内存使用, 当某一节点内存紧张时, 写到交换设备页其实是被写到另一节点的内存里...



还有一种情形, 旨在提高内存使用效率.. 比如大量内容相同的页面 (全 0 页面) 在池子里可以只保留一份; 或者, tmem 可以考虑对这些页面进行压缩, 从而增加有效内存的使用.



Linux 内核 从 3.X 系列开始陆续加入 tmem 相关的基础设施支持, 并逐步加入了关于内存压缩的功能. 进一步讨论内核中的实现前, 需要对这一问题再进一步细化, 以方便讨论细节.



前文说了内核需要通过 API 访问 tmem, 那么进一步, 可以细化为两个问题.

1.  内核的哪些部分内存可以被 tmem 管理呢? 有两大类, 即前面提到过的文件缓存页和匿名页.

2.  tmem 如何管理其池子中的内存. 针对前述三种情形, 有不同的策略. Linux 现在的主要解决方案是针对内存压缩, 提高内存使用效率.



针对这两个问题, 可以把内核对于 tmem 的支持分别分为 ** 前端 ** 和 ** 后端 **. 前端是内核与 tmem 通讯的接口; 而后端则实现 tmem 的管理策略.


## 11.1 tmem 前端
-------

### 11.1.1 前端接口之 CLEANCACHE
-------

**3.0(2011 年 7 月发布)**


前面章节提到过, 内核会利用空闲内存缓存后备外设中的内容, 以期在近期将要使用时不用从缓慢的外设读取. 这些内容叫文件缓存页. 它的特点就是只要是页是干净的(没有被写过), 那么在系统需要内存时, 随时可以直接丢弃这些页面以腾出空间(因为它随时可以从后备文件系统中读取).



然而, 下次内核需要这些文件缓存页时, 又得从外设读取. 这时, tmem 可以起作用了. 内核丢弃时, 假如这些页面被 tmem 接收并管理起来, 等内核需要的时候, tmem 把它归还, 这样就省去了读写磁盘的操作, 提高了性能. 3.0 引入的 CLEANCACHE, 作为 tmem 前端接口, hook 进了内核丢弃这些干净文件页的地方, 把这些页面截获了, 放进 tmem 后端管理(后方讲). 从内核角度看, 这个 CLEANCACHE 就像一个魔盒的入口, 它丢弃的页面被吸入这个魔盒, 在它需要时, 内核尝试从这个魔盒中找, 如果找得到, 搞定; 否则, 它再去外设读取. 至于为何被吸入魔盒的页面为什么会找不到, 这跟前文说过的 tmem 的特性有关. 后文讲 tmem 后端时再说.



### 11.1.2 前端接口之 FRONTSWAP
-------

**3.5(2012 年 7 月发布)**


除了文件缓存页, 另一大类内存页面就是匿名页. 在系统内存紧张时, 内核必须要把这些页面写出到外设的交换设备或交换分区中, 而不能简单丢弃(因为这些页面没有后备文件系统). 同样, 涉及到读写外设, 又有性能考量了, 此时又是 tmem 起作用的时候了.



同样, FRONTSWAP 这个前端接口, 正如其名字一样, 在 内核 swap 路径的前面, 截获了这些页面, 放入 tmem 后端管理. 如果后端还有空闲资源的话, 这些页面被接收, 在内核需要这些页面时, 再把它们吐出来; 如果后端没有空闲资源了, 那么内核还是会把这些页面按原来的走 swap 路径写到交换设备中.



## 11.2 后端
-------

[In-kernel memory compression](https://lwn.net/Articles/545244)

### 11.2.1 后端之 ZCACHE
-------


**(没能进入内核主线)**

讲完了两个前端接口, 接下来说后端的管理策略. 对应于 CLEANCACHE, 一开始是有一个专门的后端叫 [zcache](https://lwn.net/Articles/397574), ** 不过最后被删除了.** 它的做法就是把这些被内核逐出的文件缓存页压缩, 并存放在内存中. 所以, zcache 相当于把内存页从内存中的一个地方移到另一个地方, 这乍一看, 感觉很奇怪, 但这正是 tmem 的灵活性所在. 它允许后端有不同的管理策略, 比如在这个情况下, 它把内存页压缩后仍然放在内存中, 这提高了内存的使用. 当然, 毕竟 zcache 会占用一部分物理内存, 导致可用的内存减小. 因此, 这需要有一个权衡. 高效 (压缩比, 压缩时间) 的压缩算法的使用, 从而使更多的文件页待在内存中, 使得其带来的避免磁盘读写的优势大于减少的这部分内存的代价. 不过, 也因为如此, 它的实现过于复杂, ** 以至最终没能进入内核主线.** 开发者在开始重新实现一个新的替代品, 不过截止至 4.2 , 还没有看到成果.



### 11.2.2 后端之 ZRAM
-------

**3.14(2014 年 3 月发布)**



FRONTSWAP 对应的一个后端叫 ZRAM. 前身为 compcache, 它早在 2.6.33(2010 年 2 月发布) 时就已经进入内核的 staging 分支经历了 4 年的 staging 后, 于内核 3.14 版本进入 mainline(2014 年 3 月), 目前是谷歌 Android 系统 (自 4.4 版本) 和 Chrome OS(自 2013 年)使用的默认方案.

> Staging 分支是在内核源码中的一个子目录, 它是一个独立的分支, 主要维护着独立的 driver 或文件系统, 这些代码未来可能也可能不进入主线.


ZRAM 是一个在内存中的块设备(块设备相对于字符设备而言, 信息存放于固定大小的块中, 支持随机访问, 磁盘就是典型的块设备, 更多将在块层子系统中讲), 因此, 内核可以复用已有的 swap 设备设施, 把这个块设备格式化为 swap 设备. 因此, 被交换出去的页面, 将通过 FRONTSWAP 前端进入到 ZRAM 这个伪 swap 设备中, 并被压缩存放! 当然, 这个 ZRAM 空间有限, 因此, 页面可能不被 ZRAM 接受. 如果这种情形发生, 内核就回退到用真正的磁盘交换设备.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:---:|:----------:|:----:|
| 2022/09/30 | Brian Geffon <bgeffon@google.com> | [zram: Always expose rw_page](https://patchwork.kernel.org/project/linux-mm/patch/20220930195215.2360317-1-bgeffon@google.com/) | 682376 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220930195215.2360317-1-bgeffon@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20221003144832.2906610-1-bgeffon@google.com) |


### 11.2.3 后端之 ZSWAP
-------

**3.11(2013 年 9 月发布)**


FRONTSWAP 对应的另一个后端叫 [ZSWAP](https://lwn.net/Articles/537422). ZSWAP 的做法其实也是尝试把内核交换出去的页面压缩存放到一个内存池子中. 当然, ZSWAP 空间也是有限的. 但同 ZRAM 不同的是, ZSWAP 会智能地把其中一些它认为近期不会使用的页面解压缩, 写回到真正的磁盘外设中. 因此, 大部分情况下, 它能避免磁盘写操作, 这比 ZRAM 不知高明到哪去了.

### 11.2.3.1 ZSWAP
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:---:|:----------:|:----:|
| 2021/08/19 | Johannes Weiner <hannes@cmpxchg.org> | [mm: Kconfig: simplify zswap configuration](https://lore.kernel.org/patchwork/patch/1479229) | 重构 CONFIG_ZSWAP. | v1 ☐ | [PatchWork](https://lore.kernel.org/patchwork/patch/1479229) |
| 2022/04/27 | Johannes Weiner <hannes@cmpxchg.org> | [zswap: cgroup accounting & control](https://patchwork.kernel.org/project/linux-mm/cover/20220427160016.144237-1-hannes@cmpxchg.org/) | 636247 | v1 ☐☑ | [LORE v1,0/5](https://lore.kernel.org/r/20220427160016.144237-1-hannes@cmpxchg.org)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/6](https://lore.kernel.org/all/20220510152847.230957-1-hannes@cmpxchg.org) |
| 2022/08/25 | Liu Shixin <liushixin2@huawei.com> | [Delay the initializaton of zswap](https://patchwork.kernel.org/project/linux-mm/cover/20220825142037.3214152-1-liushixin2@huawei.com/) | 671093 | v1 ☐☑ | [LORE v1,0/3](https://lore.kernel.org/r/20220825142037.3214152-1-liushixin2@huawei.com) |
| 2022/10/05 | Sergey Senozhatsky <senozhatsky@chromium.org> | [zram: Support multiple compression streams](https://patchwork.kernel.org/project/linux-mm/cover/20221005024014.22914-1-senozhatsky@chromium.org/) | [Google Engineer Experimenting With ZRAM Handling For Multiple Compression Streams](https://www.phoronix.com/news/ZRAM-Multiple-Compression) 和 [Linux 6.2 Lands Support For Multiple Compression Streams With ZRAM](https://www.phoronix.com/news/Linux-6.2-ZRAM-Multi-Compress) | v1 ☐☑ | [LORE v1,0/8](https://lore.kernel.org/r/20221005024014.22914-1-senozhatsky@chromium.org)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20221005173713.1308832-1-yosryahmed@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/13](https://lore.kernel.org/r/20221109115047.2921851-1-senozhatsky@chromium.org) |


### 11.2.3.2 ZTREE
-------

| 时间   | 作者  | 特性 | 描述  |  是否合入主线  | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2022/02/28 | Ananda <a.badmaev@clicknet.pro> | [[PATCH/RESEND] mm: add ztree - new allocator for use via zpool API](https://patchwork.kernel.org/project/linux-mm/patch/20220228110546.151513-1-a.badmaev@clicknet.pro/) | 用于压缩页面的专用分配器 Ztree. 在大多数情况下, Ztree 提供了快速写入、有限的最坏情况操作时间和良好的压缩比.<br> 每个 Ztree 块存储整数个压缩对象. 这些块由几个物理页面 (从 1 到 8) 组成, 并使用红黑树用于高效的区块组织.<br> 从 0 到 PAGE_SIZE 的范围被划分为与树的数量相对应的区间数, 每棵树只操作其区间中的大小对象.<br>1. 块树彼此隔离, 这使得同时对来自不同树的多个对象执行操作成为可能. 块可以密集地排列各种大小的对象, 从而降低内部碎片.<br>2. 此外, 这个分配器试图填充不完整的块, 而不是添加新的块, 因此在许多情况下, 它提供的压缩比大大高于 z3fold 和 zbud.<br>3. 除了更大的灵活性, Ztree 在最糟糕的执行时间方面明显优于其他 ZPOOL 后端, 从而允许更好的响应. | v1 ☐☑ |[LORE v1,0/1](https://lore.kernel.org/r/20220228110546.151513-1-a.badmaev@clicknet.pro)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20220301092503.44444-1-a.badmaev@clicknet.pro)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/1](https://lore.kernel.org/r/20220307142724.14519-1-a.badmaev@clicknet.pro)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/1](https://lore.kernel.org/r/20220311085807.27038-1-a.badmaev@clicknet.pro) |


### 11.2.3.3 ZBUD
-------

| 时间   | 作者  | 特性 | 描述  |  是否合入主线  | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2013/09/11 | Krzysztof Kozlowski <k.kozlowski@samsung.com> | [mm: migrate zbud pages](https://lore.kernel.org/all/1378889944-23192-1-git-send-email-k.kozlowski@samsung.com) | 1378889944-23192-1-git-send-email-k.kozlowski@samsung.com | v2 ☐☑✓ | [LORE v2,0/5](https://lore.kernel.org/all/1378889944-23192-1-git-send-email-k.kozlowski@samsung.com) |
| 2015/09/22 | Vitaly Wool <vitalywool@gmail.com> | [zbud: allow up to PAGE_SIZE allocations](https://lore.kernel.org/all/20150922141733.d7d97f59f207d0655c3b881d@gmail.com) | 20150922141733.d7d97f59f207d0655c3b881d@gmail.com | v2 ☐☑✓ | [LORE](https://lore.kernel.org/all/20150922141733.d7d97f59f207d0655c3b881d@gmail.com) |

### 11.2.3.4 ZSMALLOC
-------

| 时间   | 作者  | 特性 | 描述  |  是否合入主线  | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2022/10/24 | Sergey Senozhatsky <senozhatsky@chromium.org> | [zsmalloc/zram: configurable zspage size](https://patchwork.kernel.org/project/linux-mm/cover/20221024161213.3221725-1-senozhatsky@chromium.org) | 688281 | v1 ☐☑ | [LORE v1,0/6](https://lore.kernel.org/r/20221024161213.3221725-1-senozhatsky@chromium.org)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/9](https://lore.kernel.org/r/20221026112933.4122957-1-senozhatsky@chromium.org)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/9](https://lore.kernel.org/r/20221027042651.234524-1-senozhatsky@chromium.org) |
| 2022/10/27 | Nhat Pham <nphamcs@gmail.com> | [[v2,5/5] zsmalloc: Implement writeback mechanism for zsmalloc](https://patchwork.kernel.org/project/linux-mm/patch/20221027182736.513530-1-nphamcs@gmail.com/) | 689539 | v2 ☐☑ | [LORE v2,0/5](https://lore.kernel.org/r/20221027182736.513530-1-nphamcs@gmail.com) |
| 2022/10/26 | Nhat Pham <nphamcs@gmail.com> | [Implement writeback for zsmalloc](https://patchwork.kernel.org/project/linux-mm/cover/20221026200613.1031261-1-nphamcs@gmail.com/) | 689169 | v1 ☐☑ | [LORE v1,0/5](https://lore.kernel.org/r/20221026200613.1031261-1-nphamcs@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/5](https://lore.kernel.org/all/20221108193207.3297327-1-nphamcs@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/5](https://lore.kernel.org/r/20221117163839.230900-1-nphamcs@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/6](https://lore.kernel.org/r/20221118182407.82548-1-nphamcs@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/6](https://lore.kernel.org/r/20221119001536.2086599-1-nphamcs@gmail.com)<br>*-*-*-*-*-*-*-* <br>[LORE v7,0/6](https://lore.kernel.org/r/20221128191616.1261026-1-nphamcs@gmail.com) |
| 2022/11/23 | Nhat Pham <nphamcs@gmail.com> | [Implement writeback for zsmalloc (fix)](https://patchwork.kernel.org/project/linux-mm/cover/20221123191703.2902079-1-nphamcs@gmail.com/) | 698636 | v6 ☐☑ | [LORE v6,0/6](https://lore.kernel.org/r/20221123191703.2902079-1-nphamcs@gmail.com) |
| 2023/02/23 | Sergey Senozhatsky <senozhatsky@chromium.org> | [zsmalloc: fine-grained fullness and new compaction algorithm](https://patchwork.kernel.org/project/linux-mm/cover/20230223030451.543162-1-senozhatsky@chromium.org/) | 724223 | v1 ☐☑ | [LORE v1,0/6](https://lore.kernel.org/r/20230223030451.543162-1-senozhatsky@chromium.org)<br>*-*-*-*-*-*-*-* <br>[2023/03/03 LORE v1,0/4](https://lore.kernel.org/all/20230303073130.1950714-1-senozhatsky@chromium.org)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/4](https://lore.kernel.org/r/20230304034835.2082479-1-senozhatsky@chromium.org) |



## 11.3 一些细节
-------


这一章基本说完了, 但牵涉到后端, 其实还有一些细节可以谈, 比如对于压缩的效率的考量, 会影响到后端实现的选择, 比如不同的内存页面的压缩效果不同 (全 0 页和某种压缩文件占据的内存页的压缩效果显然差距很大) 对压缩算法的选择; 压缩后页面的存放策略也很重要, 因为以上后端都存在特殊情况要把页面解压缩写回到磁盘外设, 写回页面的选择与页面的存放策略关系很大. 但从用户角度讲, 以上内容足以, 就不多写了.



关于 tmem, lwn 上的两篇文章值得关注技术细节的人一读:

[Transcendent memory in a nutshell [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/454795)

[In-kernel memory compression [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/545244)



# 12 分级内存支持
-------

## 12.1 分级内存
-------

### 12.1.1 不同介质的内存的支持
-------

#### 12.1.1.1 非易失性内存 (NVDIMM, Non-Volatile DIMM) 支持
-------

[Linux Kernel 中 AEP 的现状和发展](https://kernel.taobao.org/2019/05/NVDIMM-in-Linux-Kernel)


计算机的存储层级是一个金字塔体系, 从塔尖到塔基, 访问速度递减, 而存储容量递增. 从访问速度考量, 内存 (DRAM) 与磁盘 (HHD) 之间, 存在着[显著的差异(可达到 10^5 级别)](https://www.directionsmag.com/article/3794). 因此, 基于内存的缓存技术一直都是系统软件或数据库软件的重中之重. 即使近些年出现的新兴的最快的基于 PCIe 总线的 SSD, 这中间依然存在着鸿沟.


![](https://pic4.zhimg.com/50/0c0850cde43c84764e65bc24942bc6d3_hd.jpg)


另一方面, 非可易失性内存也并不是新鲜产物. 然而实质要么是一块 DRAM, 后端加上一块 NAND FLASH 闪存, 以及一个超级电容, 以在系统断电时的提供保护; 要么就是一块简单的 NAND FLASH, 提供类似 SSD 一样的存储特性. 所有这些, 从访问速度上看, 都谈不上真正的内存, 并且, NAND FLASH 的物理特性, 使其免不了磨损(wear out); 并且在长时间使用后, 存在写性能下降的问题.



2015 年算得上闪存技术革命年. 3D NAND FLASH 技术的创新, 以及 Intel 在酝酿的完全不同于 NAND 闪存技术的 3D XPoint 内存, 都将预示着填充上图这个性能鸿沟的时刻的临近. 它们不仅能提供更在的容量(TB 级别), 更快的访问速度(3D XPoint 按 Intel 说法能提供 `~1000` 倍快于传统的 NAND FLASH, 5 - 8 倍慢于 DRAM 的访问速度), 更持久的寿命.



相应的, Linux 内核也在进行相应的功能支持.


#### 12.1.1.2 NVDIMM 支持框架
-------

** libnvdimm 4.2(2015 年 8 月 30 日发布)**

2015 年 4 月发布的 [ACPI 6.0 规范](https://uefi.org/sites/default/files/resources/ACPI/_6.0.pdf](https://link.zhihu.com/?target=http%3A//www.uefi.org/sites/default/files/resources/ACPI_6.0.pdf), 定义了 NVDIMM Firmware Interface Table (NFIT), 详细地规定了 NVDIMM 的访问模式, 接口数据规范等细节. 在 Linux 4.2 中, 内核开始支持一个叫 libnvdimm 的子系统, 它实现了 NFIT 的语义, 提供了对 NVDIMM 两种基本访问模式的支持, 一种即内核所称之的 PMEM 模式, 即把 NVDIMM 设备当作持久性的内存来访问; 另一种则提供了块设备模式的访问. 开始奠定 Linux 内核对这一新兴技术的支持.

#### 12.1.1.3 DAX
-------

**4.0(2015 年 4 月发布)**

与这一技术相关的还有另外一个特性值得一提, 那就是 DAX(Direct Access, 直接访问, X 无实义, 只是为了酷).



传统的基于磁盘的文件系统, 在被访问时, 内核总会把页面通过前面所提的文件缓存页 (page cache) 的缓存机制, 把文件系统页从磁盘中预先加载到内存中, 以提速访问. 然后, 对于新兴的 NVDIMM 设备, 基于它的非易失特性, 内核应该能直接访问基于此设备之上的文件系统的内容, 它使得这一拷贝到内存的操作变得不必要. 4.0 开始引入的 DAX 就是提供这一支持. 截至 4.3, 内核中已经有 XFS, EXT2, EXT4 这几个文件系统实现这一特性.


#### 12.1.1.4 Compute Express Link(CXL)
-------

[LWN: LSFMM-2022/CXL 1: Management and tiering](https://lwn.net/Articles/894598)

[](https://www.phoronix.com/scan.php?page=news_item&px=Linux-5.18-NUMA-Regression-Fix)

### 12.1.2 多级内存(Top-tier memory management)/ 内存分级(memory tiering) 支持
-------

#### 12.1.2.1 NUMA nodes for persistent-memory management
-------

多亏了 Dave Hansen 的补丁 [Allow persistent memory to be used like normal RAM](https://lore.kernel.org/patchwork/patch/1045596), 它使 PMEM 作为 NUMA 节点的一部分当普通内存一样来使用.

如何同时使用 PMEM 和普通 DRAM 仍然是一个悬而未决的问题. 各家厂商提出了不同的思路和想法.

Intel 的吴峰光 [PMEM NUMA node and hotness accounting/migration](https://lore.kernel.org/linux-mm/20181226131446.330864849@intel.com)

[Page demotion for memory reclaim](https://lore.kernel.org/linux-mm/20190321200157.29678-1-keith.busch@intel.com)

阿里巴巴的 Yang Shi [Another Approach to Use PMEM as NUMA Node](https://lore.kernel.org/linux-mm/1553316275-21985-1-git-send-email-yang.shi@linux.alibaba.com), [mm: vmscan: demote anon DRAM pages to PMEM node](https://lore.kernel.org/linux-mm/6A903D34-A293-4056-B135-6FA227DE1828@nvidia.com)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2018/12/26 | Fengguang Wu <fengguang.wu@intel.com> | [PMEM NUMA node and hotness accounting/migration](https://lore.kernel.org/patchwork/patch/1027864) | 1. 隔离 DRAM 和 PMEM. 为 PMEM 单独构造了一个 zonelist, 这样一般的内存分配是不会分配到 PMEM 上的 <br>2. 跟踪内存的冷热. 利用内核中已经有的 idle page tracking 功能 (目前主线内核只支持系统全局的 tracking), 在 per process 的粒度上跟踪内存的冷热 <br>3. 利用现有的 page reclaim, 在 reclai m 时将冷内存迁移到 PMEM 上(只能迁移匿名页).  <br>4. 利用一个 userspace 的 daemon 和 idle page tracking, 来将热内存(在 PMEM 上的) 迁移到 DRA M 中. | RFC v2 ☐ 4.20 | PatchWork RFC,v2,00/21](https://lore.kernel.org/patchwork/patch/1027864), [LKML](https://lkml.org/lkml/2018/12/26/138), [github/intel/memory-optimizer](http://github.com/intel/memory-optimizer) |
| 2019/02/25 | Dave Hansen <dave.hansen@linux.intel.com><br>Huang Ying <ying.huang@intel.com> | [Allow persistent memory to be used like normal RAM](https://lore.kernel.org/patchwork/patch/1045596) | 通过 memory hotplug 的方式把 PMEM 添加到 Linux 的 buddy allocator 里面. 新添加的 PMEM 会以一个或多个 NUMA node 的形式出现, Linux Kernel 就可以分配 PMEM 上的 memory, 这样和使用一般 DRAM 没什么区别 | v5 ☑ 5.1-rc1 | [PatchWork v5,0/5](https://lore.kernel.org/patchwork/patch/1045596) |
| 2019/04/11 | Fengguang Wu <fengguang.wu@intel.com> | [Another Approach to Use PMEM as NUMA Node](https://patchwork.kernel.org/project/linux-mm/cover/1554955019-29472-1-git-send-email-yang.shi@linux.alibaba.com) | 通过 memory reclaim 把 "冷" 内存迁移到慢速的 PMEM node 中, NUMA Balancing 访问到这些冷 page 的时候可以选择是否把这个页迁移回 DRAM, 相当于是一种比较粗粒度的 "热" 内存识别. | RFC v2 ☐ 4.20 | [PatchWork v2,RFC,0/9](https://patchwork.kernel.org/project/linux-mm/cover/1554955019-29472-1-git-send-email-yang.shi@linux.alibaba.com) |
| 2019/04/04 | Zi Yan <zi.yan@sent.com> | [Accelerate page migration and use memcg for PMEM management](https://patchwork.kernel.org/project/linux-mm/cover/20190404020046.32741-1-zi.yan@sent.com) | TODO | RFC ☐ | [PatchWork RFC,00/25](https://patchwork.kernel.org/project/linux-mm/cover/20190404020046.32741-1-zi.yan@sent.com) |


#### 12.1.2.2 memory tiering and demotion
-------


[Top-tier memory management](https://lwn.net/Articles/857133)

[LWN: Top-tier memory management!](https://blog.csdn.net/Linux_Everything/article/details/117970246)

[NUMA rebalancing on tiered-memory systems](https://lwn.net/Articles/893024)

[Linux Developers Discuss Improvements To Memory Tiering](https://www.phoronix.com/scan.php?page=news_item&px=Linux-Better-Memory-Tiering)

[RFC: Memory Tiering Kernel Interfaces (v2)](https://lore.kernel.org/lkml/CAAPL-u-DGLcKRVDnChN9ZhxPkfxQvz9Sb93kVoX_4J2oiJSkUw@mail.gmail.com)

[Two memory-tiering patch sets](https://lwn.net/Articles/898766)

[Explicit Memory Tiers May Be Ready For Linux 6.1](https://www.phoronix.com/news/Linux-6.1-Improve-Memory-Tiers)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/04/15 | Tim Chen <tim.c.chen@linux.intel.com> | [Manage the top tier memory in a tiered memory](https://lore.kernel.org/patchwork/patch/1408180) |  memory tiers 的配置管理. 监控系统和每个 cgroup 中各自使用的 top-tier 内存的数量. 当前使用了 soft limit, 用 kswapd 来把某个 cgroup 中超过 soft limit 限制的 page 迁移到较慢的 memory 类型上去. 这里所说的 soft limit, 是指如果 top-tier memory 很充足的话, cgroup 可以拥有超过此限制的 page 数量, 但如果资源紧张的话则会被迅速削减从而满足这个 limit 值. 后期可用于对于不同的任务划分快、慢内存(fast and slow memory). 即让高优先级的任务获得更多 top-tier memory 访问优先, 而低优先级的任务则要受到更严格的限制 | v1 ☐ 5.13 | [PatchWork RFC,v1,00/11](https://patchwork.kernel.org/project/linux-mm/cover/cover.1617642417.git.tim.c.chen@linux.intel.com/) |
| 2021/07/15 | Dave Hansen <dave.hansen@linux.intel.com><br>Huang Ying <ying.huang@intel.com> | [Migrate Pages in lieu of discard](https://lore.kernel.org/patchwork/patch/1393431) | 页面回收阶段引入页面降级 (demote pages) 策略. 在一个具备了持久性内存的系统中(这些系统具有多种类型的内存, 具有不同的性能特征, 而不是普通 NUMA 系统, 在普通 NUMA 系统中, 相同类型的内存存在于不同的距离), 在回收期间允许页面迁移使这些系统能够在快速层承受压力时将页面从快速层迁移到慢速层. 具体做法是可以把一些需要回收的 page 从 DRAM 迁移到较慢的 memory 中, 后面如果再次需要这些数据了, 也是仍然可以直接访问到, 只是速度稍微慢一些. 目前的版本还不完善, 被迁移的 page 将被永远困在慢速内存区域中, 没有机制使其回到更快的 DRAM.(Huang Ying 后续工作, 重新调整 autonuma 的用途, 将热门页面推广回 DRAM). 这个降级策略可以通过 sysctl 的 ~~vm.zone_reclaim_mode 中把 bitmask 设置为 8 从而启用这个功能~~ `/sys/kernel/mm/numa/demotion_enabled` 开关来控制. 参见报道 [Linux 5.15 Has A Critical Improvement For Tiered Memory Servers](https://www.phoronix.com/scan.php?page=news_item&px=Linux-5.15-Demote-During-Reclai). | v10 ☑ 5.15-rc1 | [PatchWork -V10,0/9](https://patchwork.kernel.org/project/linux-mm/cover/20210715055145.195411-1-ying.huang@intel.com), [PatchWork -V10,0/9](https://lore.kernel.org/all/20210715055145.195411-1-ying.huang@intel.com), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=26aa2d199d6f2cfa6f2ef2a5dfe891f2250e71a0) |
| 2022/02/21 | Huang Ying <ying.huang@intel.com> | [NUMA balancing: optimize memory placement for memory tiering system](https://www.phoronix.com/scan.php?page=news_item&px=Intel-Optimize-PMEM-Place-v9) | 将经常使用的 page 从慢速内存迁移到快速内存的方案. 对 Linux 内核的 NUMA balancing 行为进行了调整, 以优化内存分层系统的内存位置. 这些补丁进一步优化了内核在持久内存存在的情况下对页面的处理, 同时将最重要的页面保留在 DRAM 中.<br> 优化 numa balancing 的页面迁移策略, 利用了这些 numa fault 来对哪些 page 属于常用 page 进行更准确地估算. 新的策略依据从 page unmmap 到发生 page fault 之间的时间差来判断 page 是否常用, 并提供了一个 sysctl 开关来定义阈值: kernel.numa_balancing_hot_threshold_ms. 所有 page fault 时间差低于阈值的 page 都被判定是常用 page. 由于对于系统管理员来说可能很难决定这个阈值应该设置成什么值, 所以这组 patch 中的实现了自动调整的方法. 为了实现这个自动调整, kernel 会根据用户设置的平衡速率限制(balancing rate limit). 内核会对迁移的 page 数量进行监控, 通过增加或减少阈值来使 balancing rate 更接近该目标值. 仓库地址 [vishal/tiering.git](https://git.kernel.org/pub/scm/linux/kernel/git/vishal/tiering.git/log/?h=tiering-0.72). 参见报道 [Intel Continues Optimizing Linux For Optane DC Persistent Memory Servers](https://www.phoronix.com/scan.php?page=news_item&px=Intel-Optimize-PMEM-Place-v9) | v13 ☑ 5.18-rc1 | [PatchWork RFC,-V6,0/6](https://lore.kernel.org/patchwork/patch/1393431)<br>*-*-*-*-*-*-*-* <br>[PatchWork -V8,0/6](https://patchwork.kernel.org/proje3t/linux-mm/cover/20210914013701.344956-1-ying.huang@intel.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork -V9,0/6](https://patchwork.kernel.org/project/linux-mm/cover/20211008083938.1702663-1-ying.huang@intel.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork -V10,0/6](https://patchwork.kernel.org/project/linux-mm/cover/20211116013522.140575-1-ying.huang@intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v13,0/3](https://lore.kernel.org/r/20220221084529.1052339-1-ying.huang@intel.com) |
| 2021/10/14 | Yang Shi <shy828301@gmail.com> | [mm: migrate: make demotion knob depend on migration](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=20f9ba4f995247bb79e243741b8fdddbd76dd923) | NA | v1 ☑✓ 5.16-rc1 | [LORE](https://lore.kernel.org/all/20211015005559.246709-1-shy828301@gmail.com), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=20f9ba4f995247bb79e243741b8fdddbd76dd923) |
| 2021/11/24 | Hasan Al Maruf <hasan3050@gmail.com> | [Transparent Page Placement for Tiered-Memory](https://patchwork.kernel.org/project/linux-mm/cover/cover.1637778851.git.hasanalmaruf@fb.com) | 由于不同类型的内存对性能的影响程度不同, 因此如何跨 NUMA 节点管理页面应该是一个值得关注的问题.  Dave Hansen 的补丁集 ["Migrate Pages in replace of discard"](https://lwn.net/Articles/860215) 在回收过程中将顶级页面降级到慢级节点. 然而, 他的补丁集不包括将慢层内存节点上的页面提升到顶级内存节点的功能. 因此, 在慢层节点上降级或新分配的页面将经历 NUMA 延迟, 并损害应用程序性能. 在这个补丁集中,<br>1. 通过增强现有的 AutoNUMA 机制, 将页面从慢级节点提升到顶级节点.<br>2. 将顶级节点的回收和分配逻辑解耦, 以便回收在更高的水印处触发, 并将较冷的页面降级到慢层内存中. 因此, 顶级节点可以保留一些空闲空间, 以接受来自慢级节点的新分配和提升.<br> 在对页面升级期间, 添加滞后性, 只升级那些在短时间内不太可能被降级页面. 这减少了页面在短时间内频繁降级和提升而在 NUMA 节点之间来回跳转的机会. 作者在支持 cxl 的 DRAM 和 PMEM 层的系统上测试了这个补丁集. 可以将较热的页面转移到顶级节点, 而将较冷的页面移动到慢层节点, 从而产生大量的 Meta 生产工作负载和实时流量. 因此, 顶级节点提供更多的热页面, 应用程序的性能也得到了提高. 将 80% 的匿名者带到顶级节点. 慢层内存中的匿名页大多是冷页. 由于顶级节点不能承载所有热内存, 一些热文件仍然留在慢级节点上. 尽管如此, 远程 NUMA 读带宽从 80% 减少到 40%. 与服务于整个工作集的顶级节点的基线相比, 使用这个补丁集的吞吐量回归仅为 5%. 参见报道 [Facebook/Meta Tackling Transparent Page Placement For Tiered-Memory Linux Systems](https://www.phoronix.com/scan.php?page=news_item&px=Meta-Hot-Pages-High-Tiers). | v1 ☐ | [PatchWork 0/5]https://patchwork.kernel.org/project/linux-mm/cover/cover.1637778851.git.hasanalmaruf@fb.com) |
| 2021/12/11 | Baolin Wang <baolin.wang@linux.alibaba.com> | [Add speculative numa fault support](https://lore.kernel.org/patchwork/patch/1479229) | 这个 RFC 补丁集为分级内存等场景添加了推测性的 numa 错误支持. 在分层内存系统上, 它将依靠 numa 平衡将慢内存和热内存提升为快内存, 以提高性能. 因此, 我们可以根据某些工作负载的数据局部性, 提前在低速内存中提升多个顺序页面, 以提高性能. 那么现在有多少页面需要提升到最快的内存是最好的? 现在这个 RFC 补丁集只实现了一个基本而简单的机制来推测每个 VMA 的 numa 故障窗口. 它将为每个 VMA 引入一个新的原子成员来记录 numa 故障窗口信息, 该信息用于确定它是扩展还是减少 numa 故障窗口的顺序流. 在分层内存系统中测试 mysql 可以看到大约 6% 的改进. 注意: 这个补丁集是[基于实现分级内存提升的补丁集 NUMA balancing: optimize page placement for memory tiering system](https://lore.kernel.org/lkml/87bl2gsnrd.fsf@yhuang6-desk2.ccr.corp.intel.com). 参见报道 [Speculative NUMA Fault Support Proposed For Improving Tiered Memory Linux Performance](https://www.phoronix.com/scan.php?page=news_item&px=Speculative-NUMA-Fault) | v1 ☐ | [LORE](https://lore.kernel.org/lkml/cover.1639306956.git.baolin.wang@linux.alibaba.com), [PatchWork](https://patchwork.kernel.org/project/linux-mm/cover/cover.1639306956.git.baolin.wang@linux.alibaba.com) |
| 2022/07/13 | Huang Ying <ying.huang@intel.com> | [memory tiering: hot page selection](https://patchwork.kernel.org/project/linux-mm/cover/20220408071222.219689-1-ying.huang@intel.com/) | 在分级内存的系统中, 需要使用 NUMA 来优化页面在内存中的位置. 基本上, 最初的 NUMA Balancing 实现选择并提升最近访问最多 (mostly recently accessed/MRU) 的页面. 但最近访问的页面可能很冷. 因此, 在这个补丁集中, 我们基于 NUMA 平衡页表扫描和页面 Page Fault & Hint 之间的延迟实现了一个新的热页识别算法. 另外一方面热页升级可能会在系统中产生一些开销. 为了控制开销, 实现了一种简单的提升速率限制机制. 用于识别热页的热阈值通常取决于工作负载. 因此, 我们还实现了一个热页阈值自动调整算法. 基本思想是增加 / 减少热页阈值, 使通过热页阈值提升到快速内存的页面数接近速率限制. | v1 ☐☑ | [2022/04/08 LORE v1,0/3](https://lore.kernel.org/r/20220408071222.219689-1-ying.huang@intel.com)<br>*-*-*-*-*-*-*-* <br>[2022/06/22 LORE v1,0/3](https://lore.kernel.org/r/20220622083519.708236-1-ying.huang@intel.com)<br>*-*-*-*-*-*-*-* <br>[2022/07/13 LORE v1,0/3](https://lore.kernel.org/r/20220713083954.34196-1-ying.huang@intel.com) |
| 2022/04/13 | Jagdish Gediya <jvgediya@linux.ibm.com> | [mm: demotion: Introduce new node state N_DEMOTION_TARGETS](https://patchwork.kernel.org/project/linux-mm/cover/20220413092206.73974-1-jvgediya@linux.ibm.com/) | 631805 | v2 ☐☑ | [LORE v2,0/5](https://lore.kernel.org/r/20220413092206.73974-1-jvgediya@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/7](https://lore.kernel.org/r/20220422195516.10769-1-jvgediya@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/3](https://lore.kernel.org/r/20220426085105.60822-1-ying.huang@intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/3](https://lore.kernel.org/r/20220510063958.86985-1-ying.huang@intel.com) |
| 2022/07/14 | Aneesh Kumar K.V <aneesh.kumar@linux.ibm.com> | [mm/demotion: Memory tiers and demotion](https://patchwork.kernel.org/project/linux-mm/cover/20220527122528.129445-1-aneesh.kumar@linux.ibm.com/) | 645595 | v4 ☐☑ | [LORE v4,0/7](https://lore.kernel.org/r/20220527122528.129445-1-aneesh.kumar@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/9](https://lore.kernel.org/r/20220603134237.131362-1-aneesh.kumar@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/13](https://lore.kernel.org/r/20220610135229.182859-1-aneesh.kumar@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v7,0/12](https://lore.kernel.org/r/20220622082513.467538-1-aneesh.kumar@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v9,0/8](https://lore.kernel.org/r/20220714045351.434957-1-aneesh.kumar@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v10,0/8](https://lore.kernel.org/r/20220720025920.1373558-1-aneesh.kumar@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v11,0/8](https://lore.kernel.org/r/20220728190436.858458-1-aneesh.kumar@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v12,0/8](https://lore.kernel.org/r/20220729061349.968148-1-aneesh.kumar@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v13,0/9](https://lore.kernel.org/r/20220808062601.836025-1-aneesh.kumar@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v14,0/10](https://lore.kernel.org/r/20220812055710.357820-1-aneesh.kumar@linux.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v15,0/10](https://lore.kernel.org/r/20220818131042.113280-1-aneesh.kumar@linux.ibm.com) |
| 2022/06/06 | Aneesh Kumar K V <aneesh.kumar@linux.ibm.com> | [mm/demotion: Add sysfs ABI documentation](https://patchwork.kernel.org/project/linux-mm/patch/87r1428k9n.fsf@linux.ibm.com/) | 647508 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/87r1428k9n.fsf@linux.ibm.com) |
| 2022/06/07 | Johannes Weiner <hannes@cmpxchg.org> | [mm: mempolicy: N:M interleave policy for tiered memory nodes](https://patchwork.kernel.org/project/linux-mm/patch/20220607171949.85796-1-hannes@cmpxchg.org/) | 648110 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220607171949.85796-1-hannes@cmpxchg.org) |
| 2022/06/14 | Tim Chen <tim.c.chen@linux.intel.com> | [Cgroup accounting of memory tier usage](https://patchwork.kernel.org/project/linux-mm/cover/cover.1655242024.git.tim.c.chen@linux.intel.com/) | 650358 | v1 ☐☑ | [LORE v1,0/3](https://lore.kernel.org/r/cover.1655242024.git.tim.c.chen@linux.intel.com) |
| 2022/08/29 | Aneesh Kumar K V <aneesh.kumar@linux.ibm.com> | [[v2] mm/demotion: Expose memory tier details via sysfs](https://patchwork.kernel.org/project/linux-mm/patch/20220829060745.287468-1-aneesh.kumar@linux.ibm.com/) | 671886 | v2 ☐☑ | [LORE v2,0/1](https://lore.kernel.org/r/20220829060745.287468-1-aneesh.kumar@linux.ibm.com) |
| 2022/10/20 | Huang, Ying <ying.huang@intel.com> | [memory tier, sysfs: rename attribute"nodes"to"nodelist"](https://patchwork.kernel.org/project/linux-mm/patch/20221020015122.290097-1-ying.huang@intel.com/) | 686946 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221020015122.290097-1-ying.huang@intel.com) |
| 2022/11/22 | Mina Almasry <almasrymina@google.com> | [[RFC,V1] mm: Disable demotion from proactive reclaim](https://patchwork.kernel.org/project/linux-mm/patch/20221122203850.2765015-1-almasrymina@google.com/) | 698228 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221122203850.2765015-1-almasrymina@google.com) |



## 12.2 远端内存
-------

[持久内存 - RDMA 让远程数据不再远](http://blog.itpub.net/31493717/viewspace-2731431)

[详谈 RDMA(远程直接内存访问)技术原理和三种实现方式](https://blog.csdn.net/Rong_Toa/article/details/114747763)

[关于 RDMA 技术原理、三种主流实现技术对比](https://www.idcbest.com/idcnews/11004565.html)

## 12.3 异构内存(CPU & GPU)
-------


AMD 正在研究一种异构超级计算机架构, 该架构在 CPU 和 GPU 之间具有一致的互连. 这种硬件架构允许 CPU 连贯地访问 GPU 设备内存. Alex Sierra 所在的实验室正在与合作伙伴 HPE 合作, 开发 BIOS、固件和软件, 以交付给 DOE.

系统 BIOS 在 UEFI 系统地址映射中将 GPU 设备内存 (也称为 VRAM) 作为 SPM(专用内存)播发. AMD GPU 驱动程序可以通过 lookup_resource() 查找到这些内存资源, 并使用  devm_memremap_pages() 将其注册为通用内存设备 (MEMORY_DEVICE_GENERIC) 使用.

补丁集 [Support DEVICE_GENERIC memory in migrate_vma_*](https://lore.kernel.org/patchwork/patch/1472581) 进行了一些更改, 尝试通过使用 `migrate_vma_*()` 辅助函数将数据迁移到该内存中, 以便在统一内存分配中支持基于页面的迁移, 同时还支持 CPU 直接访问对这些页面.

这项工作基于 HMM 和 SVM 内存管理器, 该内存管理器最近被上传到 Dave Airlie 的 [drm-next](https://cgit.freedesktop.org/drm/drm/log/?h=drm-next), 其中对用于迁移的 VRAM 管理进行了一些返工, 以消除一些不正确的假设, 允许在 VRAM 和系统内存中混合页面的部分成功迁移和 GPU 内存映射, 参见 [drm/amdkfd: add owner ref param to get hmm pages Felix](https://lore.kernel.org/dri-devel/20210527205606.2660-6-Felix.Kuehling@amd.com/T/#r996356015e295780eb50453e7dbd5d0d68b47cbc)


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/08/05 | Alex Sierra <alex.sierra@amd.com> | [Support DEVICE_GENERIC memory in migrate_vma_*](https://lore.kernel.org/patchwork/patch/1472581) | NA | v1 ☐ | [PatchWork v6,00/13](https://patchwork.kernel.org/project/linux-mm/cover/20210813063150.2938-1-alex.sierra@amd.com) |





# 13 内存管理调试支持
-------




由前面所述, 内存管理相当复杂, 代码量巨大, 而它又是如此重要的一个的子系统, 所以代码质量也要求非常高. 另一方面, 系统各个组件都是内存管理子系统的使用者, 而如果缺乏合理有效的约束, 不正当的内存使用 (如内存泄露, 内存覆写) 都将引起系统的崩溃, 以至于数据损坏. 基于此, 内存管理子系统引入了一些调试支持工具, 方便开发者 / 用户追踪, 调试内存管理及内存使用中的问题. 本章介绍内存管理子系统中几个重要的调试工具.



## 13.1 页分配的调试支持
-------


**2.5(2003 年 7 月之后发布)**

前面提到过, 内核自己用的内存, 由于是绕过寻常的逐级页表机制, 采用直接映射(提高了效率), 即虚拟地址与页面实际的物理地址存在着一一线性映射的关系. 另一方面, 内核使用的内存又是出于各种重要管理目的, 比如驱动, 模块, 文件系统, 甚至 SLAB 子系统也是构建于页分配器之上. 以上二个事实意味着, 相邻的页, 可能被用于完全不同的目的, 而这两个页由于是直接映射, 它们的虚拟地址也是连续的. 如果某个使用者子系统的编码有 bug, 那么它的对其内存页的写操作造成对相邻页的覆写可能性相当大, 又或者不小心读了一个相邻页的数据. 这些操作可能不一定马上引起问题, 而是在之后的某个地方才触发, 导致数据损坏乃至系统崩溃.



为此, 2.5 中, 针对 Intel 的 i386 平台, 内核引入了一个 **CONFIG\_DEBUG\_PAGEALLOC** 开关, 它在页分配的路径上插入钩子, 并利用 i386 CPU 可以对页属性进行修改的特性, 通过修改未分配的页的页表项的属性, 把该页置为 "隐藏". 因此, 一旦不小心访问该页(读或写), 都将引起处理器的缺页异常, 内核将进入缺页处理过程, 因而有了一个可以检查捕捉这种内存破坏问题的机会.



在 2.6.30 中, 又增加了对无法作处理器级别的页属性修改的体系的支持. 对于这种体系, 该特性是将未分配的页 ** 毒化(POISON),** 写入特定模式的值, 因而一旦被无意地访问(读或写), 都将可能在之后的某个时间点被发现. ** 注意, 这种通用的方法就无法像前面的有处理器级别支持的方法有立刻捕捉的机会.**





** 当然, 这个特性是有性能代价的, 所以生产系统中可别用哦.**



## 13.2 SLAB 子系统的调试支持
-------


SLAB 作为一个相对独立的子模块, 一直有自己完善的调试支持, 包括有:



- 对已分配对象写的边界超出的检查

- 对未初始化对象写的检查

- 对内存泄漏或多次释放的检查

- 对上一次分配者进行记录的支持等

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/11/05 | Kees Cook <keescook@chromium.org> | [mm/slab_common: Restore passing"caller"for tracing](https://patchwork.kernel.org/project/linux-mm/patch/20221105063529.never.818-kees@kernel.org/) | 692349 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221105063529.never.818-kees@kernel.org) |


## 13.3 内存检测工具
-------


### 13.3.1 错误注入机制
-------

**2.6.20(2007 年 2 月发布)**


内核有着极强的健壮性, 能对各种错误异常情况进行合理的处理. 然而, 毕竟有些错误实在是极小可能发生, 测试对这种小概率异常情况的处理的代码实在是不方便. 所以, 2.6.20 中内核引入了错误注入机制, 其中跟 MM 相关的有两个, 一个是对页分配器的失败注入, 一个是对 SLAB 对象分配器的失败注入. 这两个注入机制, 可以触发内存分配失败, 以测试极端情况下 (如内存不足) 系统的处理情况.

### 13.3.2 KMEMCHECK - 内存非法访问检测工具
-------

**2.6.31(2009 年 9 月发布)**

对内存的非法访问, 如访问未分配的内存, 或访问分配了但未初始化的内存, 或访问了已释放了的内存, 会引起很多让人头痛的问题, 比如程序因数据损坏而在某个地方莫名崩溃, 排查非常困难. 在用户态内存检测工具 valgrind 中, 有一个 Memcheck 插件可以检测此类问题. 2.6.31, Linux 内核也引进了内核态的对应工具, 叫 KMEMCHECK.



它是一个内核态工具, 检测的是内核态的内存访问. 主要针对以下问题:

> 1. 对已分配的但未初始化的页面的访问
> 2. 对 SLAB 系统中未分配的对象的访问
> 3. 对 SLAB 系统中已释放的对象的访问

为了实现该功能, 内核引入一个叫 ** 影子页(shadow page)** 的概念, 与被检测的正常 ** 数据页 ** 一一相对. 这也意味着启用该功能, 不仅有速度开销, 还有很大的内存开销.



在分配要被追踪的数据页的同时, 内核还会分配等量的影子页, 并通过数据页的管理数据结构中的一个 **shadow** 字段指向该影子页. 分配后, 数据页的页表中的 **present** 标记会被清除, 并且标记为被 KMEMCHECK 跟踪. 在第一次访问时, 由于 **present** 标记被清除, 将触发缺页异常. 在缺页异常处理程序中, 内核将会检查此次访问是不是正常. 如果发生上述的非法访问, 内核将会记录下该地址, 错误类型, 寄存器, 和栈的回溯, 并根据配置的值大小, 把该地址附近的数据页和影子页的对应内容, 一并保存到一个缓冲区中. 并设定一个稍后处理的软中断任务, 把这些内容报告给上层. 所有这些之后, 将 CPU 标志寄存器 TF (Trap Flag)置位, 于是在缺页异常后, CPU 又能重新访问这条指令, 走正常的执行流程.



这里面有个问题是, **present** 标记在第一次缺页异常后将被置位, 之后如果再次访问, KMEMCHECK 如何再次利用该机制来检测呢? 答案是内核在上述处理后还 ** 打开了单步调试功能,** 所以 CPU 接着执行下一条指令前, 又陷入了调试陷阱(debug trap, 详情请查看 CPU 文档), 在处理程序中, 内核又会把该页的 **present** 标记会清除.


## 13.3.3 KMEMLEAK - 内存泄漏检测工具
-------

**2.6.31(2009 年 9 月发布)**

内存漏洞一直是 C 语言用户面临的一个问题, 内核开发也不例外. 2.6.31 中, 内核引入了 [KMEMLEAK 工具](https://lwn.net/Articles/187979), 用以检测内存泄漏. 它采用了标记 - 清除的垃圾收集算法, 对通过 SLAB 子系统分配的对象, 或通过 _vmalloc_ 接口分配的连续虚拟地址的对象, 或分配的 per-CPU 对象 (per-CPU 对象是指每个 CPU 有一份拷贝的全局对象, 每个 CPU 访问修改本地拷贝, 以提高性能) 进行追踪, 把指向对象起始的指针, 对象大小, 分配时的栈踪迹(stack trace) 保存在一个红黑树里(便于之后的查找, 同时还会把对象加入一个全局链表中). 之后, KMEMLEAK 会启动一个每 10 分钟运行一次的内核线程, 或在用户的指令下, 对整个内存进行扫描. 如果某个对象 ** 从其起始地址到终末地址 ** 内没有别的指针指向它, 那么该对象就被当成是泄漏了. KMEMLEAK 会把相关信息报告给用户.



扫描的大致算法如下:



> 1. 首先会把全局链表中的对象加入一个所谓的 ** 白名单 ** 中, 这是所有待查对象. 然后 , 依次扫描数据区(data 段, bss 段), per-CPU 区, 还有针对每个 NUMA 节点的所有页. 另外, 如果用户有指定, 还会描扫所有线程的栈区(之所以这个不是强制扫描, 是因为栈区是函数的活动记录, 变动迅速, 引用可能稍纵即逝). 扫描过程中, 与之前存的红黑树中的对象进行比对, 一旦发现有指针指向红黑树中的对象, 说明该对象仍有人引用 , 没被泄漏, 把它加入一个所谓的 ** 灰名单 ** 中.
> 2. 然后, 再扫描一遍灰名单, 即已经被确认有引用的对象, 找出这些对象可能引用的别的所有对象, 也加入灰名单中.
> 3. 最后剩下的, 在白名单中的, 就是被 KMEMLEAK 认为是泄漏了的对象.

由于内存的引用情况各异, 存在很多特殊情况, 可能存在误报或漏报的情况, 所以 KMEMLEAK 还提供了一些接口, 方便使用者告知 KMEMLEAK 某些对象不是泄露, 某些对象不用检查, 等等.

这个工具当然也存在着显著的影响系统性能的问题, 所以也只是作为调试使用.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/09/20 | zhaoyang.huang <zhaoyang.huang@unisoc.com> | [[RFC] mm: track bad page via kmemleak](https://patchwork.kernel.org/project/linux-mm/patch/1663679468-16757-1-git-send-email-zhaoyang.huang@unisoc.com/) | 678650 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/1663679468-16757-1-git-send-email-zhaoyang.huang@unisoc.com)[LORE v2,0/1](https://lore.kernel.org/r/1663730246-11968-1-git-send-email-zhaoyang.huang@unisoc.com) |
| 2022/10/27 | zhaoyang.huang <zhaoyang.huang@unisoc.com> | [[Resend,PATCHv2] mm: use stack_depot for recording kmemleak's backtrace](https://patchwork.kernel.org/project/linux-mm/patch/1666864224-27541-1-git-send-email-zhaoyang.huang@unisoc.com/) | 689350 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/1664447407-8821-1-git-send-email-zhaoyang.huang@unisoc.com)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/1](https://lore.kernel.org/r/1666864224-27541-1-git-send-email-zhaoyang.huang@unisoc.com) |


### 13.3.4 KASAN - 内核地址净化器
-------

**4.0(2015 年 4 月发布)**

[ASAN 和 HWASAN 原理解析](https://blog.csdn.net/21cnbao/article/details/107096665)

[浅谈 ARM64 基于硬件 tag 的 KASAN](http://news.eeworld.com.cn/mp/ymc/a133074.jspx)

#### 13.3.4.1 Generic KASAN
-------


4.0 引入的 [KASan (The kernel address sanitizer)](https://lwn.net/Articles/612153) 可以看作是 KMEMCHECK 工具的替代器, 它的目的也是为了检测诸如 ** 释放后访问(use-after-free), 访问越界(out-of-bouds)** 等非法访问问题. 它比后者更快, 因为它利用了编译器的 **Instrument** 功能, 也即编译器会在访问内存前插入探针, 执行用户指定的操作, 这通常用在性能剖析中. 在内存的使用上, KASan 也比 KMEMCHECK 有优势: 相比后者 1:1 的内存使用 , 它只要 1:1/8.


总的来说, 它利用了 GCC 5.0 的新特性, 可对内核内存进行 Instrumentaion, 编译器可以在访问内存前插入指令, 从而检测该次访问是否合法. 对比前述的 KMEMCHECK 要用到 CPU 的陷阱指令处理和单步调试功能, KASan 是在编译时加入了探针, 因此它的性能更快.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/08/04 | Andrey Konovalov <andreyknvl@google.com> | [kasan: support stack instrumentation for tag-based mode](https://lore.kernel.org/patchwork/patch/1284062) | NA | v11 ☑ 5.9-rc1 | [PatchWork v2,0/5](https://lore.kernel.org/patchwork/patch/1284062), [Clang](https://clang.llvm.org/docs/HardwareAssistedAddressSanitizerDesign.html) |
| 2022/03/23 | andrey.konovalov@linux.dev <andrey.konovalov@linux.dev> | [kasan, arm64, scs, stacktrace: collect stack traces from Shadow Call Stack](https://patchwork.kernel.org/project/linux-mm/cover/cover.1648049113.git.andreyknvl@google.com) | 目前, 当保存 alloc 和 free 堆栈跟踪时, kasan 总是使用正常的堆栈跟踪收集例程, 它依赖于 unwind. 通过复制帧从阴影调用堆栈收集堆栈跟踪, 每当它被启用. 这减少了 30% 的启动时间, 所有的 KASAN 模式时, 影子呼叫堆栈被启用. 堆栈栈位通过新的 stack_trace_save_shadow () 接口从 Shadow Call Stack 中收集. | v2 ☐☑ | [LORE v2,0/4](https://lore.kernel.org/r/cover.1648049113.git.andreyknvl@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/3](https://lore.kernel.org/r/cover.1649877511.git.andreyknvl@google.com) |
| 2022/09/10 | Peter Collingbourne <pcc@google.com> | [kasan: also display registers for reports from HW exceptions](https://patchwork.kernel.org/project/linux-mm/patch/20220910052426.943376-1-pcc@google.com/) | 675886 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220910052426.943376-1-pcc@google.com) |
| 2022/10/11 | Josh Poimboeuf <jpoimboe@kernel.org> | [kasan: disable stackleak plugin in report code](https://patchwork.kernel.org/project/linux-mm/patch/20221011190548.blixlqj6dripitaf@treble/) | 684584 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221011190548.blixlqj6dripitaf@treble) |
| 2022/10/19 | David Gow <davidgow@google.com> | [kasan: Enable KUnit integration whenever CONFIG_KUNIT is enabled](https://patchwork.kernel.org/project/linux-mm/patch/20221019085747.3810920-1-davidgow@google.com/) | 686607 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221019085747.3810920-1-davidgow@google.com) |
| 2023/02/24 | Marco Elver <elver@google.com> | [[v5,1/4] kasan: Emit different calls for instrumentable memintrinsics](https://patchwork.kernel.org/project/linux-mm/patch/20230224085942.1791837-1-elver@google.com/) | 724563 | v5 ☐☑ | [LORE v5,0/4](https://lore.kernel.org/r/20230224085942.1791837-1-elver@google.com) |


#### 13.3.4.2 Software tag-based KASAN
-------

基于软件 tag 的 KASAN 使用软件内存标记方法来检查访问的内存是否有效, 目前仅在 arm64 架构上实现. 基于软件 tag 的 KASAN 使用 arm64 CPU 的 Top Byte Ignore(TBI) 功能将指针 tag 存储在指针的顶部字节中. 它使用影子内存来存储与每个 16 字节内存单元关联的内存 tag(因此, 它将内核内存的 1/16 专用于影子内存).

1.  在每次内存分配时, 生成一个随机 tag, 用这个 tag 标记分配的内存, 并将相同的 tag 嵌入到返回的指针中.

2.  编译时, 在每次内存访问之前插入检测指令, 这些检测指令会比较正在访问的内存的 tag(影子区)和用于访问该内存的指针的 tag 是否匹配.

3.  如果不匹配, 基于软件 tag 的 KASAN 会打印错误报告.

Software tag-based KASAN 使用 0xFF 作为匹配所有指针 tag(不检查通过带有 0xFF 指针 tag 的指针进行的访问). 0xFE 用于标记释放的内存区域.

目前 Software tag-based KASAN 只支持 slab 和 page_alloc 内存检测.

简单总结下 Software tag-based KASAN 的特点:

1.  编译时插入用于检测内存访问指令.
2.  专用的影子内存大小为检测内存大小的 1/16.
3.  通过指令对比内存 tag (影子区域) 和指针 tag.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2018/12/06 | Andrey Konovalov <andreyknvl@google.com> | [kasan: add software tag-based mode for arm64](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=e886bf9d9abedf8236464bfd21bc5707748b4a02) | 该补丁集为 [KASAN](https://www.kernel.org/doc/html/latest/dev-tools/kasan.html) 添加了一种新的基于软件标签的模式. 最初这种模式被称为 KHWASAN, 但后来被重新命名. 通过使用 arm64 CPU 的 Top Byte Ignore(TBI) 特性, 可以在每个内核指针的顶端字节中存储指针标记. 将现有的 KASAN 模式被重命名为 generic KASAN(generic 意味着它是纯软件实现, 可以在任何架构上轻易支持). | v13 ☑✓ 5.0-rc1 | [LORE v13,0/25](https://lore.kernel.org/all/cover.1544099024.git.andreyknvl@google.com) |


#### 13.3.4.2 Hardware tag-based KASAN
-------

ARM 引入了一个 [内存标签扩展](https://community.arm.com/developer/ip-products/processors/b/processors-ip-blog/posts/enhancing-memory-safety) 的硬件特性. 然后 Google 的 [Andrey Konovalov](https://github.com/xairy/linux)基于此硬件特性重写了 KASan. 实现了 [hardware tag-based mode 的 KASan](https://lore.kernel.org/patchwork/patch/1344197). 参见 [The Arm64 memory tagging extension in Linux](https://lwn.net/Articles/834289), 译文 [LWN：Arm64 的内存标记扩展功能!](https://blog.csdn.net/Linux_Everything/article/details/109396397).

基于硬件 tag 的 KASAN 和基于软件 tag 的 KASAN 原理非常类似, Software tag-based KASAN 的内存的 tag(影子区)和用于访问该内存的指针的 tag 的比较是由插入的汇编指令完成的, 而 Hardware tag-based KASAN 是由硬件自动完成比较的, 对于软件是透明的. Hardware tag-based KASAN 目前也是仅支持 arm64 架构, 并且它的实现是基于 ARMv8.5 指令集架构中引入的内存标记扩展 (MTE) 和 Top Byte Ignore(TBI).

在每次内存访问时, 硬件都会比较正在访问的内存的 tag 和用于访问该内存指针的 tag. 如果 tag 不匹配, 则会触发一个异常.

基于硬件 tag 的 KASAN 使用 0xFF 作为匹配所有指针 tag(不检查通过带有 0xFF 指针 tag 的指针进行的访问). 0xFE 用于标记释放的内存区域. 同和 Software tag-based KASAN 一样, 当前只支持 slab 和 page_alloc 检测. 这些特性. 如果硬件不支持 MTE(ARMv8.5 之前), 则不会启用 Hardware tag-based KASAN.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/11/23 | Andrey Konovalov <andreyknvl@google.com> | [kasan: add hardware tag-based mode for arm64](https://lore.kernel.org/patchwork/patch/1344197) | NA | v11 ☑ [5.11-rc1](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork mm,v11,00/42](https://patchwork.kernel.org/project/linux-mm/cover/cover.1606161801.git.andreyknvl@google.com) |
| 2020/11/23 | Andrey Konovalov <andreyknvl@google.com> | [kasan: boot parameters for hardware tag-based mode](https://lore.kernel.org/patchwork/patch/1344258) | NA | v4 ☑ [5.11-rc1](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork mm,v4,00/19](https://patchwork.kernel.org/project/linux-mm/patch/748daf013e17d925b0fe00c1c3b5dce726dd2430.1606162397.git.andreyknvl@google.com) |
| 2021/01/15 | Andrey Konovalov <andreyknvl@google.com> | [kasan: HW_TAGS tests support and fixes](https://lore.kernel.org/patchwork/patch/1366086) | NA | v4 ☑ [5.12-rc1](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork mm,v4,00/15](https://patchwork.kernel.org/project/linux-mm/cover/cover.1610733117.git.andreyknvl@google.com) |
| 2021/02/05 | Andrey Konovalov <andreyknvl@google.com> | [kasan: optimizations and fixes for HW_TAGS](https://lore.kernel.org/patchwork/patch/1376340) | NA | v3 ☑ [5.12-rc1](https://kernelnewbies.org/Linux_5.11#Memory_management) | [PatchWork mm,v3,mm,00/13](https://patchwork.kernel.org/project/linux-mm/cover/cover.1612546384.git.andreyknvl@google.com) |
| 2021/12/30 | andrey.konovalov@linux.dev | [kasan, vmalloc, arm64: add vmalloc tagging support for SW/HW_TAGS](https://patchwork.kernel.org/project/linux-mm/cover/cover.1638308023.git.andreyknvl@google.com/) | NA | v1 ☐ | [2021/11/30 PatchWork 00/31](https://patchwork.kernel.org/project/linux-mm/cover/cover.1638308023.git.andreyknvl@google.com)<br>*-*-*-*-*-*-*-* <br>[2021/12/13 PatchWork v3,00/38](https://patchwork.kernel.org/project/linux-mm/cover/cover.1639432170.git.andreyknvl@google.com)<br>*-*-*-*-*-*-*-* <br>[2021/12/20 PatchWork v4,00/39](https://patchwork.kernel.org/project/linux-mm/cover/cover.1640036051.git.andreyknvl@google.com))<br>*-*-*-*-*-*-*-* <br>[2021/12/30 PatchWork v5,00/39](https://patchwork.kernel.org/project/linux-mm/cover/cover.1640891329.git.andreyknvl@google.com) |
| 2022/02/02 | Christophe Leroy <christophe.leroy@csgroup.eu> | [mm/kasan: Add CONFIG_KASAN_SOFTWARE](https://patchwork.kernel.org/project/linux-mm/patch/a480ac6f31eece520564afd0230c277c78169aa5.1643791473.git.christophe.leroy@csgroup.eu/) | 610616 | v1 ☐ | [PatchWork v1,0/4](https://lore.kernel.org/r/a480ac6f31eece520564afd0230c277c78169aa5.1643791473.git.christophe.leroy@csgroup.eu) |
| 2022/06/13 | andrey.konovalov@linux.dev <andrey.konovalov@linux.dev> | [kasan: switch tag-based modes to stack ring from per-object metadata](https://patchwork.kernel.org/project/linux-mm/cover/cover.1655150842.git.andreyknvl@google.com/) | 649958 | v1 ☐☑ | [LORE v1,0/32](https://lore.kernel.org/r/cover.1655150842.git.andreyknvl@google.com) |
| 2022/10/27 | andrey.konovalov@linux.dev <andrey.konovalov@linux.dev> | [kasan: allow sampling page_alloc allocations for HW_TAGS](https://patchwork.kernel.org/project/linux-mm/patch/c124467c401e9d44dd35a36fdae1c48e4e505e9e.1666901317.git.andreyknvl@google.com/) | 689571 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/c124467c401e9d44dd35a36fdae1c48e4e505e9e.1666901317.git.andreyknvl@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/2](https://lore.kernel.org/r/4c341c5609ed09ad6d52f937eeec28d142ff1f46.1669489329.git.andreyknvl@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/1](https://lore.kernel.org/r/323d51d422d497b3783dacb130af245f67d77671.1671228324.git.andreyknvl@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/1](https://lore.kernel.org/r/129da0614123bb85ed4dd61ae30842b2dd7c903f.1671471846.git.andreyknvl@google.com) |

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2019/07/23 | Andrey Konovalov <andreyknvl@google.com> | [arm64: untag user pointers passed to the kernel](https://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git/log/?id=9ce1263033cd2ad393e2ff0df4a1c4ab4992c9df) | 扩展内核 ABI 以允许将标记的用户指针 (顶部字节设置为 0x00 以外的其他值) 作为系统调用参数传递的系列的一部分. 通过 CONFIG_TAGGED_ADDR_ABI 选项控制. | v19 ☐☑✓ | [LORE v19,0/15](https://lore.kernel.org/all/cover.1563904656.git.andreyknvl@google.com) |


### 13.3.5 KCSAN
-------


[KernelDOC: The Kernel Concurrency Sanitizer (KCSAN)](https://www.kernel.org/doc/html/latest/dev-tools/kcsan.html)

开源书籍 [PerfBook](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html), 由前 IBM 开发者, RCU 的作者和 Maintainer Paul 撰写. 国内的书籍 <<深入理解并行编程>> 就是其翻译版本.

[An introduction to lockless algorithms](https://lwn.net/Articles/844224)

[Detecting missing memory barriers with KCSAN](https://lwn.net/Articles/877200)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/11/30 | Andrey Konovalov <andreyknvl@google.com> | [kcsan: Support detecting a subset of missing memory barriers](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=bd3d5bd1a0ad386475ea7a3de8a91e7d8a600536) | KCSAN 增加对 LKMM 定义的弱内存子集建模的支持, 它支持检测由于丢失内存障碍而导致的数据竞争子集.<br> 当内存操作的结果应该由 barrier 来排序时, KCSAN 可以检测数据竞争, 在这种情况下, 冲突只发生在由于重新排序访问而丢失 barrier 的情况下.<br>KCSAN 检测内存障碍缺失的方法是基于对访问重新排序的建模, 设置了观察点检测对每个内存访问, 也选择在其功能范围内进行模拟重排序. 由于运行时不能 "预取" 访问, 我们只能对延迟访问效果进行建模, 一旦选择了某个访问进行重新排序, 就会在每次其他访问中检查它, 直到函数范围结束. 如果遇到适当的内存障碍, 访问将不再考虑重新排序. | v3 ☑ 5.17-rc1 | [2021/10/05 PatchWork 00/23](https://patchwork.kernel.org/project/linux-mm/cover/20211005105905.1994700-1-elver@google.com)<br>*-*-*-*-*-*-*-* <br>[2021/11/18 PatchWork v2,00/23](https://patchwork.kernel.org/project/linux-mm/cover/20211118081027.3175699-1-elver@google.com)<br>*-*-*-*-*-*-*-* <br>[2021/11/30 PatchWork v3,00/25](https://patchwork.kernel.org/project/linux-mm/cover/20211130114433.2580590-1-elver@google.com)<br>*-*-*-*-*-*-*-* <br>[2021/12/24 LORE kcsan for 5.17, 00/29](https://lore.kernel.org/all/20211214220356.GA2236323@paulmck-ThinkPad-P17-Gen-1) |

### 13.3.6 KMSAN
-------


KernelMemorySanitizer (KMSAN) 是一个检测与未初始化内存使用相关的错误的检测器. 它依赖于 Clang 工具实现, 类似于用户空间 [MemorySanitizer](https://clang.llvm.org/docs/MemorySanitizer.html), 并跟踪内核内存的每一位状态, 如果在条件 (condition)、解引用(dereferenced) 或转译 (escapes) 到用户空间 或者 DMA 时使用了未初始化的值, 则能够报告错误.

KMSAN 最早的 github 仓库位于 : https://github.com/google/kmsan

v3 的报道 [KMSAN Patches For The Linux Kernel Updated For Catching Uninitialized Memory Problems](https://www.phoronix.com/scan.php?page=news_item&px=KernelMemorySanitizer-2022).

v4 的报道 [KernelMemorySanitizer v4 Published While Already Having Found 300+ Kernel Bugs](https://www.phoronix.com/scan.php?page=news_item&px=KernelMemorySanitizer-v4).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/08/26 | Alexander Potapenko <glider@google.com> | [Add KernelMemorySanitizer infrastructure](https://patchwork.kernel.org/project/linux-mm/cover/20211214162050.660953-1-glider@google.com) | 该补丁集包含 KMSAN 运行时实现, 以及使 KMSAN 工作所需的对其他子系统的修改.<br>1. 添加 KMSAN 所需的现有代码的更改和重构.<br>2. 通用代码中的 KMSAN 相关声明、KMSAN 运行时库.<br>3. 添加来自不同子系统的钩子, 以通知 KMSAN 有关内存的信息.<br>4. 通过显式初始化内存、禁用可能欺骗 KMSAN 的优化代码、有选择地跳过检测来防止错误报告的更改.<br>5. Noinstr 的处理.<br> 这个补丁集允许在 QEMU 上启动并运行 defconfig + KMSAN 内核, 而不会出现已知的误报. 然而, 它不能保证在某些设备或测试较少的子系统的驱动程序中没有误报, 尽管 KMSAN 在 syzbot 上使用较大的配置进行了积极的测试. | v1 ☐ | [PatchWork 00/43](https://patchwork.kernel.org/project/linux-mm/cover/20211214162050.660953-1-glider@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,00/46](https://lore.kernel.org/lkml/20220426164315.625149-1-glider@google.com)<br>*-*-*-*-*-*-*-* <br>[2022/07/01 LORE v4,00/45](https://lore.kernel.org/all/20220701142310.2188015-1-glider@google.com)<br>*-*-*-*-*-*-*-* <br>[2022/08/26 LORE v5,0/44](https://lore.kernel.org/r/20220826150807.723137-1-glider@google.com)<br>*-*-*-*-*-*-*-* <br>[2022/09/05 LORE v6,0/44](https://lore.kernel.org/r/20220905122452.2258262-1-glider@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v7,0/43](https://lore.kernel.org/r/20220915150417.722975-1-glider@google.com) |






补丁集是相对于 Linux v5.16-rc5 生成的.

### 13.3.7 KFENCE 一个新的内存安全检测工具
-------

5.12 引入一个[新的内存错误检测工具 : KFENCE (Kernel Electric-Fence, 内核电子栅栏)](https://mp.weixin.qq.com/s/62Oht7WRnKPUBdOJfQ9cBg), 是一个低开销的基于采样的内存安全错误检测器, 用于检测堆后释放、无效释放和越界访问错误. 该系列使 KFENCE 适用于 x86 和 arm64 体系结构, 并将 KFENCE 钩子添加到 SLAB 和 SLUB 分配器.

KFENCE 被设计为在生产内核中启用, 它的性能开销几乎为零. 与 KASAN 相比, KFENCE 以性能换取精度. KFENCE 设计背后的主要动机是, 如果总正常运行时间足够长, KFENCE 将检测非生产测试工作负载通常不会执行的代码路径中的 BUG.

KFENCE 的灵感来自于 [GWP-ASan](http://llvm.org/docs/GwpAsan.html), 这是一个具有类似属性的用户空间工具. "KFENCE" 这个名字是对 [Electric Fence Malloc Debugger](https://linux.die.net/man/3/efence) 的致敬.

[利器解读！Linux 内核调测中最最让开发者头疼的 bug 有解了｜龙蜥技术](https://openanolis.cn/blog/detail/553167069912907824)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/11/03 | Marco Elver <elver@google.com> | [KFENCE: A low-overhead sampling-based memory safety error detector](https://lore.kernel.org/patchwork/patch/1331483) | 轻量级基于采样的内存安全错误检测器 | v7 ☑ 5.12-rc1 | [PatchWork v24](https://lore.kernel.org/patchwork/patch/1331483) |
| 2020/04/21 | Marco Elver <elver@google.com> | [kfence: optimize timer scheduling](https://lore.kernel.org/patchwork/patch/1416384) | ARM 支持 PTDUMP | RFC ☑ 3.19-rc1 | [PatchWork v24](https://lore.kernel.org/patchwork/patch/1416384) |
| 2020/09/17 | Marco Elver <elver@google.com> | [kfence: count unexpectedly skipped allocations](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=5cc906b4b4a510b274113ddb3f88d60644553f79) | ARM 支持 PTDUMP | RFC ☑ 5.16-rc1 | [PatchWork 1/3](https://patchwork.kernel.org/project/linux-mm/patch/20210917110756.1121272-1-elver@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/5](https://lore.kernel.org/all/20210923104803.2620285-1-elver@google.com) |
| 2021/10/19 | Marco Elver <elver@google.com> | [kfence: always use static branches to guard kfence_alloc()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=4f612ed3f748962cbef1316ff3d323e2b9055b6e) | TODO | v1 ☐☑✓ | [LORE v1,0/2](https://lore.kernel.org/all/20211019102524.2807208-1-elver@google.com) |
| 2022/03/07 | Tianchen Ding <dtcccc@linux.alibaba.com> | [provide the flexibility to enable KFENCE](https://patchwork.kernel.org/project/linux-mm/cover/20220307074516.6920-1-dtcccc@linux.alibaba.com/) | 620847 | v3 ☐☑ | [LORE v3,0/2](https://lore.kernel.org/r/20220307074516.6920-1-dtcccc@linux.alibaba.com) |
| 2022/03/08 | Marco Elver <elver@google.com> | [kfence: allow use of a deferrable timer](https://patchwork.kernel.org/project/linux-mm/patch/20220308141415.3168078-1-elver@google.com/) | 采样间隔控制设置 KFENCE 分配的计时器. 默认情况下, 为了保持真实采样间隔的可预测性, 正常计时器会在系统完全空闲时导致 CPU 唤醒. 这在功率受限的系统上可能是不可取的. 因此这个补丁允许 KFENCE 使用可延迟 timer, 当系统空闲时, 它不会强制 CPU 唤醒. 引入一个选项 CONFIG_KFENCE_DEFERRABLE 来开启, 同时提供启动参数 ``kfence. deferrable=1`` 来切换到一个 "deferrable timer" 计时器, 该计时器不会强制在空闲系统上唤醒 CPU, 从而冒着采样间隔不可预测的风险. 使用延迟 timer 是样本间隔变得非常不可预测, 以至于无法保证 KFENCE-KUnit 测试仍然通过. 然而, 在功率受限的系统上, 这可能更可取, 因此, 如果用户接受上述权衡. | v2 ☐☑ | [LORE v2,0/1](https://lore.kernel.org/r/20220308141415.3168078-1-elver@google.com) |
| 2022/06/09 | Jason A. Donenfeld <Jason@zx2c4.com> | [mm/kfence: select random number before taking raw lock](https://patchwork.kernel.org/project/linux-mm/patch/20220609121709.12939-1-Jason@zx2c4.com/) | 648846 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220609121709.12939-1-Jason@zx2c4.com)<br>*-*-*-*-*-*-*-*<br>[LORE v2,0/1](https://lore.kernel.org/r/20220609123319.17576-1-Jason@zx2c4.com) |
| 2022/06/28 | Yee Lee <yee.lee@mediatek.com> | [[v2,1/1] mm: kfence: apply kmemleak_ignore_phys on early allocated pool](https://patchwork.kernel.org/project/linux-mm/patch/20220628113714.7792-2-yee.lee@mediatek.com/) | 654579 | v2 ☐☑ | [LORE v2,0/1](https://lore.kernel.org/r/20220628113714.7792-2-yee.lee@mediatek.com) |
| 2022/07/27 | Imran Khan <imran.f.khan@oracle.com> | [[RFC] mm/kfence: Introduce kernel parameter for selective usage of kfence.](https://patchwork.kernel.org/project/linux-mm/patch/20220727234241.1423357-1-imran.f.khan@oracle.com/) | 663583 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220727234241.1423357-1-imran.f.khan@oracle.com) |
| 2022/08/14 | Imran Khan <imran.f.khan@oracle.com> | [[v3] kfence: add sysfs interface to disable kfence for selected slabs.](https://patchwork.kernel.org/project/linux-mm/patch/20220814195353.2540848-1-imran.f.khan@oracle.com/) | 667448 | v3 ☐☑ | [LORE v3,0/1](https://lore.kernel.org/all/20220814195353.2540848-1-imran.f.khan@oracle.com) |


## 13.4 Debugging
-------

### 13.4.1 ptdump
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2014/11/26 | Russell King - ARM Linux | [ARM: add support to dump the kernel page tables](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20131024071600.GC16735@n2100.arm.linux.org.uk) | ARM 支持 PTDUMP | RFC ☑ 3.19-rc1 | [PatchWork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20131024071600.GC16735@n2100.arm.linux.org.uk), [LWN](https://lwn.net/Articles/572320) |
| 2014/11/26 | Laura Abbott | [arm64: add support to dump the kernel page tables](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1416961719-6644-1-git-send-email-lauraa@codeaurora.org) | ARM64 支持 PTDUMP | RFC ☑ 3.19-rc1 | [PatchWork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1416961719-6644-1-git-send-email-lauraa@codeaurora.org) |
| 2019/12/19 | Colin Cross | [Generic page walk and ptdump](https://lore.kernel.org/patchwork/patch/1169746) | 重构了 ARM64/X86_64 的 PTDUMP, 实现了通用的 PTDUMP 框架. 目前许多体系结构都有一个 debugfs 文件用于转储内核页表. 目前, 每个体系结构都必须为此实现自定义函数, 因为内核使用的页表的遍历细节在不同的体系结构之间是不同的. 本系列扩展了 walk_page_range()的功能, 使其能够处理内核的页表 (内核没有 vma, 可以包含比用户空间现有页面更大的页面). 通用的 PTDUMP 实现是利用 walk_page_range() 的新功能实现的, 最终 arm64 和 x86 将转而使用它, 删除了自定义的表行器. | v17 ☑ 5.6-rc1 |[PatchWork RFC](https://lore.kernel.org/patchwork/patch/1169746) |
| 2020/02/10 | Zong Li | [RISC-V page table dumper](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20131024071600.GC16735@n2100.arm.linux.org.uk) | ARM 支持 PTDUMP | RFC ☑ 3.19-rc1 | [LKML](https://lkml.org/lkml/2020/2/10/100) |
| 2021/08/02 | Gavin Shan <gshan@redhat.com> | [mm/debug_vm_pgtable: Enhancements](https://patchwork.kernel.org/project/linux-mm/cover/20210802060419.1360913-1-gshan@redhat.com) | 目前的实现存在一些问题, 本系列试图解决这些问题:<br>1. 所有需要的信息分散在变量中, 传递给各种测试函数. 代码以非常轻松的方式组织.<br>2. 在页面表条目修改测试期间, 页面没有从 buddy 分配. 该页面可能是无效的, 与 ARM64 上的 set_xxx_at() 的实现冲突. 访问目标页面, 以便在 ARM64 上授予执行权限时刷新 iCache.<br>3. 此外, 目标页面可以被取消映射, 访问它会导致内核崩溃.<br> 引入 "struct pgtable_debug_args" 来解决问题 (1).<br> 对于问题(2), 使用的页面是从页表条目修改测试中的 buddy 分配的. 如果我们没有分配(巨大的) 页面, 则跳过. 对于其他测试用例, 仍然使用到内核符号 (@start_kernel) 的原始页面. | RFC ☐ 5.14-rc4 | [PatchWork v5,00/12](https://patchwork.kernel.org/project/linux-mm/cover/20210802060419.1360913-1-gshan@redhat.com) |


### 13.4.2 Page Owner
-------

早在 2.6.11 的时代就通过 [Page owner tracking leak detector](https://lwn.net/Articles/121271) 给大家展示了页面所有者跟踪 [Page Owner](https://lwn.net/Articles/121656). 但是它一直停留在 Andrew 的源代码树上, 然而, 没有人试图 upstream 到 mainline, 虽然已经有不少公司使用这个特性来调试内存泄漏或寻找内存占用者. 最终在 v3.19, Joonsoo Kim 在一个重构中, 将这个特性推到主线.

这个功能帮助我们知道谁分配了页面. 当分配一个页面时, 我们将一些关于分配的信息存储在额外的内存中. 之后, 如果我们需要知道所有页面的状态, 我们可以从这些存储的信息中获取并分析它.

虽然我们已经有了跟踪页分配 / 空闲的跟踪点, 使用它来分析页面所有者是相当复杂的. 我们需要扩大跟踪缓冲区, 以防止重叠, 直到用户空间程序启动. 而且, 启动的程序不断地将跟踪缓冲区转储出来供以后分析, 它将更有可能改变系统行为, 而不是仅仅将其保存在内存中, 因此不利于调试.

此外, 我们还可以将 page_owner 特性用于各种目的. 例如, 我们可以借助 page_owner 实现的碎片统计以及一些 CMA 故障调试特性.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2013/11/24 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [Resurrect and use struct page extension for some debugging features](https://lore.kernel.org/patchwork/patch/520462) | 1. 引入 page_ext 作为 page 的扩展来存储调试相关的变量 <br>2. upstream page_owner | v7 ☑ 3.19-rc1 | [PatchWork v3](https://lore.kernel.org/patchwork/patch/520462), [Kernel Newbies](https://kernelnewbies.org/Linux_3.19#Memory_management) |
| 2020/12/10 | Liam Mark <lmark@codeaurora.org><br>Georgi Djakov <georgi.djakov@linaro.org> | [mm/page_owner: Record timestamp and pid](https://lore.kernel.org/patchwork/patch/1352029) | 收集页面所有者中记录的每个分配的时间和 PID, 以便可以测量分配 "激增" 及其来源. | v3 ☑ 5.11-rc1 | [PatchWork v3](https://lore.kernel.org/patchwork/patch/1352029), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9cc7e96aa846f9086431d6c2d33ff9ab42d72b2d) |
| 2021/02/03 | Georgi Djakov <georgi.djakov@linaro.org> | [mm/page_owner: Record the timestamp of all pages during free](https://lore.kernel.org/patchwork/patch/1375264) | 收集每个分配被释放的时间, 以帮助 kdump/ramdump 进行内存分析. 在 page_owner debugfs 文件中添加时间戳, 并将其打印到 dump_page() 中. 当我们释放页面时, 拥有另一个时间戳有助于调试页面迁移问题. 例如, 如果 alloc 时间戳和 free 时间戳相同, 则可能提示迁移内存存在问题, 而不是迁移过程中刚刚删除的页面. | v2 ☑ 5.13-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/patch/1375264), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=866b485262173a2b873386162b2ddcfbcb542b4a) |
| 2021/04/02 | Sergei Trofimovich <slyfox@gentoo.org> | [mm: page_owner: detect page_owner recursion via task_struct](https://lore.kernel.org/patchwork/patch/1406950) | 通过记录在 'struct task_struct' 中使用 page_owner 递归标志来防止递归. 在此之前防止递归是通过获取 backtrace 并检查当前的指令指针来检测 page_owner 递归, 这种方式效率是比较低的. 因为它需要额外的 backtrace, 以及对结果的线性堆栈扫描. | v1 ☑ 5.13-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/patch/1406950), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8e9b16c47680f6e7d6e5864a37f313f905a91cf5) |
| 2021/12/28 | Minchan Kim <minchan@kernel.org> | [mm: introduce page pin owner](https://patchwork.kernel.org/project/linux-mm/patch/20211228175904.3739751-1-minchan@kernel.org) | NA | RFC,v2 ☐ | [PatchWork RFC,v2](https://patchwork.kernel.org/project/linux-mm/patch/20211228175904.3739751-1-minchan@kernel.org) |
| 2022/02/08 | Waiman Long <longman@redhat.com> | [mm/page_owner: Extend page_owner to show memcg information](https://patchwork.kernel.org/project/linux-mm/cover/20220128195642.416743-1-longman@redhat.com) | 609625 | v1 ☐ | [PatchWork v1,0/2](https://lore.kernel.org/r/20220128195642.416743-1-longman@redhat.com)<br>*-*-*-*-*-*-*-* <br>[PatchWorkv2,0/3](https://lore.kernel.org/r/20220129205315.478628-1-longman@redhat.com)<br>*-*-*-*-*-*-*-* <br>[PatchWorkv3,0/4](https://lore.kernel.org/r/20220131192308.608837-1-longman@redhat.com)<br>*-*-*-*-*-*-*-* <br>[PatchWorkv4,0/4](https://lore.kernel.org/r/20220202203036.744010-1-longman@redhat.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v5,0/4](https://lore.kernel.org/r/20220208000532.1054311-1-longman@redhat.com) |
| 2022/02/19 | Yixuan Cao <caoyixuan2019@email.szu.edu.cn> | [mm/page_owner.c: record tgid](https://patchwork.kernel.org/project/linux-mm/patch/20220219180450.2399-1-caoyixuan2019@email.szu.edu.cn/) | 615993 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220219180450.2399-1-caoyixuan2019@email.szu.edu.cn) |
| 2022/03/22 | Yinan Zhang <zhangyinan2019@email.szu.edu.cn> | [[1/2] mm/page_owner.c: introduce vmalloc allocator for page_owner](https://patchwork.kernel.org/project/linux-mm/patch/20220322032225.1402992-1-zhangyinan2019@email.szu.edu.cn/) | 625342 | v1 ☐☑ | [LORE v1,0/2](https://lore.kernel.org/r/20220322032225.1402992-1-zhangyinan2019@email.szu.edu.cn) |
| 2022/08/24 | lizhe.67@bytedance.com <lizhe.67@bytedance.com> | [[v2] page_ext: introduce boot parameter 'early_page_ext'](https://patchwork.kernel.org/project/linux-mm/patch/20220824065058.81051-1-lizhe.67@bytedance.com/) | 670498 | v2 ☐☑ | [LORE v2,0/1](https://lore.kernel.org/r/20220824065058.81051-1-lizhe.67@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/1](https://lore.kernel.org/r/20220825063102.92307-1-lizhe.67@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/1](https://lore.kernel.org/r/20220825102714.669-1-lizhe.67@bytedance.com) |
| 2022/09/01 | Oscar Salvador <osalvador@suse.de> | [page_owner: print stacks and their counter](https://patchwork.kernel.org/project/linux-mm/cover/20220901044249.4624-1-osalvador@suse.de/) | 673059 | v1 ☐☑ | [LORE v1,0/3](https://lore.kernel.org/r/20220901044249.4624-1-osalvador@suse.de)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/3](https://lore.kernel.org/r/20221104093354.6218-1-osalvador@suse.de) |


### 13.4.3 PROC_PAGE_MONITOR
-------

[深入理解 Linux 内存回收 - Round One](https://zhuanlan.zhihu.com/p/320688306)

| 文件 | 描述 |
|:---:|:----:|
| `/proc/pid/smaps` | 使用 `/proc/pid/maps` 可以高效的确定映射的内存区域、跳过未映射的区域. |
| `/proc/kpagecount` | 这个文件包含 64 位计数 ,  表示每一页被映射的次数, 按照 PFN 值固定索引. |
| [`/proc/kpageflags`](https://www.kernel.org/doc/html/latest/admin-guide/mm/pagemap.html?highlight=kpageflags) | 此文件包含为 64 位的标志集, 表示该页的属性, 按照 PFN 索引. |

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/10/09 | Matt Mackall <mpm@selenic.com> | [maps4: pagemap monitoring v4](https://lore.kernel.org/patchwork/patch/95279) | 引入 CONFIG_PROC_PAGE_MONITOR, 管理了 `/proc/pid/clear_refs`, `/proc/pid/smaps`, `/proc/pid/pagemap`, `/proc/kpagecount`, `/proc/kpageflags` 多个接口. | v1 ☑ 2.6.25-rc1 | [PatchWork v4 0/12](https://lore.kernel.org/patchwork/patch/95279) |
| 201=09/05/08 | Joonsoo Kim <iamjoonsoo.kim@lge.com> | [export more page flags in /proc/kpageflags (take 6)](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=35efa5e993a7a00a50b87d2b7725c3eafc80b083) | 在 kpageflags 中导出了更多的 type. 同时新增了一个用户态工具 page-types 可以调试进程和内核的 page-types. 该工具后期[移到了 tools/vm 目录下](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c6dd897f3bfc54a44942d742d6dfa842e33d88e0) | v6 ☑ 3.4-rc1 | [PatchWork v3](https://lore.kernel.org/patchwork/patch/520462), [Kernel Newbies](https://kernelnewbies.org/Linux_3.19#Memory_management) |



### 13.4.5 vmstat
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/08/05 | Mel Gorman <mgorman@techsingularity.net> | [Protect vmstats on PREEMPT_RT](https://lore.kernel.org/patchwork/patch/1472709) | NA | v2 ☐ | [PatchWork 0/1,v2](https://patchwork.kernel.org/project/linux-mm/cover/20210723100034.13353-1-mgorman@techsingularity.net) |
| 2021/12/22 | Shakeel Butt <shakeelb@google.com> | [memcg: add per-memcg vmalloc stat](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4e5aa1f4c2b489bc6f3ab5ca54747b18a847289d) | NA | v1 ☑✓ 5.17-rc1 | [PatchWork v1](https://patchwork.kernel.org/project/linux-mm/patch/20211221215336.1922823-1-shakeelb@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://patchwork.kernel.org/project/linux-mm/patch/20211222052457.1960701-1-shakeelb@google.com) |


### 13.4.6 meminfo
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/12/16 | Qi Zheng <zhengqi.arch@bytedance.com> | [add MemAvailable to per-node meminfo](https://patchwork.kernel.org/project/linux-mm/cover/20211216124655.32247-1-zhengqi.arch@bytedance.com) | 在 `/proc/meminfo` 中, 展示了所有可用内存的总和显示为 "MemAvailable". 将相同的计数器也添加到 `/sys` 下的每个节点 `meminfo` 中. | v1 ☐ | [PatchWork 0/1,v2](https://patchwork.kernel.org/project/linux-mm/cover/20211216124655.32247-1-zhengqi.arch@bytedance.com/) |
| 2022/08/16 | Kefeng Wang <wangkefeng.wang@huawei.com> | [[RFC] mm, proc: add PcpFree to meminfo](https://patchwork.kernel.org/project/linux-mm/patch/20220816084426.135528-1-wangkefeng.wang@huawei.com/) | 667921 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220816084426.135528-1-wangkefeng.wang@huawei.com) |




### 13.4.7 DEBUG
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/02/03 | Christian Borntraeger <borntraeger@de.ibm.com> | [Optimize CONFIG_DEBUG_PAGEALLOC (x86 and s390)](https://damonitor.github.io) | 优化 CONFIG_DEBUG_PAGEALLOC, 提供了 debug_pagealloc_enabled(), 可以动态的开启 DEBUG_PAGEALLOC. | v4 ☑ 4.6-rc1 | [PatchWork v4,0/4](https://lore.kernel.org/patchwork/patch/642851) |


## 13.5 tracepoint
-------

内核提供的 MM 相关的 tracepoint, 介绍参见 Documentation/trace/events-kmem.rst, 使用和调试方法参见 Documentation/trace/tracepoint-analysis.rst.


### 13.5.1 内存分配路径 tracepoint
-------


| tracepoint | 提供版本 | 用途 | COMMIT |
|:----------:|:-------:|:---:|:------:|
| mm_page_alloc | v2.6.32 | NA | [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4b4f278c030aa4b6ee0915f396e9a9478d92d610) |
| mm_page_alloc | v2.6.32 | NA | [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4b4f278c030aa4b6ee0915f396e9a9478d92d610) |
| mm_page_free_direct/mm_page_free | v2.6.32,v3.3 | NA | [COMMIT1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4b4f278c030aa4b6ee0915f396e9a9478d92d610), [COMMIT2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=90a5d5af74f6570af063fb6bff33c6b2f8361bbc) |
| mm_pagevec_free/mm_page_free_batched | v2.6.32,v3.3 | NA | [COMMIT1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4b4f278c030aa4b6ee0915f396e9a9478d92d610), [COMMIT2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=90a5d5af74f6570af063fb6bff33c6b2f8361bbc) |
| mm_page_alloc_extfrag | v2.6.32 | 内核通过迁移类型分组来避免外碎片化. 在 `__rmqueue_smallest()` 无法获取指定 MIGRATETYPE 的情况下, 通过 `__rmqueue_fallback()` 来从其他 MIGRATETYPE 的空闲列表上窃取页面出来, 某些情况下甚至会更改整个 pageblock 的 MIGRATETYPE. 那么, 这个动作发生的次数越多, 碎片化就会越严重. TRACEPOINT mm_page_alloc_extfrag 用于跟踪 `__rmqueue_fallback()` 中页面用于跟踪窃取动作的发生以及其相关行为, 比如 order 的大小, 期望的 MIGRATETYPE 以及所使用的 MIGRATETYPE 列表, 更关键的是是否整个 pageblock 改变类型和这个事件是否对避免分裂与否是很重要的. 此信息可用于帮助分析碎片化, 并帮助决定是否应该增加 min_free_kbytes. |
| mm_page_alloc_zone_locked | v2.6.32 | 页面分配的 TRACEPOINT mm_page_alloc 只报告页面已成功分配, 但不指定它来自何处. 在分析性能时, 区分来自 PCP 还是 BUDDY 可能是很重要的, 因为后者需要持有 zone->lock, 并可能经历了页面窃取等诸多操作. 通过 TRACEPOINT mm_page_alloc_zone_locked 表明这个页面是从 BUDDY 中获取到的, 它经历了 LOCKED zone->lock. | [COMMIT1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0d3d062a6e289e065bd0aa537a6806a1806bf8aa),[COMMIT2](https://lkml.kernel.org/r/20201228132901.41523-1-carver4lio@163.com) |
| mm_page_pcpu_drain | v2.6.32 | 跟踪归还 PCP 的页面到伙伴系统的事件. | [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0d3d062a6e289e065bd0aa537a6806a1806bf8aa) |

针对页面分配器相关的 TRACEPOINT 事件添加了一个简单的后处理脚本 trace-pagealloc-postprocess.pl. 它可以用来指示谁是分配程序最密集的进程, 以及在跟踪期间执行区域锁定的频率. 参见 [commit c9d05cfc001f ("tracing, page-allocator: add a postprocessing script for page-allocator-related ftrace events")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c9d05cfc001fef3d6d37651e19ab9227a32b71f5)


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2009/08/10 | Mel Gorman <mel@csn.ul.ie> | [Add some trace events for the page allocator v6](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=8fbb398f5c78832ee61e0d5ed0793fa8857bd853) | NA | v6 ☑✓ 2.6.32-rc1 | [LORE v3,0/4](https://lore.kernel.org/all/1249409546-6343-1-git-send-email-mel@csn.ul.ie)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/6](https://lore.kernel.org/all/1249918915-16061-1-git-send-email-mel@csn.ul.ie) |
| 2011/11/11 | Konstantin Khlebnikov <khlebnikov@openvz.org> | [mm-tracepoint: rename page-free events](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=90a5d5af74f6570af063fb6bff33c6b2f8361bbc) | 从 [v2.6.33-5426-gc475dab](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c475dab63ae798d81fb597a6a1859986b296d9d0)内核为所有被释放的页面都添加了 tracepoint mm_page_free_direct, 而不仅仅是直接被释放的页面. 此外对于通过 page-list 释放的页面, 我们也会触发 mm_page_free_batch 事件. 所以, 对 free 相关的 tracepoint 的命名进行修正. 将 mm_page_free_direct 重命名为 mm_page_free, mm_pagevec_free 重命名为 mm_page_free_batch. | v3 ☑✓ 3.3-rc1 | [LORE v3,0/4](https://lore.kernel.org/all/20111111123957.7371.72792.stgit@zurg) |
| 2020/12/28 | Hailong liu <carver4lio@163.com> | [mm/page_alloc:add a missing mm_page_alloc_zone_locked tracepoint](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ce8f86ee94fabcc98537ddccd7e82cfd360a4dc5) | 20201228132901.41523-1-carver4lio@163.com | v1 ☐☑✓ | [LORE](https://lore.kernel.org/all/20201228132901.41523-1-carver4lio@163.com) |


### 13.5.2 LRU tracepoint
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2009/04/21 | Larry Woodman <lwoodman@redhat.com> | [mm tracepoints update](https://lwn.net/Articles/329577/) | 清理 mm 的 tracepoint, 以跟踪页面分配和释放、各种类型的页面错误和取消映射, 以及页面回收等路径. 这对在高内存压力下调试和分析内存分配问题和系统性能问题非常有用. | v1 ☐ | [LORE v1,0/9](https://lore.kernel.org/all/1283770053-18833-1-git-send-email-mel@csn.ul.ie)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/8](https://lore.kernel.org/lkml/1284553671-31574-1-git-send-email-mel@csn.ul.ie) |
| 2010/09/06 | Mel Gorman <mel@csn.ul.ie> | [tracing, vmscan: add trace events for LRU list shrinking](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e11da5b4fdf01d71d73c21cb92b00595b917d7fd) | [Reduce latencies and improve overall reclaim efficiency](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=08fc468f4eaf6683bae5bdb94743a09d8630cb80) 引入 lumpy_mode, 减少成块回收过程中的等待和延迟系列补丁集的其中一个补丁. 为了方便跟踪和计算扫描 nr_scanned / 回收 nr_scanned 比率, 作为页面回收所做工作量的度量. 在 shrink_inactive_list() 中添加了 tracepoint mm_vmscan_lru_shrink_inactive. | v2 ☑✓ 2.6.37-rc1 | [LORE v1,0/9](https://lore.kernel.org/all/1283770053-18833-1-git-send-email-mel@csn.ul.ie)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/8](https://lore.kernel.org/lkml/1284553671-31574-1-git-send-email-mel@csn.ul.ie) |
| 2010/06/14 | Mel Gorman <mel@csn.ul.ie> | [Avoid overflowing of stack during page reclaim V2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=1489fa14cb757b496c8fa2b63097dbcee6690695) | 这是两个补丁集的合并. 第一个减少了页面回收中的堆栈使用, 第二个在回收过程中写入连续页面, 并避免了直接回收器中的写回. 其中前几个补丁引入了一些 LRU 相关的 tracepoint, 可以用来评估在回收中发生了什么, 以及事情是变得更好还是更糟.<br>1. [commit1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=33906bc5c87b50028364405ec425de9638afc719) 添加了[直接回收](https://elixir.bootlin.com/linux/v2.6.36/source/mm/vmscan.c), [唤醒 KSWAPD](https://elixir.bootlin.com/linux/v2.6.36/source/mm/vmscan.c#L2403), 以及 [KSWAPD 睡眠](https://elixir.bootlin.com/linux/v2.6.36/source/mm/vmscan.c#L2380) 的 tracepoint.<br>2. [commit2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a8a94d151521b248727c1f88756174e15260815a) 添加了 [isolate_lru_pages() 的 tracepoint](https://elixir.bootlin.com/linux/v2.6.36/source/mm/vmscan.c#L1038).<br>3. [commit3](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=755f0225e8347b23a33ee6e3fb14a35310f95766) 中在 pageout 中添加了 [writepage 的 tracepoint](https://elixir.bootlin.com/linux/v2.6.36/source/mm/vmscan.c#L404).<br>4. [commit4](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b898cc70019ce1835bbf6c47bdf978adc36faa42) 为回收相关跟踪事件添加一个简单的[后处理脚本 trace-vmscan-postprocess.pl](https://elixir.bootlin.com/linux/v2.6.36/source/Documentation/trace/postprocess/trace-vmscan-postprocess.pl). 它可以用来指示 LRU 列表上有多少流量, 以及回收造成的延迟有多严重.<br> 之后的 6 个 commit 通过将较大的分配移出主调用路径, 减少了页面回收的堆栈占用空间. 如果 KSWAPD 要直接回写页面, 那么这是为了给文件系统提供尽可能多的堆栈. [补丁 11](https://lore.kernel.org/all/1276514273-27693-12-git-send-email-mel@csn.ul.ie) 在找到脏页面时将它们放在一个临时列表中, 然后使用一个助手函数将它们全部写出来. [补丁 12](https://lore.kernel.org/all/1276514273-27693-13-git-send-email-mel@csn.ul.ie) 完全阻止直接回收写出来的页面, 取而代之的是脏页面被放回 LRU. 但是需要注意的是补丁 11/12 未合入主线. | v1 ☑✓ 2.6.36-rc1 | [LORE v1,00/10](https://lore.kernel.org/lkml/1271352103-2280-1-git-send-email-mel@csn.ul.ie)<br>*-*-*-*-*-*-*-* <br>[LORE v2,00/12](https://lore.kernel.org/all/1276514273-27693-1-git-send-email-mel@csn.ul.ie) |
| 2013/05/17 | Mel Gorman <mgorman@suse.de> | [mm: add tracepoints for LRU activation and insertions](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c6286c983900c77410a951874f1589f4a41fbbae) | [Obey mark_page_accessed hint given by filesystems v3r1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=c53954a092d07c5684d31ea1fc813d262cff08a5) 补丁集的其中一个补丁. Alexey Lyahkov 和 Robin Dong 等最近 (v3.10 期间) 报告了 [诸多问题](https://www.spinics.net/lists/linux-ext4/msg37340.html), 这些问题可能是由于热页太快到达非活动列表的末尾并被回收造成的. 这个补丁集解决了这个问题. [当前补丁](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c6286c983900c77410a951874f1589f4a41fbbae) 为 LRU 页面激活和插入添加了两个跟踪点 mm_lru_insertion 和 mm_lru_activate. 使用这些 tracepoint, 可以在 LRU 中构建一个可以脱机处理的页面模型, 用于离线检查 LRU 上不同页面类型的平均使用时间(ms). | v3 ☑✓ 3.11-rc1 | [LORE RFC,0/3](https://lore.kernel.org/lkml/1367253119-6461-1-git-send-email-mgorman@suse.de)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/4](https://lore.kernel.org/lkml/1368440482-27909-1-git-send-email-mgorman@suse.de)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/5](https://lore.kernel.org/all/1368784087-956-1-git-send-email-mgorman@suse.de) |
| 2017/01/04 | Michal Hocko <mhocko@kernel.org> | [vm, vmscan: enahance vmscan tracepoints](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=93607e5a554a6408a1b56cdd146edc91e5da9983) | 在调试问题 [OOM: Better, but still there on 4.9](http://lkml.kernel.org/r/20161215225702.GA27944@boerne.fritz.box) 时, 作者意识到当前提供的跟踪点集还有一些改进的空间. 新增了 tracepoint [mm_vmscan_lru_shrink_active](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9d998b4f1e39abd69441d29a1ef3250514479267) 和 [mm_vmscan_inactive_list_is_low](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dcec0b60a8213aeb876823a15d834009fce3b36e). 原来 tracepoint 中[输出 LRU 的名字](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=32b3f2974adca13f8a4a610c396e88c6f81eb10e). 封装了 [reclaim_stat](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3c710c1ad11b4a856a396b181911568f3851a5d8), 并在 mm_vmscan_lru_shrink_inactive 中进行了输出. 同时将这些新增的改进同步到了 [后处理脚本 trace-vmscan-postprocess.pl](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=93607e5a554a6408a1b56cdd146edc91e5da9983) | v1 ☑✓ 4.11-rc1 | [LORE v1,0/7](https://lore.kernel.org/all/20161228153032.10821-1-mhocko@kernel.org)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/7](https://lore.kernel.org/all/20170104101942.4860-1-mhocko@kernel.org) |


### 13.5.3 其他 tracepoint
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/02/19 | Paul Gortmaker <paul.gortmaker@windriver.com> | [mmap_lock: add tracepoints around lock acquisition](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=abedf8e2419fb873d919dd74de2e84b510259339) | mmap lock tracepoint | v9 ☑ 4.6-rc1 | [LKML v8,0/5](https://lore.kernel.org/all/20201105211739.568279-1-axelrasmussen@google.com) |
| 2022/01/07 | Anshuman Khandual <anshuman.khandual@arm.com> | [mm/migration: Add trace events for THP migrations](https://patchwork.kernel.org/project/linux-mm/patch/1641531575-28524-1-git-send-email-anshuman.khandual@arm.com) | THP migrations tracepoint | v1 ☐ | [LORE v1](https://patchwork.kernel.org/project/linux-mm/patch/1641531575-28524-1-git-send-email-anshuman.khandual@arm.com) |




## 13.6 数据访问监视器 DAMON
-------

### 13.6.1 数据访问监视器 DAMON 概述
-------

[Software Visualizations to Analyze Memory Consumption: A Literature Review](https://dl.acm.org/doi/pdf/10.1145/3485134)

[linux data access monitor (DAMON)](https://blog.csdn.net/zqh1630/article/details/109954910)

[LWN: 用 DAMON 来优化 memory-management!](https://blog.csdn.net/Linux_Everything/article/details/104707923)

[知乎 - DAMON: Linux 内存数据访问监控框架](https://zhuanlan.zhihu.com/p/446677951)

[openEuler kernel SIG 2021-11-19 周例会](https://www.bilibili.com/video/BV1Ji4y1Z7o3) 上讲解了这个特性.

对指定的程序进行内存相关优化, 了解业务给定工作负载的数据访问模式至关重要. 但是, 从庞大和复杂的工作量中手动提取此类模式非常详尽. 更糟糕的是, 现有的内存访问分析工具会为不必要的详细分析结果带来不可接受的高开销.

内存管理在很大程度上基于预测 : 给定进程在不久的将来将需要哪些内存页?

不幸的是, 事实证明, 预测是困难的, 尤其是对于未来事件的预测. 在没有从未来发送回的有用信息的情况下, 内存管理子系统被迫依赖于对最近行为的观察, 并假设该行为可能会继续. 但是, 内核的内存管理决策对于用户空间是不透明的, 并且常常导致性能不佳. SeongJae Park 实现的 [DAMON](https://lwn.net/Articles/812707) 试图使内存使用模式对用户空间可见, 并让用户空间作为响应来更改内存管理决策.

亚马逊工程师公开了他们关于 ["DAMON"](https://github.com/awslabs/damo) 的工作, 这是一个 Linux 内核模块, 用于监控特定用户空间过程的数据访问.

[DAMON](https://damonitor.github.io/_index) 允许用户监控特定用户空间过程的实际内存访问模式. 从概念上讲, 它的操作很简单;

1.  首先, 将进程的地址空间划分为多个大小相等的区域.

2.  然后, 它监视对每个区域的访问, 并提供对每个区域的访问次数的直方图作为其输出.

由此, 该信息的使用者 (在用户空间或内核中) 可以请求更改以优化进程对内存的使用.

它的目标是 :

1.  足够准确, 可用于实际性能分析和优化;

2.  足够轻量, 以便它可以在线使用;

DAMON 利用两个核心机制 : ** 基于区域的采样 ** 和 ** 自适应区域调整 **, 允许用户将跟踪开销限制在有界范围内, 而与目标工作负载的大小和复杂性无关, 同时保留结果的质量.

| 机制 | 设计初衷 | 描述 |
|:---:|:-------:|:----:|
| Access Frequency Monitor<br> 访问频率监控 | DAMON 通过采样检测出给定的持续时间内哪些页面被频繁访问. 通过设置 sampling interval(damon_ctx.sample_interval) 和 aggregation interval(damon_ctx.aggr_interval) 来控制访问频率的分辨率.<br>1. 采样间隔, DAMON 在每个采样间隔内统计每个页面访问的次数;<br>2. 聚合间隔, DAMON 在每个聚合间隔后, 调用自定义的聚合函数读取聚合的结果. | 详细地说, DAMON 检查每个 sampling interval 对每个页面的访问并汇总结果. 换句话说, 计算对每个页面的访问次数. 在每个 aggregation interval 过去之后, DAMON 调用用户先前注册的回调函数, 以便用户可以读取聚合结果, 然后清除结果. 这种机制的监视开销将随目标工作负载的大小增加而任意增加.  |
| Region Based Sampling<br> 基于区域的采样 | 基于区域的采样允许用户在监控质量和开销之间做出自己的权衡, 并限制监控开销的上限. 为了避免无限制地增加开销, DAMON 将假定具有相同访问频率的多个相邻页面分组到一个区域中. 只有在这个前提下 (一个区域内的页面具有相同的访问频率), 只需要检查该区域里的一个页面即可. 因此, 在每个 sampling interval 中, DAMON 只需要在每个区域随机抽取一个页面并清除其访问位即可. 再经过一个 sampling interval 之后, DAMON 读取页面的访问位, 如果同时设置了该位, 则增加该区域的访问频率. 因此, 通过设置区域的数量, 可以控制监控开销. DAMON 允许用户设置区域的最小和最大数量来进行权衡. 但是, 如果不能保证这个假设, 这个方案就不能保证输出的质量. | 将具有相同访问频率的连续页面块分组到一个 region, 这样只需要从每个 region 内随机选取一个页面, 检查该页面的访问情况.<br> 通过设置 regions 的个数, 可以控制 DAMON 的开销, 用户可以设置最小和最大的 region 数量(minimum number of regions, and maximum number of regions). |
| Adaptive Regions Adjustment<br> 自适应的区域调整 | 自适应区域调整机制使 DAMON 在保持用户配置的权衡的同时, 最大限度地提高精度, 减少开销. 即使初始监控目标区域构造良好, 以满足假设 (相同区域中的页面具有相似的访问频率), 数据访问模式也可以动态更改. 这将导致低监测质量. 为了尽可能地保持这个假设, DAMON 根据每个区域的访问频率自适应地进行合并和分割. 每个 aggregation interval, 比较相邻区域的访问频率, 当频率差较小时进行合并. 然后, 在报告并清除每个区域的聚合访问频率之后, 如果区域总数不超过分割后用户指定的最大区域数, 则将每个区域分割为两到三个区域. 通过这种方式, DAMON 提供了最佳的质量和最小的开销, 同时为用户设置了相应的限制. | 为了保持 region 内的页面具有相似的访问频率, DAMON 会动态地合并该和切割 region. 每次聚合间隔, DAMON 比较相邻 region 的访问频率, 如果频率相差较小, 则进行合并, 减少的 region 数量可以减少开销. 如果总 region 数不超过用户配置的最大 region, 则为了提高精度, 可以对 region 进行拆分. |
| Dynamic Target Space Update Handling | 监视目标地址范围可以动态的变化. 例如, 虚拟内存可以动态映射和取消映射. 物理内存可能被热插拔. 由于在某些情况下更改可能非常频繁, 因此 DAMON 会检查动态内存映射更改, 并仅针对每个用户指定的时间间隔 (regions update interval) 将其应用于抽象的目标区域. |



作者 SeongJae Park
> github :https://github.com/sjp38/linux   /
> 后期切到了 git.kernel.org https://git.kernel.org/pub/scm/linux/kernel/git/sj/linux.git

主页 : [damonitor](https://github.com/damonitor)

使用这个框架, 可以更好的优化内核中关于内存回收和 THP(Transparent Huge Pages)等核心的内存管理机制, 从而实现更好的内存管理. 伴有很高开销的实验性内存管理优化工作可以再尝试一次. 与此同时, 在用户空间, 具有特殊工作负载的用户将能够编写个性化的工具或应用程序, 以便对其系统进行更深入的理解和特定的优化. 参见 [Monitoring Data Accesses » Optimization Guide](https://damonitor.github.io/doc/html/latest-damon/admin-guide/mm/damon/guide.html).


参见内核文档 [Linux Memory Management Documentation » DAMON: Data Access MONitor](https://www.kernel.org/doc/html/latest/mm/damon/index.html)

SeongJae Park 发布了 DAMON 2022 年度总结 [Looking back DAMON development in 2022](https://lore.kernel.org/lkml/20221229171209.162356-1-sj@kernel.org), 参见 phoronix 报道 [Amazon Reflects On The Great Year For DAMON In The Linux Kernel](https://www.phoronix.com/news/DAMON-Linux-2022)

### 13.6.2 DAMON Data Access MONitor
-------

内核文档 [Linux Memory Management Documentation » DAMON: Data Access MONitor » Design](https://www.kernel.org/doc/html/latest/mm/damon/design.html).


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/07/16 | SeongJae Park <sjpark@amazon.com>/<sj38.park@gmail.com> | [Introduce Data Access MONitor (DAMON)](https://damonitor.github.io) | 数据访问监视器 DAMON | v34 ☑ [5.15-rc1](https://kernelnewbies.org/LinuxChanges#Linux_5.15.DAMON.2C_a_data_access_monitor) | [PatchWork v24,00/14](https://lore.kernel.org/patchwork/patch/1375732), [LWN](https://lwn.net/Articles/1461471)<br>*-*-*-*-*-*-*-* <br>[PatchWork v34,00/13](https://patchwork.kernel.org/project/linux-mm/cover/20210716081449.22187-1-sj38.park@gmail.com) |
| 2021/08/31 | SeongJae Park <sjpark@amazon.com>/<sj38.park@gmail.com>/<sjpark@amazon.de> | [mm/damon/vaddr: Safely walk page table](https://patchwork.kernel.org/project/linux-mm/patch/20210831161800.29419-1-sj38.park@gmail.com) | 提交 linux-mm 的 [3f49584b262c ("mm/damon: implement primitives for the virtual memory address spaces")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3f49584b262c)  尝试使用 follow_invalidate_PTE() 查找任意虚拟地址的 PTE 或 PMD, 但没有正确锁定. 此提交通过在正确的锁定 (保持 mmap 读取锁定) 下使用另一个页面表遍历函数来解决此问题, 并使用了更通用的 walk_page_range(). | v1 ☐ | [PatchWork v2](https://patchwork.kernel.org/project/linux-mm/patch/20210831161800.29419-1-sj38.park@gmail.com) |
| 2021/09/17 | SeongJae Park <sjpark@amazon.com> | [mm, mm/damon: Trivial fixes](https://patchwork.kernel.org/project/linux-mm/cover/20210917123958.3819-1-sj@kernel.org) | DAMON 的 fix 补丁. | v1 ☐ | [PatchWork 0/5](https://patchwork.kernel.org/project/linux-mm/cover/20210917123958.3819-1-sj@kernel.org) |
| 2021/10/12 | SeongJae Park <sjpark@amazon.com> | [DAMON: Support Physical Memory Address Space Monitoring](https://www.phoronix.com/scan.php?page=news_item&px=DAMON-Physical-Monitoring) | 允许物理地址空间监控. | v1 ☑ 5.16-rc1 | [PatchWork RFC,v9,00/10](https://patchwork.kernel.org/project/linux-mm/cover/20201007071409.12174-1-sjpark@amazon.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC,v10,00/13](https://patchwork.kernel.org/project/linux-mm/cover/20201216094221.11898-1-sjpark@amazon.com))<br>*-*-*-*-*-*-*-* <br>[PatchWork 0/7](https://patchwork.kernel.org/project/linux-mm/cover/20211012205711.29216-1-sj@kernel.org)|
| 2021/10/13 | Xin Hao <xhao@linux.alibaba.com> | [mm/damon: Adjust the size of kbuf array to avoid overflow](https://patchwork.kernel.org/project/linux-mm/patch/20211013114854.15705-1-xhao@linux.alibaba.com) | NA | v1 ☐ | [PatchWork](https://patchwork.kernel.org/project/linux-mm/patch/20211013114854.15705-1-xhao@linux.alibaba.com) |
| 2021/10/16 | Xin Hao <xhao@linux.alibaba.com> | [mm/damon/core: Optimize kdamod.%d thread creation code](https://patchwork.kernel.org/project/linux-mm/patch/20211016165914.96049-1-xhao@linux.alibaba.com) | 当 ctx->adaptive_targets 列表为空, 无需创建并调用 kdamond. 只有当 ctx->adaptive_targets 列表不为空, 且 ctx->kdamond 指针为 NULL 时, 才调用__damon_start 函数. | v1 ☐ | [PatchWork v1](https://patchwork.kernel.org/project/linux-mm/patch/20211016165616.95849-1-xhao@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://patchwork.kernel.org/project/linux-mm/patch/20211016165914.96049-1-xhao@linux.alibaba.com) |
| 2021/10/27 | Changbin Du <changbin.du@gmail.com> | [mm/damon: simplify stop mechanism](https://patchwork.kernel.org/project/linux-mm/patch/20211027130517.4404-1-changbin.du@gmail.com) | NA | v2 ☐ | [PatchWork v2](https://patchwork.kernel.org/project/linux-mm/patch/20211027130517.4404-1-changbin.du@gmail.com) |
| 2022/02/15 | SeongJae Park <sj@kernel.org> | [Allow DAMON user code independent of monitoring primitives](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=851040566a008f7248cb754d5bb9a3e34f2effe5) | 614655 | v1 ☑ 5.18-rc1 | [LORE v1,0/8](https://lore.kernel.org/all/20220215184603.1479-1-sj@kernel.org) |
| 2022/02/16 | Xin Hao <xhao@linux.alibaba.com> | [mm/damon: Add NUMA access statistics function support](https://patchwork.kernel.org/project/linux-mm/cover/cover.1645024354.git.xhao@linux.alibaba.com/) | 614856 | v1 ☐☑ | [LORE v1,0/5](https://lore.kernel.org/r/cover.1645024354.git.xhao@linux.alibaba.com) |
| 2022/03/15 | Xin Hao <xhao@linux.alibaba.com> | [mm/damon: Add CMA minotor support](https://patchwork.kernel.org/project/linux-mm/cover/cover.1647378112.git.xhao@linux.alibaba.com/) | 为 DAMON 增加 CMA 内存监控功能. 在某些内存紧张的情况下, 通过监控 CMA 内存释放更多内存将是一个不错的选择. | v1 ☐☑ | [LORE v1,0/3](https://lore.kernel.org/r/cover.1647378112.git.xhao@linux.alibaba.com) |
| 2022/04/18 | Yuanchu Xie <yuanchu@google.com> | [[RESEND] selftests/damon: add damon to selftests root Makefile](https://patchwork.kernel.org/project/linux-mm/patch/20220418202017.3583638-1-yuanchu@google.com/) | 633136 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220418202017.3583638-1-yuanchu@google.com) |
| 2021/12/10 | SeongJae Park <sj@kernel.org> | [mm/damon/schemes: Extend stats for better online analysis and tuning](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=dbcb9b9f954f71fb46be34af624c9edaaa171414) | 20211210150016.35349-1-sj@kernel.org | v1 ☑✓ 5.17-rc1 | [LORE v1,0/6](https://lore.kernel.org/all/20211210150016.35349-1-sj@kernel.org) |
| 2022/01/14 | Baolin Wang <baolin.wang@linux.alibaba.com> | [mm/damon: add access checking for hugetlb pages](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=49f4203aae06ba9d67b500c90339b262b0a52637) | TODO | v1 ☐☑✓ | [LORE](https://lore.kernel.org/all/6afcbd1fda5f9c7c24f320d26a98188c727ceec3.1639623751.git.baolin.wang@linux.alibaba.com), [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=49f4203aae06ba9d67b500c90339b262b0a52637) |
| 2022/04/29 | SeongJae Park <sj@kernel.org> | [mm/damon: Support online tuning](https://patchwork.kernel.org/project/linux-mm/cover/20220429160606.127307-1-sj@kernel.org/) | 637059 | v1 ☐☑ | [LORE v1,0/14](https://lore.kernel.org/r/20220429160606.127307-1-sj@kernel.org) |
| 2022/05/07 | Gautam Menghani <gautammenghani201@gmail.com> | [Add documentation for Enum value'NR_DAMON_OPS'in](https://patchwork.kernel.org/project/linux-mm/patch/20220507165620.110706-1-gautammenghani201@gmail.com/) | 639422 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220507165620.110706-1-gautammenghani201@gmail.com) |
| 2022/10/19 | SeongJae Park <sj@kernel.org> | [efficiently expose damos action tried regions information](https://patchwork.kernel.org/project/linux-mm/cover/20221019001317.104270-1-sj@kernel.org/) | 686501 | v1 ☐☑ | [LORE v1,0/18](https://lore.kernel.org/r/20221019001317.104270-1-sj@kernel.org) |


### 13.6.3 DAMON Interface
-------

DAMON 最开始引入的时候, 为 DAMON 设计了 debugfs, 位于 `/sys/kernel/debug/damon` 目录下. 然而, 使 DMON 工具的使用和配置依赖于 debugfs, 是非常不合理的, DAMON 并不是只用于调试. 此外, debugfs 接口通过一个文件接收多个值. 例如, schemes 文件接收 18 个值. 结果, 它效率低下, 难以使用, 并且难以扩展. 特别是, 保持用户空间工具的向后兼容性越来越具有挑战性. 因此最好是实现另一个可靠和灵活的接口, 并逐步弃用 DAMON_DBGFS.

因此, v5.18 [Introduce DAMON sysfs interface](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=b18402726bd10e122c65eecc244ca1cdcb868cc8) 引入了基于 sysfs 的 DAMON 新用户界面. 新接口的思想是, 使用目录层次结构, 每个值都有一个专用文件. 通过 damo 等用户态程序, 可以很直观的使用 json 文件灵活地配置 DAMON. 按照作者的预期, debufs 接口将在下个 LTS 版本被废弃.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/10/08 | SeongJae Park <sjpark@amazon.com> | [mm/damon/dbgfs: Implement recording feature](https://patchwork.kernel.org/project/linux-mm/patch/20211008094509.16179-1-sj@kernel.org) | 为 'damon-dbgfs' 实现 'recording' 特性 <br> 用户空间可以通过 'damon_aggregate' 跟踪点事件获得监视结果. 为了简单起见, 跟踪点事件有一些重复的信息, 比如 'target_id' 和 'nr_regions'. 这造成它的大小比实际需要的要大. 另外, 对于一些简单的用例, 处理跟踪点可能会很复杂. 为了给用户空间提供一种更有效和简单的监控结果的方法, 这个提交实现了 'damon-dbgfs' 中的 'recording' 特性. 该特性通过一个名为 'record' 的新 debugfs 文件导出到用户空间, 该文件位于 '/damon/' 目录下. 该文件允许用户以简单格式在常规二进制文件中记录监视的访问模式. 记录的结果首先写入内存缓冲区并批处理刷新到文件中. 用户可以通过读取和写入 record 文件来获取和设置缓冲区的大小和结果文件的路径. | v1 ☐ | [PatchWork 1/4](https://patchwork.kernel.org/project/linux-mm/patch/20211008094509.16179-1-sj@kernel.org) |
| 2021/10/12 | Xin Hao <xhao@linux.alibaba.com> | [mm/damon/dbgfs: add region_stat interface](https://patchwork.kernel.org/project/linux-mm/patch/20211012054948.90381-1-xhao@linux.alibaba.com) | DAMON 中使用 damon-dbgfs 操作带来了很大的遍历, 有时候如果我希望能够查看任务的划分区域 nr_access 等值, 当前这不能直接通过 dbgfs 接口查看, 所以添加一个接口 "region_stat" 来显示. | v1 ☐ | [PatchWork](https://patchwork.kernel.org/project/linux-mm/patch/20211012054948.90381-1-xhao@linux.alibaba.com/) |
| 2021/10/21 | Xin Hao <xhao@linux.alibaba.com> | [mm/damon/dbgfs: Optimize target_ids interface write operation](https://patchwork.kernel.org/project/linux-mm/patch/bc341f48b5558f6816dcef22eca4f4a590efdc67.1634834628.git.xhao@linux.alibaba.com) | NA | v2 ☐ | [PatchWork v1](https://patchwork.kernel.org/project/linux-mm/patch/20211021085611.81211-1-xhao@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://patchwork.kernel.org/project/linux-mm/patch/bc341f48b5558f6816dcef22eca4f4a590efdc67.1634834628.git.xhao@linux.alibaba.com) |
| 2022/02/17 | SeongJae Park <sj@kernel.org> | [Introduce DAMON sysfs interface](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=b18402726bd10e122c65eecc244ca1cdcb868cc8) | 615483 | v1 ☑ 5.18-rc1 | [LORE v1,0/4](https://lore.kernel.org/r/20220217161938.8874-1-sj@kernel.org)<br>*-*-*-*-*-*-*-* <br>[LORE, v1,00/12](https://patchwork.kernel.org/project/linux-mm/cover/20220223152051.22936-1-sj@kernel.org) |
| 2021/01/07 | SeongJae Park <sjpark@amazon.com> | [tools/perf: Integrate DAMON in perf](https://lore.kernel.org/all/20210107120729.22328-1-sjpark@amazon.com) | perf 支持 DAMON 监控. | v1 ☐☑✓ | [LORE](https://lore.kernel.org/all/20210107120729.22328-1-sjpark@amazon.com) |


### 13.6.4 DAMOS(DAMON Operation Schemes)
-------

#### 13.6.4.1 DAMON Operation Schemes
-------

DAMON 使用各种试探法来确定哪些内存页处于活动使用状态, 并尽可能地利用这些信息来影响内存管理.

内存管理开发人员最希望的莫过于能够知道在不久的将来需要哪些内存页. 然后, 内核可以确保这些页面驻留在 RAM 中. 不幸的是, 当前的硬件无法提供该信息, 因此内存管理代码必须进行猜测. 通常, 最好的猜测是, 最近使用过的页面可能很快就会再次使用, 而那些已经一段时间未被触及的页面可能不需要.

LRU 列表是决定哪些页面保留在 RAM 中以及哪些页面被回收的机制的关键部分. 不过, 尽管它们的名字, 这些列表充其量只是对最近使用最少的页面的粗略近似. 最好将描述给出为 "最近至少注意到要使用". 如果有更好的机制来了解哪些页面真正被大量使用, 那么应该有可能利用这些信息来改进当前的 LRU 列表.

DAMON("Data Access MONitor") 就提供了这种机制, 通过一些聪明的算法, DAMON 试图更清楚地了解实际内存使用情况, 同时限制自己的 CPU 使用率. DAMON 的设计效率足以在生产系统上使用, 同时又足够准确, 可以改进内存管理决策.

5.16 内核增加了基于数据访问监控的操作方案 DAMOS ("DAMON operation schemes"), 它增加了一种基于规则的机制, 允许将定义的 madvise() 操作应用于在指定时间内具有特定访问频率的内存区域. 只要满足特定标准, 就可以采取行动. 比如, 可以将 DAMOS 配置为将过去 N 秒内未访问的区域通过 `madvise` 配置 MADV_COLD/MADV_PAGEOUT 等不同的策略, 此外还有其他各种 `madvise` 选项以及 stat 选项可用.

DAMOS 借助 debugfs 定制自己的监控处理方案. 例如, 配置 `echo "4096 8192 0 5 10 20 2" > schemes` 意味着如果大小为 [4KiB,  8KiB] 的内存区域显示 [0, 5] 中每个聚合间隔的访问数, 则应用 madvise MADV_PAGEOUT 操作,  它们都在 [`Documentation/admin-guide/mm/damon/usage.rst`](https://www.kernel.org/doc/html/latest/admin-guide/mm/damon/usage.html#schemes) 中有详细描述.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/10/01 | SeongJae Park <sjpark@amazon.com> | [Implement Data Access Monitoring-based Memory Operation Schemes](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=68536f8e01e571f553f78fa058ba543de3834452) | 实现了 DAMON 的基于数据访问监视的操作方案 (DAMOS).<br>DAMON 可以用作支持数据访问的内存管理优化的原语. 因此, 希望进行此类优化的用户应该运行 DAMON, 读取监视结果, 分析并优化内存管理方案.<br> 然而, 在许多其他情况下, 用户只是希望系统对具有特定时间的特定访问频率的特定大小的内存区域应用内存管理操作. 例如, "page out a memory region than 100mib only rare visits more than 2 minutes", 或者 "Do not use THP For a memory region than 2mib rarely visits more than 1 seconds". 为了使工作更容易且不冗余, 这个补丁集实现了 DAMON 的一个新特性, 称为基于数据访问监视的操作方案 (DAMOS). 使用该特性, 用户可以以简单的方式描述常规方案, 并要求 DAMON 自行执行这些方案.<br>DAMOS 对于内存管理优化是准确和有用的.<br>1. THP 的一个实验性的基于 DAMON 的操作方案 'ethp' 减少了 76.15% 的 THP 内存开销, 同时保留了 51.25% 的 THP 加速.<br>2. 另一个实验性的基于 damon 的 "主动回收" 实现 "prcl", 减少了 93.38% 的  residential sets 和 23.63% 的内存占用, 而在最好的情况下只产生 1.22% 的运行时开销 (parsec3/freqmine). | v1 ☑ 5.16-rc1 | [2020/12/16 PatchWork RFC,v15.1,0/8](https://patchwork.kernel.org/project/linux-mm/cover/20201216084404.23183-1-sjpark@amazon.com)<br>*-*-*-*-*-*-*-* <br>[2021/10/01 PatchWork 0/7](https://patchwork.kernel.org/project/linux-mm/cover/20211001125604.29660-1-sj@kernel.org) |
| 2021/12/21 | Changbin Du <changbin.du@gmail.com> | [Add a new scheme to support demotion on tiered memory system](https://patchwork.kernel.org/project/linux-mm/cover/cover.1640077468.git.baolin.wang@linux.alibaba.com) | 现在, 在具有不同内存类型的分级内存系统上, shrink_page_list() 中的回收路径已经支持将页面降级以减慢内存节点的速度, 而不是丢弃页面. 然而, 此时快速内存节点的内存水印已经紧张, 这将增加页面降级期间的内存分配延迟. 因此, 从用户空间主动降格冷页面的新方法将更有帮助. 我们可以依靠用户空间中的 DAMON 来帮助监控快速内存节点上的冷内存, 并主动降级冷页面来降低内存节点的速度, 以保持快速内存节点处于健康状态. 这个补丁集引入了一个名为 DAMOS_DEMOTE 的新模式来支持这个特性. | v2 ☐ | [LORE 0/2](https://lore.kernel.org/all/cover.1640077468.git.baolin.wang@linux.alibaba.com) |
| 2022/09/13 | haoxin <xhao@linux.alibaba.com> | [mm/damon: add MADV_COLLAPSE support in damos_action](https://patchwork.kernel.org/project/linux-mm/patch/20220913114735.22841-1-xhao@linux.alibaba.com/) | 676533 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220913114735.22841-1-xhao@linux.alibaba.com) |


#### 13.6.4.2 DAMON-based Reclamation
------

* Proactive Reclamation on top of DAMON/DAMOS

v5.16 基于 DAMOS 的 pageout scheme(DAMOS_PAGEOUT), 实现了主动回收 [Introduce DAMON-based Proactive Reclamation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=bec976b691437d056a92964cb7af07ee1a54221a) 机制, 通过 DAMON_RECLAIM 开启, 可以查询查找特定时间内没有被访问的内存区域(冷页), 主动将其回收或者 PAGE_OUT. 参见 phoronix 报道 [DAMON-Based Memory Reclamation Merged For Linux 5.16](https://www.phoronix.com/news/DAMON-Reclamation-Linux-5.16) 以及 LWN 报道 [Using DAMON for proactive reclaim](https://lwn.net/Articles/863753). 内核文档 [DAMON-based Reclamation](https://www.kernel.org/doc/html/latest/admin-guide/mm/damon/reclaim.html).

v6.0 内核包含朝这个方向迈出的另一步, 使 DAMON 能够主动对内核最近最少使用(LRU) 列表上的页面进行重新排序. 参见 LWN 报道 [LRU-list manipulation with DAMON](https://lwn.net/Articles/905370) 以及内核文档 [DAMON-based LRU-lists Sorting](https://www.kernel.org/doc/html/latest/admin-guide/mm/damon/lru_sort.html). 作者 SeongJae Park 称这种机制为 主动 LRU 列表排序 PLRUS(proactive LRU-list sorting), 他在补丁集的 cover-letter 中声称, 如果调整得当, 这种机制可以产生一些不错的结果: "PLRUS 在内存压力下内存 PSI some 减少了 10%, 主要页面故障减少了 14%, 加速了 3.74%".

PLRUS 为 DAMOS 添加了两个新操作: [DAMOS_LRU_PRIO](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8cdcc532268df0893d9756f537cbce479f4c4831) 和 [DAMOS_LRU_DEPRIO](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=99cdc2cd180a7adc87badc9ca92f8af803d8bf3b), 第一个将导致指示的页面移动到活动列表的头部, 使它们成为内核将尝试停用或回收的最后一个页面; 相反, 第二个将停用给定的页面, 导致它们被移动到非活动列表, 换句话说, 通过这种变化, DAMOS 深入内存管理子系统, 利用其探测到的高级信息使 LRU 列表的顺序更接近实际使用情况, 如果系统突然面临内存压力并且必须快速开始回收内存, 则此排序可能特别有用.

当然这些效果依赖于 DAMON 提供的有效的监测数据, 一般数据准确度越高, 对性能的影响也越大. DAMOS 提供了一套灵活的参数来提高 DAMON 的精度和限制 DAMOS 本身使用的 CPU 时间, 对于资深的管理员和开发者这提供了很大的灵活性, 他们了解内存管理的工作原理, 并且能够准确地测量更改的影响, 它还使完全破坏系统的性能变得容易. 同时为了帮助那些没有时间或技能的管理员为他们的工作负载提出最佳的 DAMOS 调优, Park 还添加了一个名为 [damon_lru_sort 的新内核模块](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=40e983cca9274e177bd5b9379299b44d9536ac68), 它使用 DAMOS 在一组 “保守” 参数下执行主动 LRU 列表排序, 这些参数旨在安全地提高性能, 同时最大限度地减少开销, 该模块将使使用 LRU 列表排序功能更容易, 但它仍然具有一组重要的参数, 可以从 [内核文档](https://www.kernel.org/doc/html/latest/admin-guide/mm/damon/lru_sort.html) 中查阅相关资料.

PLRUS 这一机制旨在解决与 MGLRU 工作类似的问题, MGLRU 也试图更准确地了解哪些页面正在使用中, 以便做出更好的页面替换决策, 关于如何处理 MGLRU 页面移动, 有许多悬而未决的问题; 有开发者建议使用加载 BPF 程序来控制这些决策, 但 DAMOS 也可能有所帮助, 这两种机制之间的整合目前尚不存在, 但可以添加可能是一件好事.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/10/19 | SeongJae Park <sjpark@amazon.com> | [Introduce DAMON-based Proactive Reclamation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=bec976b691437d056a92964cb7af07ee1a54221a) | 该补丁集基于 DAMOS 改进了用于生产质量的通用数据访问模式内存管理的引擎, 并在其之上实现了主动回收. | v4 ☑ [5.16-rc1](https://kernelnewbies.org/Linux_5.16#DAMON-based_proactive_memory_reclamation.2C_operation_schemes_and_physical_memory_monitoring) | [PatchWork RFC,00/13](https://patchwork.kernel.org/project/linux-mm/cover/20210720131309.22073-1-sj38.park@gmail.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC,v2,00/14](https://patchwork.kernel.org/project/linux-mm/patch/20210608115254.11930-15-sj38.park@gmail.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC,v3,00/15](https://patchwork.kernel.org/project/linux-mm/cover/20210720131309.22073-1-sj38.park@gmail.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v4 00/15](https://lore.kernel.org/all/20211019150731.16699-1-sj@kernel.org) |
| 2022/02/04 | Jonghyeon Kim <tome01@ajou.ac.kr> | [mm/damon: Rebase DAMON_RECALIM watermarks for NUMA nodes](https://patchwork.kernel.org/project/linux-mm/patch/20220204064059.6244-1-tome01@ajou.ac.kr/) | 611199 | v1 ☐☑ | [PatchWork v1,0/1](https://lore.kernel.org/r/20220204064059.6244-1-tome01@ajou.ac.kr) |
| 2022/02/18 | Jonghyeon Kim <tome01@ajou.ac.kr> | [Rebase DAMON_RECALIM for NUMA system](https://lore.kernel.org/all/20220218102611.31895-1-tome01@ajou.ac.kr) | 615730 | v1 ☐☑ | [LORE v1,0/3](https://lore.kernel.org/r/20220218102611.31895-1-tome01@ajou.ac.kr) |
| 2022/06/13 | SeongJae Park <sj@kernel.org> | [Extend DAMOS for Proactive LRU-lists Sorting](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=6acfcd0d75244178a4a101fe0da888fa3bff29fb) | [LRU-list manipulation with DAMON](https://lwn.net/Articles/905370) | v1 ☑✓ 6.0-rc1 | [2022/05/13 LORE RFC,0/3](https://lore.kernel.org/damon/20220513150000.25797-1-sj@kernel.org)<br>*-*-*-*-*-*-*-* <br>[2022/06/13 LORE v1,0/8](https://lore.kernel.org/all/20220613192301.8817-1-sj@kernel.org) |
| 2022/10/25 | SeongJae Park <sj@kernel.org> | [mm/damon/reclaim,lru_sort: enable/disable synchronously](https://patchwork.kernel.org/project/linux-mm/cover/20221025173650.90624-1-sj@kernel.org/) | 688752 | v1 ☐☑ | [LORE v1,0/4](https://lore.kernel.org/r/20221025173650.90624-1-sj@kernel.org) |

#### 13.6.4.3 DAMOS filtering
------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/11/24 | SeongJae Park <sj@kernel.org> | [implement DAMOS filtering for anon pages and](https://patchwork.kernel.org/project/linux-mm/cover/20221124212114.136863-1-sj@kernel.org/) | 698964 | v1 ☐☑ | [LORE v1,0/11](https://lore.kernel.org/r/20221124212114.136863-1-sj@kernel.org)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/11](https://lore.kernel.org/r/20221205230830.144349-1-sj@kernel.org) |


### 13.6.5 业界的使用
-------

OpenAnolis 龙蜥社区开源了 datop 工具, 在识别冷热内存以及跨 numa

B 站视频 [龙蜥开源工具 datop, 在识别冷热内存以及跨 numa 访存方面有什么样的优秀表现](https://www.bilibili.com/video/BV1Ra411t72z)

CSDN 宣传博客 [内存不超过 5M, datop 在识别冷热内存及跨 numa 访存有多硬核？| 龙蜥技术](https://blog.csdn.net/weixin_60347558/article/details/124585007)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/12/09 | Xin Hao <xhao@linux.alibaba.com> | [`#85` Introduce Data Access MONitor (DAMON)](https://gitee.com/anolis/cloud-kernel/pulls/85/commits) | 支持 NUMA 的能力, 同时改进了 tracepoint, 方便 dattop 解析. | [openanolis, devel-5.10, PR #85](https://gitee.com/anolis/cloud-kernel/pulls/85/commits) | [GITEE,PR](https://gitee.com/anolis/cloud-kernel/pulls/85/commits) |



# 14 杂项
-------


这是最后一章, 讲几个 Linux 内存管理方面的属于锦上添花性质的功能, 它们使 Linux 成为一个更强大, 更好用的操作系统.



## 14.1 KSM - 内存去重
-------


**2.6.32(2009 年 12 月发布)**

现代操作系统已经使用了不少共享内存的技术, 比如共享库, 创建新进程时子进程共享父进程地址空间. 而 KSM(Kernel SamePage Merging, 内存同页合并, 又称内存去重), 可以看作是存储领域去重 (de-duplication) 技术在内存使用上的延伸, 它是为了解决服务器虚拟化领域的内存去重方案. 想像在一个数据中心, 一台物理服务器上可能同时跑着多个虚拟客户机 (Guest OS), 并且这些虚拟机运行着很多相同的程序, 如果在物理内存上, 这些程序文本(text) 只有一份拷贝, 将会节省相当可观的内存. 而客户机间是相对独立的, 缺乏相互的认知, 所以 KSM 运作在监管机 (hypervisor) 上.



原理上简单地说, KSM 依赖一个内核线程, 定期地或可手动启动地, 扫描物理页面(通常稳定不修改的页面是合并的候选者, 比如包含执行程序的页面. 用户也可通过一个系统调用给予指导, 告知 KSM 进程的某部份区间适合合并, 见 [Kernel Samepage Merging](https://www.kernel.org/doc/html/latest/admin-guide/mm/ksm.html), 寻找相同的页面并合并, 多余的页面即可释放回系统另为它用. 而剩下的唯一的页面, 会被标为只读, 当有进程要写该页面, 该会为其分配新的页面.



值得一提的是, 在匹配相同页面时, 一种常规的算法是对页面进行哈希, 放入哈希列表, 用哈希值来进行匹配. 最开始 KSM 确定是用这种方法, 不过 VMWare 公司拥有跟该做法很相近的算法专利, 所以后来采用了另一种算法, 用红黑树代替哈希表, 把页面内容当成一个字符串来做内容比对, 以代替哈希比对. 由于在红黑树中也是以该 "字符串值" 大小作为键, 因此查找两个匹配的页面速度并不慢, 因为大部分比较只要比较开始若干位即可. 关于算法细节, 感兴趣者可以参考这两篇文章:

1.  [Linux 内存管理:  Linux Kernel Shared Memory 剖析 Linux 内核中的内存去耦合](https://www.cnblogs.com/ajian005/archive/2012/12/18/2841108.html).

2.  [KSM tries again](https://lwn.net/Articles/330589)


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/07/14 | Zhansaya Bagdauletkyzy <zhansayabagdaulet@gmail.com> | [add KSM selftests](https://lore.kernel.org/patchwork/patch/1459608) | 新增 KSM 相关的 selftest. | v1 ☑ NA | [PatchWork v2,0/4](https://patchwork.kernel.org/project/linux-mm/cover/cover.1626252248.git.zhansayabagdaulet@gmail.com) |
| 2021/08/19 | Zhansaya Bagdauletkyzy <zhansayabagdaulet@gmail.com> | [add KSM performance tests](https://lore.kernel.org/patchwork/patch/1470603) | 新增 KSM 性能相关的 selftest. | v1 ☑ NA | [PatchWork 0/2](https://patchwork.kernel.org/project/linux-mm/cover/cover.1627828548.git.zhansayabagdaulet@gmail.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v3,0/2](https://patchwork.kernel.org/project/linux-mm/cover/cover.1629386192.git.zhansayabagdaulet@gmail.com) |
| 2022/03/14 | Lv Ruyi <cgel.zte@gmail.com> | [ksm: Count ksm-merging pages for each process](https://patchwork.kernel.org/project/linux-mm/patch/20220314015355.2111696-1-xu.xin16@zte.com.cn/) | 由于当前 KSM 只计算 KSM 合并页面的数量(例如, ksm_pages_sharing 和 ksm_pages_shared), 无法看到更细粒度的 KSM 合并, 对于上层应用程序优化, 无法根据每个进程的 KSM 页面合并概率轻松设置合并区域. 因此, 有必要添加额外的统计手段, 以便上层用户能够了解每个进程的详细 KSM 合并信息.<br> 这个补丁在 proc 下添加了一个名为 ksm_merging_pages 的新文件, 以表示该进程中涉及的 KSM 合并页面. | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/all/20220314015355.2111696-1-xu.xin16@zte.com.cn)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/all/20220315114849.2119443-1-xu.xin16@zte.com.cn) |
| 2022/03/23 | CGEL <cgel.zte@gmail.com> | [mm/vmstat: add events for ksm cow](https://patchwork.kernel.org/project/linux-mm/patch/20220323031730.2342930-1-yang.yang29@zte.com.cn/) | 当用户想要节省内存时, 可以通过调用 madvise MADV_MERGEABLE 来使用 KSM, 但是这会导致 KSM COW 延迟. 用户可以通过读取 `/sys/kernel/mm/ksm/pages_sharing` 来了解 KSM 节省了多少内存, 但他们不知道 KSM COW 的成本, 这对于一些延迟敏感的任务来说是很重要的. 因此, 添加 KSM COW 事件 (KSM_COW_SUCCESS 和 KSM_COW_FAIL) 来帮助用户评估是否或如何使用 KSM. | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/all/20220323031730.2342930-1-yang.yang29@zte.com.cn)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/all/20220323075714.2345743-1-yang.yang29@zte.com.cn)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/1](https://lore.kernel.org/all/20220324104332.2350482-1-yang.yang29@zte.com.cn) |
| 2022/05/08 | CGEL <cgel.zte@gmail.com> | [mm/ksm: introduce ksm_force for each process](https://patchwork.kernel.org/project/linux-mm/patch/20220508091426.928094-1-xu.xin16@zte.com.cn) | 要使用 KSM, 我们必须在应用程序代码中显式调用 madvise(), 这意味着在操作系统上安装的应用程序需要卸载, 源代码需要修改. 很不方便. 为了改变这种情况, 我们在 `/proc/<pid>` 下添加一个 proc 文件 ksm_force, 以支持动态地打开 / 关闭进程的 MM 的 KSM 扫描.<br>1. 如果 ksm_force 设置为 1, 强制该 mm 的所有匿名和 "合格" 的 VMA 参与 KSM 扫描, 而不显式调用 madvise() 将 VMA 标记为 MADV_MERGEABLE. 但仅当"/sys/kernel/mm/ksm/run"的 klob 设置为 1 时有效.<br>2. 如果 ksm_force 设置为 0, 则取消该进程的 ksm_force 特性, 并取消属于 vma 的合并页面, 这些 vma 不再被合并, 但是依旧保留合并的 MADV_MERGEABLE 区域的行为. | v4 ☐☑ | [LORE v4,0/1](https://lore.kernel.org/r/20220508091426.928094-1-xu.xin16@zte.com.cn)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/1](https://lore.kernel.org/r/20220510122242.1380536-1-xu.xin16@zte.com.cn)<br>*-*-*-*-*-*-*-* <br>[LORE v7,0/1](https://lore.kernel.org/all/20220512070347.1628163-1-xu.xin16@zte.com.cn) |
| 2022/06/09 | CGEL <cgel.zte@gmail.com> | [mm/ksm: provide global_force to see maximum potential merging](https://patchwork.kernel.org/project/linux-mm/patch/20220609055658.703472-1-xu.xin16@zte.com.cn/) | 648704 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220609055658.703472-1-xu.xin16@zte.com.cn) |
| 2022/07/01 | CGEL <cgel.zte@gmail.com> | [[linux-next] mm/madvise: allow KSM hints for process_madvise](https://patchwork.kernel.org/project/linux-mm/patch/20220701084323.1261361-1-xu.xin16@zte.com.cn/) | 655733 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220701084323.1261361-1-xu.xin16@zte.com.cn) |
| 2022/08/12 | CGEL <cgel.zte@gmail.com> | [propose auto-run mode of ksm and its tests](https://patchwork.kernel.org/project/linux-mm/cover/20220812101102.41422-1-xu.xin16@zte.com.cn/) | 667133 | v2 ☐☑ | [LORE v2,0/5](https://lore.kernel.org/r/20220812101102.41422-1-xu.xin16@zte.com.cn) |
| 2022/08/22 | CGEL <cgel.zte@gmail.com> | [ksm: count allocated ksm rmap_items for each process](https://patchwork.kernel.org/project/linux-mm/patch/20220822053653.204150-1-xu.xin16@zte.com.cn/) | 669627 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220822053653.204150-1-xu.xin16@zte.com.cn) |
| 2022/08/24 | CGEL <cgel.zte@gmail.com> | [ksm: count allocated rmap_items and update documentation](https://patchwork.kernel.org/project/linux-mm/cover/20220824040036.215002-1-xu.xin16@zte.com.cn/) | 670470 | v2 ☐☑ | [LORE v2,0/2](https://lore.kernel.org/r/20220824040036.215002-1-xu.xin16@zte.com.cn)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/2](https://lore.kernel.org/r/20220830143731.299702-1-xu.xin16@zte.com.cn) |
| 2022/08/29 | Qi Zheng <zhengqi.arch@bytedance.com> | [add common struct mm_slot and use it in THP and KSM](https://patchwork.kernel.org/project/linux-mm/cover/20220829143055.41201-1-zhengqi.arch@bytedance.com) | 672053 | v1 ☐☑ | [LORE v1,0/7](https://lore.kernel.org/r/20220829143055.41201-1-zhengqi.arch@bytedance.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/7](https://lore.kernel.org/r/20220831031951.43152-1-zhengqi.arch@bytedance.com) |
| 2022/10/08 | xu.xin.sc@gmail.com <xu.xin.sc@gmail.com> | [ksm: support tracking KSM-placed zero-pages](https://lore.kernel.org/all/20221008070156.308465-1-xu.xin16@zte.com.cn) | 在使能 use_zero_pages 之前, ksm 的 pages_sharing 基本是准确的. 但是当启用 use_zero_pages 时, 所有与内核零页合并的空页都不会计算在 pages_sharing 或 pages_shared 中. 这是因为这些空页面被合并为零页面, KSM 不再管理这些页面, 这至少会导致两个问题: 1. MADV_UNMERGEABLE 和其他触发取消共享的方法将不会取消 KSM 放置的共享零页(这至少是违反 MADV_UNMERGEABLE 文档的), 参见[链接](https://lore.kernel.org/lkml/4a3daba6-18f9-d252-697c-197f65578c44@redhat.com). 2. 当启用 use_zero_pages 时, 我们无法知道有多少页是 KSM 放置的零页, 这导致 KSM 对所有实际合并的页面不透明. 通过这个补丁集我们可以精确地取消共享零页 (ksm - 放置) 并计算 ksm 零页. | v1 ☐☑✓ | [2022/10/08 LORE v1,0/5](https://lore.kernel.org/all/20221008070156.308465-1-xu.xin16@zte.com.cn)<br>*-*-*-*-*-*-*-* <br>[2022/10/11 LORE v3,0/5](https://lore.kernel.org/all/20221011022006.322158-1-xu.xin16@zte.com.cn)<br>*-*-*-*-*-*-*-* <br>[2022/12/26 LORE v4,0/6](https://lore.kernel.org/r/202212260959492929897@zte.com.cn)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/6](https://lore.kernel.org/r/202212300911327101708@zte.com.cn) |
| 2023/01/23 | Stefan Roesch <shr@devkernel.io> | [mm: process/cgroup ksm support](https://patchwork.kernel.org/project/linux-mm/cover/20230123173748.1734238-1-shr@devkernel.io/) | 到目前为止, KSM 只能通过为内存区域调用 madvise 来启用. 这组补丁允许在进程 / cgroup 级别启用 / 禁用 KSM. | v1 ☐☑ | [LORE v1,0/20](https://lore.kernel.org/r/20230123173748.1734238-1-shr@devkernel.io)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/19](https://lore.kernel.org/r/20230210215023.2740545-1-shr@devkernel.io)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/3](https://lore.kernel.org/r/20230224044000.3084046-1-shr@devkernel.io)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/3](https://lore.kernel.org/all/20230310182851.2579138-1-shr@devkernel.io) |
| 2023/02/10 | Stefan Roesch <shr@devkernel.io> | [[v1] mm: add tracepoints to ksm](https://patchwork.kernel.org/project/linux-mm/patch/20230210214645.2720847-1-shr@devkernel.io/) | 720845 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20230210214645.2720847-1-shr@devkernel.io) |
| 2023/02/27 | Stefan Roesch <shr@devkernel.io> | [[v1] prctl: add flags to enable KSM at the process level](https://patchwork.kernel.org/project/linux-mm/patch/20230227220206.436662-1-shr@devkernel.io/) | 725338 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20230227220206.436662-1-shr@devkernel.io) |


## 14.2 HWPoison - 内存页错误的处理
-------

**2.6.32(2009 年 12 月发布)**


[一开始想把这节放在第 12 章"** 内存管理调试支持 **"中, 不过后来觉得这并非用于主动调试的功能, 所以还是放在此章.]



随着内存颗粒密度的增大和内存大小的增加, 内存出错的概率也随之增大. 尤其是数据中心或云服务商, 系统内存大(几百 GB 甚至上 TB 级别), 又要提供高可靠的服务(RAS), 不能随随便便宕机; 然而, 内存出错时, 特别是出现多于 ECC(Error Correcting Codes) 内存条所支持的可修复 bit 位数的错误时, 此时硬件也爱莫能助, 将会触发一个 MCE(Machine Check Error) 异常, 而通常操作系统对于这种情况的做法就是 panic (操作系统选择 go die). 但是, 这种粗暴的做法显然是 over kill, 比如出错的页面是一个 ** 文件缓存页(page cache),** 那么操作系统完全可以把它废弃掉(随后它可以从后备文件系统重新把该页内容读出), 把该页隔离开来不用即是.



这种需求在 Intel 的 Xeon 处理器中得到实现. Intel Xeon 处理器引入了一个所谓 MCA(Machine Check Abort)架构, 它支持包括 ** 内存出错的的毒化(Poisoning)** 在内的硬件错误恢复机制. 当硬件检测到一个无法修复的内存错误时, 会把该数据标志为损坏(poisoned); 当之后该数据被读或消费时, 将会触发机器检查(Machine Check), 不同的时, 不再是简单地产生 MCE 异常, 而是调用操作系统定义的处理程序, 针对不同的情况进行细致的处理.



2.6.32 引入的 HWPoison 的 patch, 就是这个操作系统定义的处理程序, 它对错误数据的处理是以页为单位, 针对该错误页是匿名页还是文件缓存页, 是系统页还是进程页, 等等, 多种细致情况采取不同的措施. 关于此类细节, 可看此文章: [HWPOISON](https://lwn.net/Articles/348886)

[How to cope with hardware-poisoned page-cache pages](https://lwn.net/Articles/893565)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:---:|:---:|:----:|:---------:|:----:|
| 2022/06/30 | Naoya Horiguchi <naoya.horiguchi@linux.dev> | [mm, hwpoison: enable 1GB hugepage support (v3)](https://patchwork.kernel.org/project/linux-mm/cover/20220630022755.3362349-1-naoya.horiguchi@linux.dev/) | 655232 | v3 ☐☑ | [LORE v3,0/9](https://lore.kernel.org/r/20220630022755.3362349-1-naoya.horiguchi@linux.dev)[LORE v4,0/9](https://lore.kernel.org/r/20220704013312.2415700-1-naoya.horiguchi@linux.dev)<br>*-*-*-*-*-*-*-* <br>[LORE v7,0/8](https://lore.kernel.org/r/20220714042420.1847125-1-naoya.horiguchi@linux.dev) |

### 14.2.1  memory-failure
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2023/01/20 | Jiaqi Yan <jiaqiyan@google.com> | [Introduce per NUMA node memory error statistics](https://patchwork.kernel.org/project/linux-mm/cover/20230120034622.2698268-1-jiaqiyan@google.com/) | 713971 | v2 ☐☑ | [LORE v2,0/3](https://lore.kernel.org/r/20230120034622.2698268-1-jiaqiyan@google.com) |




### 14.2.2 hwpoison recovery
-------


OS 判断如果是在用户态触发这个硬件内存错误时, 处理方式是杀进程. 如果是在内核态触发这个硬件错误时, 处理方式是 kernel panic. 在内核态触发硬件内存错误的处理方式是无差别的 kernel panic, 我们可以分析是否存在场景可以无需 panic, 从而进一步提升系统可靠性, 比如由硬件内存错误引起的后果不是 fatal 的(仅仅影响相关的用户态进程, 那么我们通过杀掉这个用户态进程并且隔离出错的内存页面就可以), 那么我们就无需 kernel panic.

根据这个思路, 初步识别到如下场景下在触发硬件错误时的后果不是 fatal 的, 仅仅是影响了某个进程:

1. uaccess 用户态访问, 如 `copy_{from, to}_user`, `{get, put}_user`.

2. cow 写时复制.

3. dump_user_range().

以上的这些场景 openEuler 称之为 machine check safe 场景. 并实现 [CONFIG_ARM64_UCE_KERNEL_RECOVERY](https://gitee.com/openeuler/kernel/commit/dcd1c6a940ae973d3bd2c99fa77cf590f2e58ad8) 机制, 用于在这些场景下不用进行 PANIC, 而是只是杀掉应用线程, 并将对应的物理内存隔离处理.

同样 Intel 也提出了自己的解决方案, 如果内核由于写入时复制错误而正在复制页面, 并遇到无法纠正的错误, 那么 Linux 将崩溃, 因为它没有针对内核消耗毒物的情况的恢复代码. 建立测试用例很容易. 只需在 fork 时将一个错误注入私有页面 , 并让子进程写入该页面. 作者在 [ras-tools](git://git.kernel.org/pub/scm/linux/kernel/git/aegl/ras-tools.git) 中提供了一个测试用例, 只需启用 ACPI 错误注入并运行.<br>./einj_mem-uc-f<br> 添加一个新的 copy_user_highbage_mc() 函数, 该函数在可用的架构 (当前为 x86 和 powerpc) 上使用 copy_mc_to_kernel() . 当在页面复制过程中检测到错误时, 将 VM_FAULT_HWPOISON 返回给 wp_page_copy() 的调用方, 并在在调用堆栈中向上传播. x86 和 powerpc 的故障处理程序中都有代码, 通过向应用程序发送 SIGBUS 来处理这些代码. 从而避免系统崩溃, 并向触发写入时复制操作的进程发出信号. 它不会对仍在共享页中的内存错误采取任何操作. 要处理此问题, 需要调用 memory_failure() . 但这不能通过 wp_page_copy() 完成, 因为它包含 mmap_lock(). 在 Intel/x86 上, 由于内存控制器提供了内存中 h/w 中毒的额外通知, 因此通常会自动处理此松散端, 处理程序将调用 memory_failure(). 这不是 100% 的解决方案. 如果存在多个错误, 并非所有错误都可以以这种方式记录.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:---:|:---:|:----:|:---------:|:----:|
| 2022/10/21 | Luck, Tony <tony.luck@intel.com> | [Copy-on-write poison recovery](https://patchwork.kernel.org/project/linux-mm/cover/20221021200120.175753-1-tony.luck@intel.com/) | 687657 | v3 ☐☑ | [LORE RFC](https://lore.kernel.org/all/20221017234203.103666-1-tony.luck@intel.com)<br>*-*-*-*-*-*-*-* <br>[]()<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/2](https://lore.kernel.org/r/20221021200120.175753-1-tony.luck@intel.com) |
| 2021/03/18 | liubo <liubo254@huawei.com> | [arm64: ras: copy_from_user scenario support uce kernel recovery](https://gitee.com/openeuler/kernel/issues/I5GB28) | machine check safe 特性支持. | v1 ☐ 5.10 | [COMMIT](https://gitee.com/openeuler/kernel/commit/dcd1c6a940ae973d3bd2c99fa77cf590f2e58ad8) |
| 2022/12/19 | Tong Tiangen <tongtiangen@huawei.com> | [arm64: add machine check safe support](https://patchwork.kernel.org/project/linux-mm/cover/20221219120008.3818828-1-tongtiangen@huawei.com/) | 705578 | v8 ☐☑ | [LORE v8,0/4](https://lore.kernel.org/r/20221219120008.3818828-1-tongtiangen@huawei.com) |


## 14.3 Cross Memory Attach - 进程间快速消息传递
-------

**3.2(2012 年 1 月发布)**

这一节相对于其他本章内容是独立的. MPI(Message Passing Interface, 消息传递接口) [The Message Passing Interface (MPI) standard](https://www.mcs.anl.gov/research/projects/mpi) 是一个定义并行编程模型下用于进程间消息传递的一个高性能, 可扩展, 可移植的接口规范 (注意这只是一个标准, 有多个实现). 之前的 MPI 程序在进程间共享信息是用到共享内存(shared memory) 方式, 进程间的消息传递需要 2 次内存拷贝. 而 3.2 版本引入的 "Cross Memory Attach" 的 patch, 引入两个新的系统调用接口. 借用这两个接口, MPI 程序可以只使用一次拷贝, 从而提升性能.

## 14.4 OOM
-------

[Better tools for out-of-memory debugging](https://lwn.net/Articles/894546)

| 时间 | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:---:|:----:|:---:|:----:|:---------:|:----:|
| 2022/07/08 | Gang Li <ligang.bdlg@bytedance.com> | [mm, oom: Introduce per numa node oom for CONSTRAINT_{MEMORY_POLICY,CPUSET}](https://lore.kernel.org/all/20220708082129.80115-1-ligang.bdlg@bytedance.com) | 在本系列补丁之前, OOM 只会通过选择整个系统上错误率最高的进程来杀死内存使用率最高 (最富裕) 的进程. 这在 UMA 系统上运行良好, 但在 NUMA 系统上可能会发生一些意外死亡. 比如, 如果进程 c.out 绑定到 Node1, 并继续从 Node1 分配页面, 而 OOM-Killer 将首先选择 a.out, 但是杀死 a.out 并没有释放 Node1 上的任何内存, 所以 c.out 将被杀死. 如果 mempolicy 或 cpuset 有效, out_of_memory() 将选择特定节点上的受害者进行杀死. 这样内核就可以避免 NUMA 系统上的意外杀戮. | v2 ☐☑✓ | [LORE v2,0/5](https://lore.kernel.org/all/20220708082129.80115-1-ligang.bdlg@bytedance.com) |


## 14.5 能耗感知(EAMM)
-------

[Energy-Aware Memory Management for Embedded Multimedia Systems: A Computer-Aided Design Approach: 24](https://www.amazon.com.mx/Energy-Aware-Management-Embedded-Multimedia-Systems/dp/1439814007)

[HpMC: An Energy-aware Management System of Multi-level Memory Architectures](https://www.researchgate.net/publication/280580086_HpMC_An_Energy-aware_Management_System_of_Multi-level_Memory_Architectures)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2013/09/26 | Srivatsa S. Bhat <srivatsa.bhat@linux.vnet.ibm.com> | [mm: Memory Power Management](https://lore.kernel.org/all/52437128.7030402@linux.vnet.ibm.com) | 20130925231250.26184.31438.stgit@srivatsabhat.in.ibm.com | v4 ☐☑✓ | [LORE v2,00/15](https://lore.kernel.org/lkml/20130409214443.4500.44168.stgit@srivatsabhat.in.ibm.com)<br>*-*-*-*-*-*-*-* <br>[LORE RRC v4,0/40](https://lore.kernel.org/all/52437128.7030402@linux.vnet.ibm.com) |


## 14.6 PER_CPU
-------

[PERCPU 变量实现](https://zhuanlan.zhihu.com/p/260986194)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2009/02/18 | Tejun Heo <tj@kernel.org> | [implement new dynamic percpu allocator](https://lore.kernel.org/patchwork/patch/144750) | 实现了 [vm_area_register_early()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f0aa6617903648077dffe5cfcf7c4458f4610fa7) 以支持在启动阶段注册 vmap 区域. 基于此特性实现了可伸缩的动态 percpu 分配器 (CONFIG_HAVE_DYNAMIC_PER_CPU_ARE), 可用于静态(pcpu_setup_static/pcpu_setup_first_chunk) 和动态(`__alloc_percpu`) percpu 区域, 这将允许静态和动态区域共享更快的直接访问方法. | v1 ☑ 2.6.30-rc1 | [PatchWork](https://lore.kernel.org/patchwork/patch/144750) |
| 2009/02/24 | Tejun Heo <tj@kernel.org> | [percpu: fix pcpu_chunk_struct_size](https://lore.kernel.org/patchwork/patch/145272) | 1. 为对 static per_cpu(第一个 percpu 块)分配增加更多的自由度: 引入 PERCPU_DYNAMIC_RESERVE 用于预留 percpu 空间, 修改 pcpu_setup_static() 为 pcpu_setup_first_chunk(), 并让其更灵活.<br>2. 实现嵌入式的 percpu 分配器, 在 !NUMA 模式下, 可以简单地分配连续内存并将其用于第一个块, 而无需将其映射到 vmalloc 区域. 由于内存区域是由大页物理内存映射覆盖的, 所以它允许在基于可伸缩的动态 percpu 分配器分配静态 percpu 区域不引入任何 TLB 开销, 而且实现也非常简单.  | v1 ☑ 2.6.30-rc1 | [PatchWork](https://lore.kernel.org/patchwork/patch/145272) |
| 2009/03/26 | Tejun Heo <tj@kernel.org> | [percpu: clean up percpu constants](https://lore.kernel.org/patchwork/patch/146513) | NA | v1 ☑ 2.6.30-rc1 | [PatchWork](https://lore.kernel.org/patchwork/patch/146513) |
| 2009/07/21 | Tejun Heo <tj@kernel.org> | [percpu:  implement pcpu_get_vm_areas() and update embedding first chunk allocator](https://lore.kernel.org/patchwork/patch/164588) | NA | v1 ☑ 2.6.32-rc1 | [PatchWork](https://lore.kernel.org/patchwork/patch/164588) |
| 2008/11/18 | Tejun Heo <tj@kernel.org> | [Improve alloc_percpu: expose percpu_modalloc and percpu_modfree](https://lore.kernel.org/patchwork/patch/135207) | NA | v1 ☐ | [PatchWork 0/7](https://lore.kernel.org/patchwork/patch/135207) |
| 2021/09/10 | Kefeng Wang <wangkefeng.wang@huawei.com> | [arm64: support page mapping percpu first chunk allocator](https://lore.kernel.org/patchwork/patch/1464663) | NA | v2 ☐ v5.15-rc1 | [PatchWork v4,0/3](https://patchwork.kernel.org/project/linux-mm/cover/20210910053354.26721-1-wangkefeng.wang@huawei.com) |
| 2022/11/05 | Shakeel Butt <shakeelb@google.com> | [percpu_counter: add percpu_counter_sum_all interface](https://patchwork.kernel.org/project/linux-mm/patch/20221105014013.930636-1-shakeelb@google.com/) | 692315 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221105014013.930636-1-shakeelb@google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20221109012011.881058-1-shakeelb@google.com) |


## 14.7 ASLR
-------

### 14.7.1 ASLR(User Space)
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/01/24 | Topi Miettinen <toiwoton@gmail.com> | [mm: Optional full ASLR for mmap(), vdso, stack and heap](https://lore.kernel.org/patchwork/patch/1370134) | NA | v1 ☑ 2.6.30-rc1 | [PatchWork](https://lore.kernel.org/patchwork/patch/1370134) |
| 2021/01/24 | Topi Miettinen <toiwoton@gmail.com> | [mm: Optional full ASLR for mmap(), vdso, stack and heap](https://lore.kernel.org/patchwork/patch/1370134) | NA | v1 ☑ 2.6.30-rc1 | [PatchWork](https://lore.kernel.org/patchwork/patch/1370134) |


### 14.7.2 KASLR
-------

[Kernel Address Space Layout Randomization](https://www.phoronix.com/scan.php?page=search&q=Kernel%20Address%20Space%20Layout%20Randomization)


*   内核地址随机化 KASLR

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/08/13 | Xiongwei Song <sxwjean@gmail.com> | [Use generic code for randomization of virtual address of x86](https://patchwork.kernel.org/project/linux-mm/cover/20211011143150.318239-1-sxwjean@me.com) | ASLR 的重构. | v1 ☐ | [PatchWork v2,0/6](https://patchwork.kernel.org/project/linux-mm/cover/20211011143150.318239-1-sxwjean@me.com) |
| 2017/09/03 | Ard Biesheuvel <ard.biesheuvel@linaro.org> | [implement KASLR for ARM](https://lwn.net/Articles/732891) | ARM 支持 KASLR. | v1 ☐ | [LWN](https://lwn.net/Articles/732891)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2,0/6](https://git.kernel.org/pub/scm/linux/kernel/git/ardb/linux.git/log/?h=arm-kaslr-latest) |
| 2016/01/26 | Ard Biesheuvel <ard.biesheuvel@linaro.org> | [arm64: implement support for KASLR](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=2b5fe07a78a09a32002642b8a823428ade611f16) | 支持 ARM64 KASLR | v4 ☐☑✓ 4.6-rc1 | [LORE v4,0/22](https://lore.kernel.org/all/1453828249-14467-1-git-send-email-ard.biesheuvel@linaro.org) |
| 2022/12/27 | Liu Shixin <liushixin2@huawei.com> | [[RFC] arm64/vmalloc: use module region only for module_alloc() if CONFIG_RANDOMIZE_BASE is set](https://patchwork.kernel.org/project/linux-mm/patch/20221227092634.445212-1-liushixin2@huawei.com/) | 添加 10GB 设备后, 插入模块时 module_alloc 会失败. 如果设置了 CONFIG_RANDOMIZE_BASE, 则模块区域可以完全位于 vmalloc 区域中. 尽管如果设置了 ARM64_module_PLTS, module_alloc() 可以回退到 2GB 窗口, 但模块区域仍然很容易耗尽, 因为模块区域位于 vmalloc 区域的底部, 并且 vmalloc 区是从下到上分配的. 如果不是从 module_alloc() 调用, 则跳过模块区域. | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221227092634.445212-1-liushixin2@huawei.com) |
| 2023/02/15 | Alexandre Ghiti <alexghiti@rivosinc.com> | [riscv: Introduce KASLR](https://lore.kernel.org/all/20230215145113.465558-1-alexghiti@rivosinc.com) | [Linux Kernel Address Space Layout Randomization"KASLR"For RISC-V](https://www.phoronix.com/news/Linux-RISC-V-KASLR-Patches) | v1 ☐☑✓ | [LORE v1,0/4](https://lore.kernel.org/all/20230215145113.465558-1-alexghiti@rivosinc.com) |


*   随机函数偏移(FGKASLR)

[Intel Open-Source Developer Has Been Working On"FGKASLR"For Better Kernel Security](https://www.phoronix.com/scan.php?page=news_item&px=Intel-Linux-FGKASLR-Proposal)

[Linux 5.16 Has Early Preparations For Supporting FGKASLR](https://www.phoronix.com/scan.php?page=news_item&px=Linux-5.16-Preps-For-FGKASLR)

[FGKASLR Is An Exciting Linux Kernel Improvement To Look Forward To In 2022](https://www.phoronix.com/scan.php?page=news_item&px=Linux-FGKASLR-2022)

[FGKASLR Patches Revised A 10th Time For Improving Linux Kernel Security](https://www.phoronix.com/scan.php?page=news_item&px=FGKASLR-Linux-v10)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/10/13 | Kees Cook <keescook@chromium.org> | [x86: Various clean-ups in support of FGKASLR](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=ca136cac37eb51649d52d5bc4271c55e30ed354c) | FGKASLR 的一些的准备工作, 都是一些独立的改动. 比如:<br>1. 在 relocs 工具中支持 [超过 64K 的节头](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a54c401ae66fc78f3f0002938b3465ebd6379009).<br>2. KASLR 中通过  kaslr_get_random_long() 提取随机种子时候, 通过 earlyprintk debug_putstr 输出了很多无用的调试信息, 允许将[参数 purpose 置 NULL](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0d054d4e82072bcfd5eb961536b09a9b3f5613fb) 来使输出静默.<br>4. 修改 [ORC 查找表的大小]](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ca136cac37eb51649d52d5bc4271c55e30ed354c)以覆盖整个内核的代码段. | v1 ☑ [5.16-rc1](https://lkml.org/lkml/2021/11/2/26) | [LORE 0/4](https://lore.kernel.org/all/20211013175742.1197608-1-keescook@chromium.org) |
| 2020/07/17 | Topi Miettinen <toiwoton@gmail.com> | [Function Granular KASLR](https://patchwork.kernel.org/project/kernel-hardening/cover/20200717170008.5949-1-kristen@linux.intel.com/#23528683) | NA | v1 ☑ v4 | [PatchWork v4,00/10](https://patchwork.kernel.org/project/kernel-hardening/cover/20200717170008.5949-1-kristen@linux.intel.com/#23528683)<br>*-*-*-*-*-*-*-* <br>[LORE v9,00/15](https://lore.kernel.org/lkml/20211223002209.1092165-1-alexandr.lobakin@intel.com) |


### 14.7.3 Randomize Offset
-------


*   随机栈偏移

黑客们经常利用堆栈的信息, 窥测进程运行时的指令和数据. 为了保护堆栈数据, 增加攻击的难度, Linux 内核堆栈保护不断改进, 比如基于 vmap 的堆栈分配和保护页面(VMAP_STACK)、 删除 thread_info()、[STACKLEAK](https://a13xp0p0v.github.io/img/Alexander_Popov-stackleak-LinuxPiter2017.pdf), 攻击者必须找到新的方法来利用它们的攻击.

这里面很大一部分都是 Pax 的功劳, PaX 团队在内核安全方面进行了很多实践. [Documentation for the PaX project](https://pax.grsecurity.net/docs).


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/09/13 | Elena Reshetova <elena.reshetova@intel.com> | [thread_info cleanups and stack caching](https://lore.kernel.org/all/a0898196f0476195ca02713691a5037a14f2aac5.1473801993.git.luto@kernel.org) | 引入 CONFIG_THREAD_INFO_IN_TASK 将 thread_info 保存到 task_struct 中.<br> 之前 thread_info 与内核栈放在一起, 如果不慎踩了栈会同时破坏 thread_info, 极其不可靠. 因此把 thread_info 保存到 task_struct 中, 跟内核栈分离, 提高可靠性. | v1 ☑ 4.9-rc1 | [PatchWork](https://lore.kernel.org/kernel-hardening/20190329081358.30497-1-elena.reshetova@intel.com/) |
| 2017/12/06 | Alexander Popov <alex.popov@...ux.com> | [Introduce the STACKLEAK feature and a test for it](https://www.openwall.com/lists/kernel-hardening/2017/12/05/18) | STACKLEAK 是由 Grsecurity/PaX 开发的一种安全功能, 它:<br>1. 减少了内核堆栈泄漏 bug 可能泄露的信息;<br>2. 阻止一些未初始化的堆栈变量攻击(例如 CVE-2010-2963);<br>3. 引入一些内核堆栈溢出检测的运行时检查. | v6 ☑ 4.20-rc1 | [PatchWork v6,0/6](https://lwn.net/Articles/725287)<br>*-*-*-*-*-*-*-* <br>[PatchWork v6,0/6](https://www.openwall.com/lists/kernel-hardening/2017/12/05/19) |

最早 PaX 团队的 [RANDKSTACK 特性](https://pax.grsecurity.net/docs/randkstack.txt) 提出了内核栈随机偏移的最早思想. 这个特性旨在大大增加依赖确定性堆栈结构的各种基于堆栈的攻击的难度.


其主要思想是:

1.  由于堆栈偏移量在每次系统调用时都是随机的, 因此在执行攻击时, 攻击者很难可靠地落在线程堆栈上的任何特定位置.

2.  此外, 由于随机化是在确定的 pt_regs 之后执行的, 因此在长时间运行的系统调用期间, 不应该使用基于 ptrace 的方法来发现随机化偏移量.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2019/03/29 | Elena Reshetova <elena.reshetova@intel.com> | [x86/entry/64: randomize kernel stack offset upon syscall](https://lore.kernel.org/kernel-hardening/20190329081358.30497-1-elena.reshetova@intel.com) | 引入了 CONFIG_RANDOMIZE_KSTACK_OFFSET, 在 pt_regs 的固定位置之后, 系统调用的每个条目都会随机化内核堆栈偏移量. | v1 ☐ | [PatchWork](https://lore.kernel.org/kernel-hardening/20190329081358.30497-1-elena.reshetova@intel.com/) |
| 2021/04/06 | Kees Cook <keescook@chromium.org> | [Optionally randomize kernel stack offset each syscall](https://patchwork.kernel.org/project/linux-mm/cover/20200406231606.37619-1-keescook@chromium.org) | Elena 先前添加内核堆栈基偏移随机化的工作的延续和重构. | v4 ☐ | [PatchWork v3,0/5](https://patchwork.kernel.org/project/linux-mm/cover/20200406231606.37619-1-keescook@chromium.org) |
| 2021/08/13 | Kefeng Wang <wangkefeng.wang@huawei.com> | [riscv: Improve stack randomisation on RV64](https://www.phoronix.com/scan.php?page=news_item&px=RISC-V-Better-Stack-Rand) | NA | v1 ☑ 5.15-rc1 | [PatchWork](https://patchwork.kernel.org/project/linux-riscv/patch/20210812114702.44936-1-wangkefeng.wang@huawei.com), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d5935537c8256fc63c77d5f4914dfd6e3ef43241) |

### 14.7.4 Shadow stacks
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:-----:|:----:|:----:|:----:|:------------:|:----:|
| 2022/01/30 | Edgecombe, Rick P <rick.p.edgecombe@intel.com> | [Shadow stacks for userspace](https://patchwork.kernel.org/project/linux-mm/cover/20220130211838.8382-1-rick.p.edgecombe@intel.com) | 609893 | v1 ☐☑ | [PatchWork v1,0/35](https://lore.kernel.org/r/20220130211838.8382-1-rick.p.edgecombe@intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/39](https://lore.kernel.org/r/20220929222936.14584-1-rick.p.edgecombe@intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/37](https://lore.kernel.org/r/20221104223604.29615-1-rick.p.edgecombe@intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/39](https://lore.kernel.org/r/20221203003606.6838-1-rick.p.edgecombe@intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/39](https://lore.kernel.org/r/20230119212317.8324-1-rick.p.edgecombe@intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/41](https://lore.kernel.org/r/20230218211433.26859-1-rick.p.edgecombe@intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v7,0/41](https://lore.kernel.org/r/20230227222957.24501-1-rick.p.edgecombe@intel.com) |



## 14.8 页面迁移
-------

*   migration

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2005/11/01 | Christoph Lameter <clameter@sgi.com> | [Swap Migration V5: Overview](https://lwn.net/Articles/157066) | 交换迁移允许在进程运行时通过交换在 NUMA 系统中的节点之间移动页的物理位置.<br>1. 这组补丁不会自动迁移已移动的进程的内存, 相反, 它把迁移决策留给了用户空间. 因此并没有直接解决进程跨接点迁移后内存不能跟随迁移的问题, 但它确实试图建立了迁移的通用框架, 以便最终能够发展出完整的迁移解决方案.<br>2. 引入个新的系统调用 migrate_pages 尝试将属于给定进程的任何页面从 old_nodes 移动到 new_nodes.<br>3. 新的 MPOL_MF_MOVE 选项, 在 set_mempolicy() 系统调用中使用, 可用于相同的效果. | v5 ☑ 2.6.16-rc1 | [PatchWork v5,0/5](https://lore.kernel.org/patchwork/patch/45422) |
| 2006/01/10 | Christoph Lameter <clameter@sgi.com> | [Direct Migration V9: Overview](https://lore.kernel.org/patchwork/patch/49754) | NA | v9 ☑ 2.6.16-rc2 | [PatchWork v9,0/5](https://lore.kernel.org/patchwork/patch/49754) |
| 2011/12/14 | Mel Gorman <mgorman@suse.de> | [Reduce compaction-related stalls and improve asynchronous migration of dirty pages v6](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=0cee34fd72c582b4f8ad8ce00645b75fb4168199) | 使用 VFAT 的 U 盘在启用 THP 的情况下使用时会出现严重的暂停, 这一系列会减少暂停. 这是由于内存分配慢速路径下同步内存迁移 (回收规整时进行) 时进行脏页写入造成的. 这组补丁试图缓解因为内存规整或回收过多页面而导致用户可见的方式停滞, 从而进一步扩展 THP 的使用场景.  | v6 ☑✓ 3.3-rc1 | [LORE 0/5](https://lore.kernel.org/all/1321635524-8586-1-git-send-email-mgorman@suse.de)<br>*-*-*-*-*-*-*-* <br>[LORE RRC v4r2,0/7](https://lore.kernel.org/all/1321900608-27687-1-git-send-email-mgorman@suse.de)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/11](https://lore.kernel.org/all/1323877293-15401-1-git-send-email-mgorman@suse.de) |
| 2014/05/06 | David Rientjes <rientjes@google.com> | [mm, migration: add destination page freeing callback](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=aeef4b83806f49a0c454b7d4578671b71045bee2) | alpine.DEB.2.02.1405070336200.16568@chino.kir.corp.google.com | v4 ☑✓ 3.16-rc1 | [LORE 1/2](https://lore.kernel.org/all/alpine.DEB.2.02.1404301744110.8415@chino.kir.corp.google.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4,0/6](https://lore.kernel.org/all/alpine.DEB.2.02.1405061920470.18635@chino.kir.corp.google.com) |
| 2021/08/05 | Christoph Lameter <clameter@sgi.com> | [Some cleanup for page migration](https://lore.kernel.org/patchwork/patch/1472581) | NA | v1 ☐ | [PatchWork 0/5](https://lore.kernel.org/patchwork/patch/49754) |
| 2021/09/22 | John Hubbard <jhubbard@nvidia.com> | [mm/migrate: de-duplicate migrate_reason strings](https://patchwork.kernel.org/project/linux-mm/patch/20210922041755.141817-2-jhubbard@nvidia.com/) | NA | v1 ☐ | [PatchWork 0/5](https://patchwork.kernel.org/project/linux-mm/patch/20210922041755.141817-2-jhubbard@nvidia.com/) |
| 2021/11/03 | Baolin Wang <baolin.wang@linux.alibaba.com> | [Improve the migration stats](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=84b328aa81216e08804d8875d63f26bda1298788) | 根据与 [Zi Yan](https://lore.kernel.org/linux-mm/7E44019D-2A5D-4BA7-B4D5-00D4712F1687@nvidia.com) 的谈话, 这个补丁集改变了 migrate_pages() 的返回值, 以避免返回的数字大于用户通过 move_pages() 系统调用尝试迁移的页面数. 还修复了 trace_mm_compaction_migratepages() 中的 hugetlb 迁移统计和迁移统计. | v1 ☑✓ 5.17-rc1 | [PatchWork](https://lore.kernel.org/linux-mm/b35e54802a9a82d03d24845b463e9d9a68f7fd6b.1635491660.git.baolin.wang@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork RFC,0/3](https://patchwork.kernel.org/project/linux-mm/cover/cover.1635936218.git.baolin.wang@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/3](https://lore.kernel.org/all/cover.1636275127.git.baolin.wang@linux.alibaba.com) |
| 2021/11/11 | Baolin Wang <baolin.wang@linux.alibaba.com> | [Support multiple target nodes demotion](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ac16ec835314677dd7405dfb5a5e007c3ca424c7) | 对于同时有 1 个快速 (DRAM) 内存节点和多个慢速 (持久内存) 内存节点的系统, 根据当前节点降级策略不感知节点之间的距离, 速度和带宽等. 该补丁修改了 node_demotion 数据结构, 以支持多个目标节点, 并建立迁移路径以支持多个目标节点验证节点距离是否是最好的. | v1 ☑✓ 5.17-rc1 | [PatchWork v1](https://patchwork.kernel.org/project/linux-mm/patch/8850612186ea23eb5d328d84e4008a6b60f418e1.1636514506.git.baolin.wang@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2](https://patchwork.kernel.org/project/linux-mm/patch/f6d68800ff2efcb0720599ae092d30765a640232.1636428988.git.baolin.wang@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v2,0/2](hhttps://patchwork.kernel.org/project/linux-mm/cover/cover.1636616548.git.baolin.wang@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v3](https://patchwork.kernel.org/project/linux-mm/patch/a31dc065a7901bcdca0d9642d0def0f57e865e20.1636683991.git.baolin.wang@linux.alibaba.com)<br>*-*-*-*-*-*-*-* <br>[LORE v4](https://lore.kernel.org/all/00728da107789bb4ed9e0d28b1d08fd8056af2ef.1636697263.git.baolin.wang@linux.alibaba.com) |
| 2022/01/28 | Anshuman Khandual <anshuman.khandual@arm.com> | [mm/migration: Add trace events](https://patchwork.kernel.org/project/linux-mm/cover/1643368182-9588-1-git-send-email-anshuman.khandual@arm.com) | 增加进程迁移的 tracepoint. | v3 ☐ | [PatchWork V3,0/2](https://patchwork.kernel.org/project/linux-mm/cover/1643368182-9588-1-git-send-email-anshuman.khandual@arm.com) |


* numa balancing


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/09/22 | John Hubbard <jhubbard@nvidia.com> | [Mitosis: Transparently Self-Replicating Page-Tables for Large-Memory Machines](https://research.vmware.com/files/attachments/0/0/0/0/1/0/3/aspl0359a-achermanna.pdf) | 迁移的时候考虑迁移 page-table. | v1 ☐ | [GitHub](https://github.com/mitosis-project/mitosis-linux) |




## 14.9 [Free Page Reporting](https://www.kernel.org/doc/html/latest/vm/free_page_reporting.html)
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/02/11 | Alexander Duyck <alexander.h.duyck@linux.intel.com> | [mm / virtio: Provide support for free page reporting](https://patchwork.kernel.org/project/linux-mm/cover/20200211224416.29318.44077.stgit@localhost.localdomain) | 本系列提供了一种异步方法, 用于向虚拟机监控程序报告空闲的 guest 页面, 以便主机上的其他进程和 / 或 guest 可以删除和重用与这些页面关联的内存.<br> 使用此功能, 可以避免对磁盘进行不必要的 I/O, 并在主机内存过度使用的情况下大大提高性能.<br> 启用时, 将每 2 秒扫描一次可用内存, 同时释放足够高阶的页面. | v17 ☑ 5.7-rc1 | [PatchWork v17,0/9](https://lore.kernel.org/patchwork/patch/49754) |
| 2021/03/23 | Alexander Duyck <alexander.h.duyck@linux.intel.com> | [x86/Hyper-V: Support for free page reporting](https://lore.kernel.org/patchwork/patch/1401238) | Linux 现在支持虚拟化环境的免费页面报告(36e66c554b5c"mm: introduce Reported pages". 在 Hyper-V 上, 当配置了虚拟备份虚拟机时, Hyper-V 将在支持时公布冷内存丢弃功能. 此修补程序添加了对挂接免费页面报告基础结构的支持, 并利用 Hyper-V 冷内存丢弃提示 hypercall 将这些页面报告 / 释放回主机. | v5 ☑ 5.13-rc1 | [PatchWork v5](https://lore.kernel.org/patchwork/patch/1401238) |



## 14.10 PKRAM
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/05/07 | Anthony Yznaga <anthony.yznaga@oracle.com> | [PKRAM: Preserved-over-Kexec RAM](https://lore.kernel.org/patchwork/patch/856356) | NA | v11 ☑ 4.15-rc2 | [PatchWork RFC,00/43](https://lore.kernel.org/patchwork/patch/1237362) |


## 14.11 unaccepted memory
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/08/10 | Anthony Yznaga <anthony.yznaga@oracle.com> | [x86: Impplement support for unaccepted memory](https://patchwork.kernel.org/project/linux-mm/cover/20210810062626.1012-1-kirill.shutemov@linux.intel.com) | UEFI 规范 v2.9 引入了内存接受的概念, 一些虚拟机平台, 如 Intel TDX 或 AMD SEV-SNP, 要求在来宾使用内存之前先接受内存, 并通过特定于虚拟机平台的协议进行接受.<br> 接受内存成本很高, 这会使 VMM 为接受的来宾物理地址范围分配内存, 最好等到需要使用内存时再接受内存, 这可以减少启动时间并减少内存开销.<br> 支持这种内存需要对核心 mm 代码进行少量更改:<br>1. memblock 必须在分配时接受内存;<br>2. 页面分配器必须在第一次分配页面时接受内存;<br>3. Memblock 更改是微不足道的.<br>4. 页面分配器被修改为在第一次分配时接受页面.<br>5. PageOffline() 用于指示页面需要接受.<br>6. 热插拔和引出序号当前使用的标志, 这样的页面对页面分配器不可用.<br> 如果一个体系结构想要支持不可接受的内存, 它必须提供三个助手:<br>1. accept_memory() 使一系列物理地址被接受;<br>2. 如果页面需要接受, 则通过 maybe_set_page_offline() 将页面标记为 PageOffline(), 在引导期间用于将页面放在空闲列表上.<br>3. clear_page_offline() 清除使页面被接受并清除 PageOffline(). | v1 ☐ | [PatchWork 0/5](https://patchwork.kernel.org/project/linux-mm/cover/20210810062626.1012-1-kirill.shutemov@linux.intel.com)<br>*-*-*-*-*-*-*-* <br>[LORE v1,00/12](https://lore.kernel.org/r/20220425033934.68551-1-kirill.shutemov@linux.intel.com) |
| 2022/12/07 | Kirill A. Shutemov <kirill.shutemov@linux.intel.com> | [mm, x86/cc: Implement support for unaccepted memory](https://patchwork.kernel.org/project/linux-mm/cover/20221207014933.8435-1-kirill.shutemov@linux.intel.com/) | 702371 | v1 ☐☑ | [LORE v1,0/14](https://lore.kernel.org/r/20221207014933.8435-1-kirill.shutemov@linux.intel.com) |


## 14.12 USERCOPY
-------

如果内核开启 [CONFIG_HARDENED_USERCOPY](https://blog.csdn.net/tiantao2012/article/details/108997191), 则在从 user space copy 数据到 kernel space 时做一些检查, 如果不是有效数据, 则 assert. 可以通过设置 `hardened_usercopy=0|1` 来开启和关闭 usercopy.

[一例 hardened_usercopy 的 bug 解析](http://www.wowotech.net/linux_kenrel/480.html)


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/10/04 | Kees Cook <keescook@chromium.org> | [mm: Hardened usercopy](https://lore.kernel.org/all/1467843928-29351-1-git-send-email-keescook@chromium.org) | 将 PAX_USERCOPY 推到主线. PAX_USERCOPY 的设计是为了发现在使用 copy_to_user()/copy_from_user() 时存在的几类缺陷. | v1 ☑ 4.8-rc2 | [PatchWork 0/9](https://patchwork.kernel.org/project/linux-mm/cover/20211004224224.4137992-1-willy@infradead.org)<br>*-*-*-*-*-*-*-* <br>[LWN v2](https://lwn.net/Articles/691012/) |
| 2021/12/16 | "Matthew Wilcox (Oracle)" <willy@infradead.org> | [Assorted improvements to usercopy](https://lore.kernel.org/patchwork/patch/856356) | usercopy 的各种改进 | v1 ☐ | [2021/10/04 PatchWork 0/3](https://patchwork.kernel.org/project/linux-mm/cover/20211004224224.4137992-1-willy@infradead.org)<br>*-*-*-*-*-*-*-* <br>[2021/12/16 PatchWork v4,0/4](https://patchwork.kernel.org/project/linux-mm/cover/20211216215351.3811471-1-willy@infradead.org) |


## 14.13 ZONE_MOVABLE
-------

减少内存的碎片化是永恒的主题, 但是很多情况下事与愿违. 想象一下这个场景, 当我们需要一块大的连续内存时向伙伴系统申请, 虽然对应的内存区剩余内存还很多, 但是却发现对应的内存区由于碎片化过多, 并无法满足连续的内存申请需求, 那么此时是可以通过内存迁移来完成连续内存的申请, 但是这个过程并不一定能够成功, 因为中间往往有一些页面可能是不允许迁移的.

因此内核在 v2.6.23 引入 ZONE_MOVABLE 来减少内存碎片, 提高内存迁移的成功率. 内存管理子系统本身就把内存划分为不同 zone 来管理, ZONE_MOVABLE 与其他 zone 区域不同, 它是一个 pseudo zone, 即 (虚拟的) 伪的内存区域呢. 因此它实际上它是从平台中最高内存区中 (比如 `ZONE_HIHGMEM`) 单独划出了一部分内存, 作为 ZONE_MOVABLE.

ZONE_MOVABLE 的作用:


1.  引入 ZONE_MOVABLE 的主要目的是想要把 Non-Movable 和 Movable 的内存区分管理, 当我们划分出该区域后, 那么只有可迁移的页才能够从该区域申请, 这样当我们后面回收内存时, 针对该区域就都可以执行迁移, 而不用担心有一些不能被迁移的页面混在其中, 从而保证能够获取到足够大的连续内存.

2.  除了这个用途之外, 该区域还有一种使用场景, 那就是 memory hotplug 场景, 内存热插拔场景, 当我们对内存区执行 remove 时, 必须保证其中的内容都是可以被迁移走的, 因此热插拔的内存区必须位于 ZONE_MOVABLE 区域. 这在虚拟化等场景很有用.


>一般来说, 可迁移的页面不一定都在 ZONE_MOVABLE 中, 但是 ZONE_MOVABLE 中的页面必须尽可能都是可迁移的.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/07/17 |  Mel Gorman <mel@csn.ul.ie> | [Create the ZONE_MOVABLE zone](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=7e63efef857575320fb413fbc3d0ee704b72845f) | 614354 | v1 ☑ 2.6.23-rc1 | [CGIT v1,0/5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=7e63efef857575320fb413fbc3d0ee704b72845f) |


### 14.13.1 ZONE_MOVABLE
-------


2.6.23 的时候, Mel Gorman 提交补丁集 [reate ZONE_MOVABLE to partition memory between movable and non-movable pages](https://lwn.net/Articles/224255) 创建了一个名为 ZONE_MOVABLE 的分区. 引入这个 pseudo zone 的目的是为了防止 zone 内存碎片化. ZONE_MOVABLE 分区只能通过分配时指定了 `__GFP_HIGHMEM` 和 `__GFP_MOVABLE` 参数才能使用. 这样做的效果是将所有不可移动 (non-movable/unmovabled) 的页面都被限制在一个内存分区中, 而可移动 (movable) 页面的分配则由其他的内存区满足. 这组补丁可以与基于列表的抗碎片 (list-based anti-fragmentation) 补丁 (TODO - 待确认该补丁是指什么) 一起应用, 后者根据移动性将页面分组在一起.

ZONE_MOVABLE 区域的大小可通过启动参数 "kernelcore=" 来制定, 这个参数指定了多少内存分配给 ZONE_MOVALBE, 其余的用于 ZONE_MOVABLE. ZONE_MOVABLE 中的任何页面都可以通过迁移页面或回收页面来释放.

ZONE_MOVABLE 一个 pseudo zone, 它实际是从内核划分的某个 zone 中单独划出了一部分内存来是使用的, 因此当我们选择某一个 zone 的页面用于 ZONE_MOVABLE 来使用时, 需要考虑两点:

1.  首先, 仅仅位于最高内存 zone 的内存可用于 ZONE_MOVABLE. 对于 32 位的平台(比如 x86, arm 等), 是 ZONE_HIGHMEM. 对于 PPC64 则是 ZONE_DMA; 对于 x86_64 则是 ZONE_DMA32.

2.  其次, 内核可用的内存数量将在可能的情况下均匀分布在 NUMA 节点上. 如果节点的大小不一, 就会导致内核在不同节点上可使用的内存数量不同.

内核启动的时候会通过 [find_usable_zone_for_movable()](https://elixir.bootlin.com/linux/v2.6.23/source/mm/page_alloc.c#L2697) 查找一块合适的 zone index 供 ZONE_MOVABLE 使用.

默认情况下, 这个 zone 不会被用作 hugetlb 的分配, 因为 hugetlb 的分配是固定的且不可迁移的 (至少当时 2.6.23 版本是这样). 但是, 由于 ZONE_MOVABLE 始终具有可以迁移或回收的页面, 因此即使系统已运行很长时间, 也可以在 ZONE_MOVABLE 区域轻松地满足庞大的连续页面(比如 hugepage 等) 的分配. 这更将允许管理员根据区域的大小在运行时调整 hugepage 池的大小. 因此内核仍然通过补丁集中的 [commit 396faf0303d2 ("Allow huge page allocations to use GFP_HIGH_MOVABLE")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=396faf0303d273219db5d7eb4a2879ad977ed185) 提供了一个 [sysctl hugepages_treat_as_movable](https://elixir.bootlin.com/linux/v2.6.23/source/mm/hugetlb.c#L31), 允许在 ZONE_MOVABLE 上进行 hugetlb 的分配. 这意味着, 如果页面没有被 mlock 锁定, 那么在系统的生命周期内, 可以将 hugetlb 的页面池调整为 ZONE_MOVABLE 的大小(huge page pool 的尺寸包含了 ZONE_MOVABLE 的尺寸). 尽管大页是不可移动的, 但是内核开发者们认为这不会引入额外的外部内存碎片, 因为大页不正是我们所关心的最大的连续块的分配请求么.

因此除了 hughtlb 可以是 non-movable, Mel Gorman 不再引入其他可能会导致 ZONE_MOVABLE 外碎片的分配.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/01/25 | Mel Gorman <mel@csn.ul.ie> | [Create ZONE_MOVABLE to partition memory between movable and non-movable pages](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=7e63efef857575320fb413fbc3d0ee704b72845f) | NA | v2 ☑ 2.6.23-rc1 |  [2007/01/25 LORE v1,0/8](https://lore.kernel.org/lkml/20070125234458.28809.5412.sendpatchset@skynet.skynet.ie)<br>*-*-*-*-*-*-*-* <br>[2007/03/01 LORE v2,0/8](https://lore.kernel.org/lkml/20070301100802.30048.45045.sendpatchset@skynet.skynet.ie), [关键 COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ed7ed365172e27b0efe9d43cc962723c7193e34e) |

> 内核通过将已经将内存划分为许多具有不同特征的 ZONE, 来防止出现较严重的内存碎片, 特别是通过引入 ZONE_MOVABLE 专区, 通过隔离可移动和不可移动的分配请求 (ZONE_MOVABLE 中区域, 只满足应该可以移动页面(或者简单地回收它们) 来根据需要创建更大的区域).

理想很美好, 现实很残酷, 仍然会有一些情况会破坏 ZONE_MOVABLE 想要的美好. 经常会有一些零碎的、位置不好的、当前无法被移动移动的页面固定在 ZONE_MOVABLE 中, 而导致 ZONE_MOVABLE 的整块内存无法被移动. 参见 [Preserving the mobility of ZONE_MOVABLE](https://lwn.net/Articles/843326)

例如, 在设备执行 I/O 时就往往会 PIN 住一些页面, 此时移动页面往往会产生不幸的副作用. 通常, 这种固定持续时间很短, 往往不是一个大问题. 不过, 有时候这种固定可以持续很长时间ーー例如, 数周. 用于 RDMA 的内存缓冲区常常牵涉到这种反社会行为, 但是也有其他的例子. 当然, 一个实际上连续的用户空间缓冲区可能广泛分散在物理内存中, 因此长时间固定其中一个缓冲区可能会产生同样广泛分散的问题.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2013/01/14 | Tang Chen <tangchen@cn.fujitsu.com> | [Add movablecore_map boot option](https://lore.kernel.org/all/1358154925-21537-1-git-send-email-tangchen@cn.fujitsu.com) | 1358154925-21537-1-git-send-email-tangchen@cn.fujitsu.com | v5 ☐☑✓ | [LORE v5,0/5](https://lore.kernel.org/all/1358154925-21537-1-git-send-email-tangchen@cn.fujitsu.com) |
| 2021/02/15 | Pavel Tatashin <pasha.tatashin@soleen.com> | [prohibit pinning pages in ZONE_MOVABLE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=e44605a8b1aa13d892addc59ec3d416cb186c77b) | 参见 [Preserving the mobility of ZONE_MOVABLE](https://lwn.net/Articles/843326) | v11 ☑ 5.13-rc1 | [LWN v7 00/14](https://lwn.net/ml/linux-kernel/20210122033748.924330-1-pasha.tatashin@soleen.com/), [LORE v7,0/4](https://lore.kernel.org/lkml/20210122033748.924330-1-pasha.tatashin@soleen.com)<br>*-*-*-*-*-*-*-* <br>[LORE v11,00/14](https://lore.kernel.org/all/20210215161349.246722-1-pasha.tatashin@soleen.com), [关键 COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d1e153fea2a8940273174fc17733c44323d35cd5) |
| 2022/08/18 | Wupeng Ma <mawupeng1@huawei.com> | [watermark related improvement on zone movable](https://patchwork.kernel.org/project/linux-mm/cover/20220818090430.2859992-1-mawupeng1@huawei.com) | 668702 | v1 ☐☑ | [LORE v1,0/2](https://lore.kernel.org/r/20220818090430.2859992-1-mawupeng1@huawei.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/2](https://lore.kernel.org/r/20220905032858.1462927-1-mawupeng1@huawei.com) |


### 14.13.2 配置参数
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/07/17  | Mel Gorman <mel@csn.ul.ie> | [Add a movablecore= parameter for sizing ZONE_MOVABLE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7e63efef857575320fb413fbc3d0ee704b72845f) | NA | v1 ☑ 2.6.23-rc1 | [LWN v7,00/14](https://lwn.net/ml/linux-kernel/20210122033748.924330-1-pasha.tatashin@soleen.com/), [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7e63efef857575320fb413fbc3d0ee704b72845f) |
| 2013/01/14 | Tang Chen <tangchen@cn.fujitsu.com> | [Add movablecore_map boot option](https://lore.kernel.org/all/1358154925-21537-1-git-send-email-tangchen@cn.fujitsu.com) | 1358154925-21537-1-git-send-email-tangchen@cn.fujitsu.com | v5 ☑✓ 3.9-rc1 | [LORE v5,0/5](https://lore.kernel.org/all/1358154925-21537-1-git-send-email-tangchen@cn.fujitsu.com) |
| 2016/01/08 | Taku Izumi <izumi.taku@jp.fujitsu.com> | [handle kernelcore=: generic](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ed7ed365172e27b0efe9d43cc962723c7193e34e) | 之前 kernelcore 启动参数的处理是架构相关的, 可能导致某些架构出现问题. 这个补丁使用通用代码来处理 `kernelcore` 参数. 适用于支持独立于 arch 的区域大小调整(即定义 CONFIG_ARCH_POPULATES_NODE_MAP) 的架构, 其他架构将忽略引导参数. | v1 ☑ 2.6.23-rc1 | [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ed7ed365172e27b0efe9d43cc962723c7193e34e) |
| 2018/04/02 | Dou Liyang | [x86/boot/KASLR: Extend movable_node option for KASLR](https://lkml.org/lkml/2018/4/2/600) | NA | v1 ☐ | [LKML v5,0/5](https://lkml.org/lkml/2018/4/2/600) |



### 14.13.3 内存镜像
-------

内核 v4.6 通过扩展现有的 "kernelcore" 选项, 增加 "kernelcore=mirror" 选项, 引入了内存可靠性分级的功能. 通过 BIOS 上报镜像内存 (可靠内存) 的地址范围等信息, 区分镜像内存(可靠内存) 和非镜像内存(不可靠内存).

当前的实现比较简单:

1. 从镜像区域分配内核内存.

2. 从非镜像区域分配用户内存. 通过将非镜像范围安排到可移动区域(ZONE_MOVABLE).


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2016/01/08 | Taku Izumi <izumi.taku@jp.fujitsu.com> | [mm: Introduce kernelcore=mirror option](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=342332e6a925e9ed015e5465062c38d2b86ec8f9) | 内存镜像, 提供内存可靠性分级的功能.<br> 通过 UEFI BIOS(规范 2.5) 上报镜像内存 (即高可靠内存 mirrored/reliable) 的范围和非镜像内存 (即低可靠内存) 的两种内存. 内核默认使用高可靠的内存, 用 NORMAL ZONE 管理, 用户态优先使用普通 (低可靠的) 内存, 使用 MOVABLE ZONE 管理. | v4 ☑ 4.6-rc1 | [LKML RFC](https://lkml.org/lkml/2015/10/9/24)<br>*-*-*-*-*-*-*-* <br>[LKML v1](https://lkml.org/lkml/2015/10/15/9)<br>*-*-*-*-*-*-*-* <br>[LKML v2,0/2](https://lkml.org/lkml/2015/11/27/18)<br>*-*-*-*-*-*-*-* <br>[LKML v3,0/2](https://lkml.org/lkml/2015/12/8/836)<br>*-*-*-*-*-*-*-* <br>[LKML v4,0/2](https://lkml.org/lkml/2016/1/8/88), [PatchWork v4,0/2](https://lore.kernel.org/lkml/1452241523-19559-1-git-send-email-izumi.taku@jp.fujitsu.com) |
| 2016/06/27 | Xishi Qiu <qiuxishi@huawei.com> | [mm: mirrored memory support for page buddy allocations](https://lore.kernel.org/lkml/558E084A.60900@huawei.com) | 内存镜像的功能 | RFC v2 ☐  | [PatchWork RFC,v2,0/8](https://lore.kernel.org/lkml/558E084A.60900@huawei.com) |
| 2018/2/12 | David Rientjes <rientjes@google.com> | [mm: Introduce kernelcore=mirror option](http://lore.kernel.org/patchwork/patch/574230) | `kernelcore=` 和 `movablecore=` 都可以分别用于定义系统上 ZONE_NORMAL 和 ZONE_MOVABLE 的数量. 然而, 这需要在指定命令行时知道系统内存容量. 这个补丁引入了将 `kernelcore` 和 `movablecore` 定义为系统总内存的百分比的能力. 这对于希望将 ZONE_MOVABLE 的数量定义为系统内存的比例 (而不是硬编码的字节值) 的系统软件来说是很方便的. 要定义百分比, 参数的最后一个字符应该是 '%'. | v4 ☑ 4.17-rc1 | [LKML 1/2](https://lkml.org/lkml/2018/2/12/1024) |
| 2022/03/26 | mawupeng <mawupeng1@huawei.com> | [introduce mirrored memory support for arm64](https://patchwork.kernel.org/project/linux-mm/cover/20220326064632.131637-1-mawupeng1@huawei.com) | 镜像内存支持 arm64. 参见 phoronix 报道 [Huawei Working On UEFI Mirrored Memory Support For Linux AArch64](https://www.phoronix.com/scan.php?page=news_item&px=Linux-AArch64-Mirrored-Memory). | v1 ☐☑ | [LORE v1,0/9](https://lore.kernel.org/all/20220326064632.131637-1-mawupeng1@huawei.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/9](https://lore.kernel.org/r/20220414101314.1250667-1-mawupeng1@huawei.com)<br>*-*-*-*-*-*-*-* <br>[LORE v3,0/6](https://lore.kernel.org/r/20220607093805.1354256-1-mawupeng1@huawei.com) |
| 2022/04/19 | mawupeng <mawupeng1@huawei.com> | [Add support to relocate kernel image to mirrored region](https://patchwork.kernel.org/project/linux-mm/cover/20220419070150.254377-1-mawupeng1@huawei.com/) | 633242 | v1 ☐☑ | [LORE v1,0/2](https://lore.kernel.org/r/20220419070150.254377-1-mawupeng1@huawei.com) |


### 14.13.4 其他 ZONE_MOVABLE 相关
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/02/15 | Alistair Popple <apopple@nvidia.com> | [mm/pages_alloc.c: Don't create ZONE_MOVABLE beyond the end of anode](https://patchwork.kernel.org/project/linux-mm/patch/20220215025831.2113067-1-apopple@nvidia.com/) | 614354 | v1 ☐☑ | [PatchWork v1,0/1](https://lore.kernel.org/r/20220215025831.2113067-1-apopple@nvidia.com) |


## 14.14 SHMEM
-------

### 14.14.1 SHMEM
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/11/11  | Mina Almasry <almasrymina@google.com> | [mm/shmem: support deterministic charging of tmpfs](https://patchwork.kernel.org/project/linux-mm/patch/20211110211951.3730787-2-almasrymina@google.com) | NA | v1 ☐ | [PatchWork v2,1/4](https://patchwork.kernel.org/project/linux-mm/patch/20211110211951.3730787-2-almasrymina@google.com)<br>*-*-*-*-*-*-*-* <br>[PatchWork v3,1/4](https://patchwork.kernel.org/project/linux-mm/patch/20211111234203.1824138-2-almasrymina@google.com) |
| 2022/01/18 | Khalid Aziz <khalid.aziz@oracle.com> | [Add support for shared PTEs across processes](https://patchwork.kernel.org/project/linux-mm/cover/cover.1642526745.git.khalid.aziz@oracle.com) | NA| v1 ☐ | [LKML RFC,0/6](https://patchwork.kernel.org/project/linux-mm/cover/cover.1642526745.git.khalid.aziz@oracle.com)<br>*-*-*-*-*-*-*-* <br>[LORE v1,0/14](https://lore.kernel.org/r/cover.1649370874.git.khalid.aziz@oracle.com) |
| 2022/02/11 | Charan Teja Kalla <quic_charante@quicinc.com> | [mm: shmem: implement POSIX_FADV_[WILL|DONT]NEED for shmem](https://patchwork.kernel.org/project/linux-mm/patch/1644572051-24091-1-git-send-email-quic_charante@quicinc.com/) | 613418 | v4 ☐☑ | [PatchWork v4,0/1](https://lore.kernel.org/r/1644572051-24091-1-git-send-email-quic_charante@quicinc.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/2](https://lore.kernel.org/r/cover.1646987674.git.quic_charante@quicinc.com)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/2](https://lore.kernel.org/r/cover.1675690847.git.quic_charante@quicinc.com) |
| 2022/03/14 | Xavier Roche <xavier.roche@algolia.com> | [[v4] tmpfs: support for file creation time](https://patchwork.kernel.org/project/linux-mm/patch/20220314211150.GA123458@xavier-xps/) | 623317 | v4 ☐☑ | [LORE v4,0/1](https://lore.kernel.org/r/20220314211150.GA123458@xavier-xps)|
| 2022/04/18 | Gabriel Krisman Bertazi <krisman@collabora.com> | [shmem: Allow userspace monitoring of tmpfs for lack of space.](https://patchwork.kernel.org/project/linux-mm/cover/20220322222738.182974-1-krisman@collabora.com/) | 625589 | v1 ☐☑ | [2022/03/22 LORE v1,0/3](https://lore.kernel.org/r/20220322222738.182974-1-krisman@collabora.com)<br>*-*-*-*-*-*-*-* <br>[2022/03/28 LORE v2,0/3](https://lore.kernel.org/r/20220328020443.820797-1-krisman@collabora.com)<br>*-*-*-*-*-*-*-* <br>[2022/04/18 LORE v3,0/3](https://lore.kernel.org/r/20220418213713.273050-1-krisman@collabora.com) |
| 2022/07/26 | Liu Zixian <liuzixian4@huawei.com> | [shmem: support huge_fault to avoid pmd split](https://patchwork.kernel.org/project/linux-mm/patch/20220726124315.1606-1-liuzixian4@huawei.com/) | 663073 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220726124315.1606-1-liuzixian4@huawei.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/1](https://lore.kernel.org/r/20220726132751.1639-1-liuzixian4@huawei.com) |


### 14.14.2 Anonymous Shared Memory-Ashmem
-------

[Bringing Android closer to the mainline](https://lwn.net/Articles/472984)


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2011/12/20 | Robert Love <rlove@google.com> | [ashmem: Anonymous shared memory subsystem](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=11980c2ac4ccfad21a5f8ee9e12059f1e687bb40) | Android 匿名共享内存(Ashmem) | v1 ☐☑✓ | [LORE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=11980c2ac4ccfad21a5f8ee9e12059f1e687bb40) |
| 2022/03/15 | Christoph Hellwig <hch@lst.de> | [staging: remove ashmem](https://patchwork.kernel.org/project/linux-mm/patch/20220315123457.2354812-1-hch@lst.de) | ashmem 的主线替代品是 memfd, 因此从 `drivers/staging` 中删除遗留代码. | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20220315123457.2354812-1-hch@lst.de) |


## 14.15 Page Walk
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/08/22 | Rolf Eike Beer <eb@emlix.com> | [Minor improvements for pagewalk code](https://patchwork.kernel.org/project/linux-mm/cover/3200642.44csPzL39Z@devpool047/) | 669775 | v1 ☐☑ | [LORE v1,0/6](https://lore.kernel.org/r/3200642.44csPzL39Z@devpool047) |


## 14.16 Code Tagging Framework(代码标记框架)
-------

通过代码标记标识源代码中的特定位置, 该位置在编译时生成, 可以嵌入到特定于应用程序的结构中. [Code tagging framework and applications](https://lore.kernel.org/all/20220830214919.53220-1-surenb@google.com) 代码标记框架采用了 "为给定类型的对象定义一个特殊的 elf 部分, 以便我们可以在运行时遍历它们" 的老技巧, 并为它创建一个适当的库.

并提供代码标记的几个应用程序, 提供内存分配跟踪 (Memory allocation tracking)、动态故障注入(Dynamic fault injection)、延迟跟踪(Latency tracking) 和改进的错误代码报告(Improved error codes).

| 时间 | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:---:|:----:|:---:|:----:|:---------:|:----:|
| 2022/08/30 | Suren Baghdasaryan <surenb@google.com> | [Code tagging framework and applications](https://lore.kernel.org/all/20220830214919.53220-1-surenb@google.com) | TODO | v1 ☐☑✓ | [LORE v1,0/30](https://lore.kernel.org/all/20220830214919.53220-1-surenb@google.com) |

## 14.17 RSS
-------

| 时间 | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:---:|:----:|:---:|:----:|:---------:|:----:|
| 2009/06/02 | KAMEZAWA Hiroyuki <kamezawa.hiroyu@jp.fujitsu.com> | [mm rss counting updates](https://lore.kernel.org/patchwork/patch/182191) | NA | v1 ☑ 2.6.29-rc1 | [PatchWork v1](https://lore.kernel.org/patchwork/patch/182191)) |
| 2022/10/24 | Shakeel Butt <shakeelb@google.com> | [mm: convert mm's rss stats into percpu_counter](https://patchwork.kernel.org/project/linux-mm/patch/20221024052841.3291983-1-shakeelb@google.com/) | 688034 | v1 ☐☑ | [LORE v1,0/1](https://lore.kernel.org/r/20221024052841.3291983-1-shakeelb@google.com) |

## 14.18 Document
-------

[Improving memory-management documentation](https://lwn.net/Articles/894374)

[都 2022 年了, 这 20 篇 Linux 内存管理的期刊论文, 你读了吗？](https://zhuanlan.zhihu.com/p/450826949)


# 15 测试
-------

## 15.1 Intel® Memory Latency Checker (Intel® MLC)
-------

INTEL MLC 可以测量出机器的内存访问延迟和带宽, 并且可以观察出它们是如何随着机器负载的增加而变化的. Intel 的处理器有一些内存预取功能, 可能会影响测试结果.

[INTEL MLC (Memory Latency Checker) 介绍](https://zhuanlan.zhihu.com/p/359823092)

[MLC 内存测试结果解读到 CPU 架构设计分析](https://zhuanlan.zhihu.com/p/447936509)

[Intel® Memory Latency Checker](https://www.intel.com/content/www/us/en/developer/articles/tool/intelr-memory-latency-checker.html)

---

** 引用:**

<div id="ref-anchor-1"></div>
- [1] [Single UNIX Specification](https://en.wikipedia.org/wiki/Single_UNIX_Specification%23Non-registered_Unix-like_systems)

<div id="ref-anchor-2"></div>
- [2] [POSIX 关于调度规范的文档](http://nicolas.navet.eu/publi/SlidesPosixKoblenz.pdf)

<div id="ref-anchor-3"></div>
- [3] [Towards Linux 2.6](https://link.zhihu.com/?target=http%3A//www.informatica.co.cr/linux-scalability/research/2003/0923.html)

<div id="ref-anchor-4"></div>
- [4] [Linux 内核发布模式与开发组织模式(1)](https://link.zhihu.com/?target=http%3A//larmbr.com/2013/11/02/Linux-kernel-release-process-and-development-dictator-%26-lieutenant-system_1)

<div id="ref-anchor-5"></div>
- [5] IBM developworks 上有一篇综述文章, 值得一读 :[Linux 调度器发展简述](https://link.zhihu.com/?target=http%3A//www.ibm.com/developerworks/cn/linux/l-cn-scheduler)

<div id="ref-anchor-6"></div>
- [6] [CFS group scheduling [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/240474)

<div id="ref-anchor-7"></div>
- [7] [http://lse.sourceforge.net/numa/](https://link.zhihu.com/?target=http%3A//lse.sourceforge.net/numa)

<div id="ref-anchor-8"></div>
- [8] [CFS bandwidth control [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/428230)

<div id="ref-anchor-9"></div>
- [9] [kernel/git/torvalds/linux.git](https://link.zhihu.com/?target=https%3A//git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/%3Fid%3D5091faa449ee0b7d73bc296a93bca9540fc51d0a)

<div id="ref-anchor-10"></div>
- [10] [DMA 模式 \_百度百科](https://link.zhihu.com/?target=http%3A//baike.baidu.com/view/196502.htm)

<div id="ref-anchor-11"></div>
- [11] [进程的虚拟地址和内核中的虚拟地址有什么关系? - 詹健宇的回答](http://www.zhihu.com/question/34787574/answer/60214771)

<div id="ref-anchor-17"></div>
- [17] [kernel 3.10 内核源码分析 --TLB 相关 --TLB 概念、flush、TLB lazy 模式 - humjb\_1983-ChinaUnix 博客](https://link.zhihu.com/?target=http%3A//blog.chinaunix.net/uid-14528823-id-4808877.html)

<div id="ref-anchor-18"></div>
- [18] [Toward improved page replacement[LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/226756)

<div id="ref-anchor-20"></div>
- [20] [The state of the pageout scalability patches [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/286472)

<div id="ref-anchor-21"></div>
- [21] [kernel/git/torvalds/linux.git](https://link.zhihu.com/?target=https%3A//git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/%3Fid%3D894bc310419ac95f4fa4142dc364401a7e607f65)

<div id="ref-anchor-22"></div>
- [22] [Being nicer to executable pages [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/333742)

<div id="ref-anchor-23"></div>
- [23] [kernel/git/torvalds/linux.git](https://link.zhihu.com/?target=https%3A//git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/%3Fid%3D8cab4754d24a0f2e05920170c845bd84472814c6)

<div id="ref-anchor-24"></div>
- [24] [Better active/inactive list balancing [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/495543)

<div id="ref-anchor-29"></div>
- [29] [On-demand readahead [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/235164)

<div id="ref-anchor-30"></div>
- [30] [Transparent huge pages in 2.6.38 [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/423584)

<div id="ref-anchor-32"></div>
- [32] [transcendent memory for Linux [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/338098)

<div id="ref-anchor-33"></div>
- [33] [linux kernel monkey log](https://link.zhihu.com/?target=http%3A//www.kroah.com/log/linux/linux-staging-update.html)

<div id="ref-anchor-34"></div>
- [34] [zcache: a compressed page cache [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/397574)

<div id="ref-anchor-36"></div>
- [36] [Linux-Kernel Archive: Linux 2.6.0](https://link.zhihu.com/?target=http%3A//lkml.iu.edu/hypermail/linux/kernel/0312.2/0348.html)

<div id="ref-anchor-37"></div>
- [37]抢占支持的引入时间: [https://www.kernel.org/pub/linux/kernel/v2.5/ChangeLog-2.5.4](https://link.zhihu.com/?target=https%3A//www.kernel.org/pub/linux/kernel/v2.5/ChangeLog-2.5.4)

<div id="ref-anchor-38"></div>
- [38] [RAM is 100 Thousand Times Faster than Disk for Database Access](https://link.zhihu.com/?target=http%3A//www.directionsmag.com/entry/ram-is-100-thousand-times-faster-than-disk-for-database-access/123964)

<div id="ref-anchor-39"></div>
- [39] [http://www.uefi.org/sites/default/files/resources/ACPI\_6.0.pdf](https://link.zhihu.com/?target=http%3A//www.uefi.org/sites/default/files/resources/ACPI_6.0.pdf)

<div id="ref-anchor-40"></div>
- [40] [Injecting faults into the kernel [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/209257)

<div id="ref-anchor-47"></div>
- [47] [Fast interprocess messaging [LWN.net]](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/405346)



---8<---

** 更新日志:**

**- 2015.9.12**

o 完成调度器子系统的初次更新, 从早上 10 点开始写, 写了近７小时, 比较累, 后面更新得慢的话大家不要怪我(对手指

**- 2015.9.19**

o 完成内存管理子系统的前 4 章更新. 同样是写了一天, 内容太多, 没能写完......

**- 2015.9.21**

o 完成内存管理子系统的第 5 章 "页面写回" 的第 1 小节的更新.
**- 2015.9.25**

o 更改一些排版和个别文字描述. 接下来周末两天继续.
**- 2015.9.26**

o 完成内存管理子系统的第 5, 6, 7, 8 章的更新.
**- 2015.10.14**

o 国庆离网 10 来天, 未更新.  今天完成了内存管理子系统的第 9 章的更新.
**- 2015.10.16**

o 完成内存管理子系统的第 10 章的更新.
**- 2015.11.22**

o 这个月在出差和休假, 一直未更新. 抱歉! 根据知友 [@costa](https://www.zhihu.com/people/78ceb98e7947731dc06063f682cf9640) 提供的无水印图片和考证资料, 进行了一些小更新和修正. 特此感谢 !

o 完成内存管理子系统的第 11 章关于 NVDIMM 内容的更新.
**- 2016.1.2**

o 中断许久, 今天完成了内存管理子系统的第 11 章关于调试支持内容的更新.
**- 2016.2.23**

o 又中断许久, 因为懒癌发作 Orz... 完成了第二个子系统的所有章节.
[编辑于 06-27](https://www.zhihu.com/question/35484429/answer/62964898)
