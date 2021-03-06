# 2.6　评价

我们通过在`Amazon EC2` 上进行一系列的实验和用户应用程序的基准测试，对`Spark`和`RDDs`进行了性能评估。总体而言，我们的测试结果显示如下：
- 在迭代机器学习和图形应用程序中，`Spark` 性能要比`Hadoop` 模型好80倍。这些性能提升来自于将数据以`java` 对象存入内存从而避免系统` I/O `和反序列化带来的成本。
- 用户应用程序同样有很好的性能和扩展性。尤其，我们使用 `Spark` 来运行一个原本运行在`Hadoop`上的分析报告，性能提升了 40 倍.。
- 当出现节点故障时，`Spark`可以通过只重建那些丢失的`RDD` 分区，从而快速地故障恢复。
- `Spark` 可以在 5-7 秒延迟内交互式地查询 1 `TB` 的数据集。

我们首先提供了基准测试，为迭代机器学习（§2.6.1)）以及`PageRank` 算法（§2.6.2）与`Hadoop`进行了对比。然后评估了`Spark`的容错性（§2.6.3）
和数据不能完全存入内存（§2.6.4）时的状况表现。最后，我们讨论了它们在交互式数据挖掘（§2.6.5）和一些真实用户应用程序（§2.6.6）中的表现。

除非另有说明，我们在测试中使用` m1.xlarge EC2` 节点，配有 4 核`CPU` 和 15 `GB` 的内存。我们使用`HDFS` 来存储数据，每个文件块是256`MB`。
每次测试之前，我们都会清除操作系统的缓冲区，从而得到更精确的`I/O` 开销。

# 2.6.1 迭代式机器学习应用

我们实现了两种迭代式机器学习应用，逻辑回归和 k-均值算法（`k-means`），来对以下系统进行性能对比：
- `Hadoop` ：` Hadoop 0.20.2` 稳定版
- `HadoopBinMem`：一种`Hadoop`，在首轮迭代中将输入数据转换成为开销较低的二进制格式，从而削减了后续迭代中文本解析的开销，并将数据存储在基于内存的`HDFS` 中。
• `Spark`：我们的`RDDs`的实现版本。

![2.7](../images/2.7.png "2.7")
图 2.7 在一个 100节点集群上处理 100`GB`数据的逻辑回归和k-均值算法，`Hadoop`，`HadoopBinMem`和`Spark` 各自的首次迭代和后续迭代时长。

我们使用 25-100 台机器来运行两种算法，在 100`GB`数据集上迭代 10 次。两个应用的关键区别在于对每个字节数据的计算量不同。
`k-means`的迭代时间取决于计算量，然而逻辑回归并非计算密集型的，对反序列化和`I/O` 时间开销更敏感。

由于典型的机器学习算法需要数十轮迭代直到收敛，我们分别统计了首轮迭代和后续迭代的耗时。我们发现通过`RDDs`共享数据极大地加快了后续迭代的速度。

**首轮迭代**：在首轮迭代过程中，所有 3 个系统都是从`HDFS`中读取文本数据作为输入数据。如图 2.7 中的浅色条所示，整个实验中`Spark`都要比`Hadoop`快一些。
差异是因为`Hadoop`中的`Master` 和`Slave`之间基于心跳协议的信令开销。`HadoopBinMem` 是最慢的，因为它运行了一个额外的`MapReduce` 作业来将数据转换成二进制格式，
并且它需要通过网络传输将数据写入备份的内存式HDFS 实例中。
**后续迭代**：图 2.7 显示了后续迭代的平均耗时。对于逻辑回归，在 100 台机器上运行，`Spark`分别比`Hadoop`和`HadoopBinMem`快 85 和 70 倍。
对于更加计算密集型的` K-means`，`Spark`仍然分别比`Hadoop`和`HadoopBinMem`快 26 和 21 倍。注意在所有这些案例中，程序通过同样的算法计算出了同样的结果。
**理解速度提升**： 我们非常惊奇地发现，`Spark`甚至超过了基于内存存储二进制数据的 `Hadoop(HadoopBinMem)`高达 85 倍之多。
在`HadoopBinMem`中，我们使用`Hadoop`的标准二进制格式（序列文件）和 256` MB` 的大文件块，并且我们强制使`HDFS`的数据目录加载到内存文件系统中。
然而，`Hadoop`仍然运行的比较慢，是由于以下几个原因：

