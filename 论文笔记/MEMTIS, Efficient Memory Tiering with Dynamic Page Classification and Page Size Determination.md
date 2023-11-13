
# Background

之前关于内存分层的工作往往无法做出最优的页放置决策，因为它们依赖于各种启发式方法和静态阈值，而没有考虑整体的内存访问分布。此外，为应用程序确定合适的页大小也很困难，由于内存访问并不总是均匀的，使用大页不一定能得到正向收益。

此前的分层内存系统利用 page fault，PT scanning，hardware-based sampling等方法进行访存追踪。然而基于page fault的系统会显著增加内存访问的 latency，基于PT scanning的系统的可扩展性较差，基于硬件sampling 的工作无法追踪 subpage access。

# Design

本文提出Memtis，一种利用informed decision-making机制来制定页大小和页放置策略的分层内存系统。Memtis利用已分配页的访问分布，以最优的方式逼近热数据集的fast tier capacity。

Memtis 主要解决三个问题：
- 如何使用基于硬件的内存访问采样，以细粒度和轻量级的方式跟踪内存访问：使用Processor Event-Based Sampling (PEBS) 对内存访问进行采样。由于PEBS样本包含精确的内存地址，使得Memtis可以支持细粒度的访问跟踪。在Memtis维护一个内核后台线程（ksampled）来处理采样的地址，Memtis可以动态调整访存采样频率，确保CPU开销在阈值(< 3%)以下。
- 如何根据整体内存访问频率分布，动态准确地确定热页和冷页：Memtis使用page access counts来维护所有已分配页面的热度分布，并且利用该数据做出最佳的memory tiering决策，将最热的页放置在fast tier以最小化访问延迟。
- 如何动态确定页大小(大页与普通页)，以在不浪费fast tier 内存的情况下降低转换成本：在Linux中默认使用Transparent Huge Pages(THP)来减少地址翻译的开销。使用ksampled维护的emulated base page histogram来估计仅使用普通页时的最大命中率，并将估计的最大命中率与从PEBS的采样记录中获得的实际命中率进行比较，以此来估计拆分大页的潜在收益。如果潜在的好处很大，Memtis会选择子页面中访存模式最不均匀的大页作为拆分候选页。然后，在后台对大页进行拆分，并根据大页中维护的子页访问信息将每个分片后的子页放置到相应的内存层。