1. `Hadoop`软件栈的最低开销
2. `HDFS` 读取数据的开销
3. 将二进制记录转化为可用的在内存中的`Java` 对象的反序列化的代价

我们通过单独的微基准测试确认了这些因素。例如，为了测试`Hadoop`的启动开销，我们运行空操作（`no-op`）的`Hadoop`作业，观察到仅仅完成作业的最小需求：设置、启动任务、清理工作就
至少耗时 25 秒。至于`HDFS` 的开销，我们发现为了维护每一个数据块，`HDFS`进行了多次内存复制以及校验和计算。最后，我们发现，即使是在内存中的二进制数据，通过`Hadoop`
的`SequenceFileInPutFormat` 读取数据来反序列化这一步，也比逻辑回归的计算花费了更多时间，这解释了为什么`HadoopBinMem`更慢。

# 2.6.2 `PageRank`

![2.8](../images/2.8.png "2.8")
图 2.8 在`Hadoop`和`Spark`各自执行`PageRank`的性能

我们对一个 54`GB` 的维基百科数据进行`PageRank`来比较`Hadoop`和`Spark`的性能。我们对`PageRank`算法迭代 10 次，处理一个大约有 400 万词条的连接图。
由图 2.8 可见，在 30个节点的集群上，基于内存存储的`Spark`获得了相较于`Hadoop` 2.4 倍的性能提升。此外如 2.3.2 章节所述，控制`RDDs` 的分区
使其在每次迭代中保持一致，可以将性能提升到 7.4 倍。同样，当节点扩展到 60个时，性能也保持着线性增长。

我们也评测了一种基于`Spark`的`Pregel` 实现的`PageRank`（将在 2.7.1 小节描述）。迭代次数和图 2.8描述类似，但大约长了 4 秒钟。
这是因为每轮迭代中，`Pregel`都需要额外的运算让顶点“投票”决定是否结束作业。

# 2.6.3 故障恢复

![2.9](../images/2.9.png "2.9")
图 2.9 出现故障时`K-means`的迭代时间。由于一台机器在第六次迭代开始就被终止了，这就导致`RDD`会通过其血统来进行部分重建。

我们评估了在`K-means`作业中一个节点出现故障后，`Spark`通过血统重建`RDD`分区的开销。图 2.9 显示了在一个 75 节点集群正常运行场景下，对`K-means`算法迭代 10 次的运行
时间。在这种场景下，有个节点会在第六轮迭代开始时出现故障。当没有任何故障时，每轮迭代包含 400 个运行在 100`GB` 数据上的任务。

直到第 5 次迭代结束，迭代时间约为 58 秒。在第 6 次迭代，一台机器被杀掉，导致了该机器丢失了其正在运行的任务和缓存的`RDD`分区。`Spark`将并发地在其他机器上重新执行这些任务。
这些任务重新读取相应的输入数据并根据血统重建`RDDs`，导致迭代时间增长到 80 秒。一旦丢失的`RDD`分区被重建完成，迭代时间将会降回58 秒。

注意，基于检查点的故障恢复机制，可能会需要重新运行几轮迭代来实现恢复，而迭代的次数取决于设置检查点的频率。此外，该系统可能需要通过网络备份 100`GB` 的工作集（被转换成二进制数据
的文本数据），这将会消耗两倍`Spark`的内存以将数据备份在内存中，或者将不得不等待数据集写入磁盘完成。相反，我们例子中`RDDs`血统图的大小都是小于 10`KB`。

# 2.6.4 内存不足的表现

目前为止，我们保证了集群中每一台机器都有足够的内存来缓存迭代过程中所有的`RDDs`。一个很自然的问题是在没有足够内存可以存储一个任务所需的数据的情况下，`Spark`如何运行。
在这个实验中，我们配置了`Spark`，只能使用一定比例的内存来缓存`RDDs`。我们在图 2.10 中展示了逻辑回归算法在使用不同大小存储空间下的表现。我们发现，当使用较少内存空间时性能急剧下降。

![2.10](../images/2.10.png "2.10")
图 2.10 在内存中缓存不同数量的数据时，逻辑回归算法的性能表现（100`GB` 数据以及 25 个节点）

![2.11](../images/2.11.png "2.11")
图 2.11 `Spark`上交互式查询的响应时间（在 100台机器的集群上扫描逐步增加的数据量）

# 2.6.5 交互式数据挖掘

为了说明 `Spark`对大数据集进行交互式查询的能力，我们用它来分析 1`TB` 的维基百科页面日志（2 年的数据）。在这个实验中，我们使用了 100 个`m2.4xlarge EC2` 节点，各配有 8 个` CPU`
内核和 68`GB` 内存。我们通过查询来找出总的视图量，包括（1）所有页面，（2）标题与指定关键字完全匹配的页面以及（3）标题与指定关键词部分匹配的页面。每个查询都会扫
描所有的输入数据。

图 2.11 给出了在完整数据集，50%数据集，和 10%数据集上查询操作各自对应的响应时间，即使在 1`TB`数据上，`Spark`查询操作也仅花了 5-7 秒。相比于对磁盘数据进行查询，这速度提升了不止
一个数量级；例如，从磁盘上查询 1`TB` 的文件需要花费 170 秒。这证明了`RDDs`可以使得`Spark`成为一个强大的交互式数据挖掘工具。

# 2.6.6 实际应用

![2.12](../images/2.12.png "2.12")
图 2.12 `Spark`实现的两个用户应用的每轮迭代时间，误差表示为标准差

- 内存分析：一家视频分发公司`Conviva Inc`，用`Spark`替代了`Hadoop` 来加速一些以前的数据分析报告应用。
例如，其中一个报告运行了一系列`Hive` 查询，为消费者计算各种统计数据。这些查询都是在相同的数据子集（那些和用户提供的过滤条件匹配的记录）上进行的, 但是对不同的组域进执行
聚合操作 (平均值，百分比和 统计非重复值) ，这些操作都需要独立的`MapReduce` 作业来完成。通过在`Spark`上实现查询，并一次性把它们之间共享的数据子集加载到`RDD`中，公司的报告生成速度
可以提高到之前的40倍。`Hadoop` 集群处理 200 `GB`的压缩数据要花费 20 个小时，而`Spark`只需要使用2 台机器在 30 分钟完成。另外，`Spark`只需要 96`GB`的内存，因为它只需要将
符合用户过滤规则的行和列存储在`RDD`中，而不是整个解压缩文件。

- 交通建模： 伯克利大学的` Mobile Millennium` 项目【57】研究人员并行化了一个通过零星的汽车`GPS` 数据去推断道路拥堵情况的学习算法。
数据源是市区中 10000 条道路交通网，以及装有`GPS` 设备的汽车点到点花费时间的 60 万个样本（每个线路的花费时长可能包括多条道路）。基于一种交通模型，该
系统可以估算出经过某条特定线路需要的时间。研究人员使用期望最大化（`EM`）算法来训练这个模型，该算法会迭代两次`map` 和`reduceByKey` 操作。该应用测试了从 20 节点线性扩展到 80
个节点，每个节点配置 4 个`CPU`内核，如图 2.12(a)所示。

- `Twitter` 垃圾分类： 伯克利大学的`Monarch` 项目【102】使用`Spark` 来识别`Twitter`消息中的垃圾信息。他们在`Spark`上使用了类似于 2.6.1 中的逻辑回归分类器，
但是他们使用了分布式的`reduceByKey` 操作来并行地计算梯度向量的总和。在图 2.12(b)中，我们展示了在50`GB`数据子集上分类器训练的性能扩展结果：
这些数据包括 25 万个`URL`以及千万个与这些地址页面相关的网络特性/维度和每个`URL`上页面的内容特性。由于每轮迭代中有较高的固定网络开销，性能并没有获得线性的增长。