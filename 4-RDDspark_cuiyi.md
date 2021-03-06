现有模式存在的问题：
现在的计算架构处理一些计算问题的时候效率比较低，iterative algorithm和interactive data mining，因为现有的算法不是in-memory computing的，所以对于很多reuse中间结果的算法来说，效率就非常低，而之前的解决方案则是向外部的storage输出计算的中间结果，这就因为replica，disk I/O等问题，效率大大降低了
也有很多框架着手解决这一问题，而最大的障碍在于怎样对于in-memory computing做到fault-tolerant，而现有的方法是在节点之间做数据的备份或者是log的备份，效率就大大降低了
解决的方案：提出了Resilient Distributed Dataset，可以在大规模的计算集群上面运行in-memory computing的算法，基于coarse-grained transformation。如果一个RDD的partition丢失了，就根据lineage重新计算一个partition出来。


RDD: 
弹性分布式数据集,RDD是只读的、分区的记录集合。这个数据集的全部或者部分可以缓存在内存中，以便在迭代计算的时候可以重复利用，减少了I/O的开销，当内存不够时，可以与磁盘进行交换。用户可以显式地指定哪一部分分区的数据可以缓存在内存中，以及内存不够时缓存这些分区的优先级。RDD只能通过对稳定物理存储中的数据或者其他已有的RDD执行一些确定性操作来创建。这些操作被称作transformation，常见的transformation包括map, filter,join等
一个RDD有足够的信息(lineage)关于它自身是如何从其他数据集或其他RDD衍生而来的，使用这些信息，利用稳定存储中的数据可以再次计算出这个RDD中的分区。
用户可以显式地指定RDD的存储方式和分区方式：比如指定哪个RDD会被重用，指定一个RDD是存在磁盘中还是内存中；决定RDD中的元素按照某个key来划分，及其划分的方式等。
RDD具有很好的容错机制，Spark可以根据lineage的信息重建丢失的RDD分区，而不需在各个结点之间拷贝大量的数据副本。

Spark提供的接口:
Spark中RDD的操作：
1、定义及创建RDD的接口(transformation):
map, filter
flatMap
union,join,sample
sort, partitionBy
2、使用RDD的接口(action):
这些操作给应用程序返回一个结果或者向存储系统中写入数据
count:返回数据集中元素的个数
collect:返回元素本身
reduce
save:向存储系统写入数据集
persist:指定以后要复用的RDD，spark默认将要复用的RDD放在内存中

Spark中RDD的内部接口：
partitions()：返回一组Partition对象
preferredLocations(p):根据数据存放的位置，返回分区p在哪些节点访问更快
dependencies():返回一组依赖
iterator(p, parentIters)：按照父分区的迭代器，逐个计算分区p的元素
partitioner():返回RDD是否被hash/range分区的元数据信息

RDD相比于其他集群程序框架模型的优点:
MapReduce, Dryad和Ciel提供了丰富的数据处理操作，但是他们通过稳定的物理外存来共享复用数据。而RDD不需要通过物理外存来共享数据，从而避免了数据冗余,I/O和序列化的开销。

一些高等级的编程接口如Dryad，FlumeJava等提供了API使得用户可以操纵并行集合，但是这些系统不能有效地跨查询共享数据。RDD保留了这些更高级别的编程特性，同时也能支持更多类型的应用。

Pregel,Twister和HaLoop支持迭代的运算。然而这些框架只是对他们所支持的特定的计算模式提供隐式的数据共享复用。用户不能显式地指定将哪个数据集加载到内存中然后执行查询，只是为了达到优化的目的上述模型会在运行时隐式地将数据集加载到内存然后共享复用。相反，RDD显式地提供了分布式存储抽象模型，使得用户可以显式控制加载数据集到内存中，提高了灵活性。此外还支持一些现有系统未能处理的应用，如利用scala解释器来进行交互式数据挖掘。

一些系统提供了共享可变状态，通过它来允许用户执行内存计算。比如Piccolo使得用户可以运行并行函数来读和更新分布式hash表中的元素。分布式共享内存系统(DSM)和RAMCloud也提供了类似的模型。但是Piccolo和DSM提供的接口只能读或者更新表中的元素。而RDD提供了更高等级的编程接口，这些接口基于map,sort,join等算子。另一方面Piccolo和DSM只能通过设置检查点和回滚来实现错误恢复，这样的开销远比RDD中基于lineage的恢复机制的开销要高得多。RDD只需要将lineage的信息做log，当分区丢失的时候，通过lineage可以重新计算出丢失的分区。此外，由于RDD是不可变的，这一特性有助于缓解缓慢结点(stragglers)的情况——可以复制一份slow task的副本然后在其他结点上运行。

论文中代码的执行：

一、

![](img/4_code1.1.png)

![](img/4_code1.2.png)

第一行定义了一个由HDFS保存的RDD
第二行将第一行产生的RDD过滤掉不必要的信息，保留以ERROR开头的信息，产生新的RDD
第三行将error RDD保留在内存中，以便于它可以被跨查询共享。Spark会将error的分区保存在内存中
然后对error RDD再次过滤出包含"HDFS"的数据项，然后筛选出以制表符分隔的第三个字段。Spark调度器会将这些transformation指令发送给缓存着errors分区的结点。最后将结果记录返回。

二、逻辑回归

![](img/4_code2.png)

将文本文件的每一行做map转换解析成Point对象，然后将所得的points RDD保留在内存中。
随机生成一个向量赋给w
在缓存有points RDD的结点中反复调用map和reduce转换，在每一步的迭代中利用w的函数计算gradient的值。然后将w与gradient的值相减得到新的w，带入下一次迭代。迭代多次后，w会收敛，得到最终的结果。

三、PageRank

![](img/4_code3.png)

首先将图分解成url和它所指向的链接的对组成的RDD links，然后将这个RDD缓存在内存中
随机初始化一个url和它所对应的rank值组成的RDD ranks。
构建一个contribs RDD，该RDD包含了url以及指向它的url对其rank值所做的贡献。在每一步的迭代中都用links和当前ranks的值更新contribs的值，然后再用计算得到的contribs的值更新ranks的值，然后进行下一次迭代。迭代多次后，ranks的值会收敛。每一步迭代都会更新ranks的值，因此为了减少错误恢复的时间，用户可以在迭代一定次数后将ranks的值写入到磁盘做备份，这样以来当ranks的分区丢失时，就不需要从头开始迭代计算了。此外，可以人为地将links RDD根据URL在结点之间进行分区，然后将ranks按照同样的方式进行分区，这样以来在join的时候就不需要跨结点进行通讯了。Spark将每个url当前的贡献值发送到它的link lists所在的机器结点上，在那些结点机器上计算对应的URL的新的rank值，然后再与其link lists做join，依此类推。迭代多次后ranks值会收敛。

作业：


1.What are the advantages of spark compared to MapReduce?
1.Spark 在内存中处理数据，而 Hadoop MapReduce 是通过 map 和 reduce 操作在磁盘中处理数据，所以处理某一些应用Spark比Mapreduce效率要高。
2. 使用Mapreduce的时候需要将原来的算法转化并且分解为Map和Reduce，相比之下增加了编程的难度，尤其是很多问题并不适合这样来解决。
3.Spark支持多种多种语言以及很多现有的算法，能够很方便的进行整合以及快速地进行开发。
2.Describe the pros and cons of lineage and checkpoint?
lineage的优点：一个RDD通过lineage记录它如何从其他RDD转化而来，如果一个RDD出错，就可以通过lineage的链进行还原。
lineage的缺点：果有一个任务计算时间需要很长，而中间发生错误，如果使用lineage的方法的话需要从头开始进行计算，额外开销会比较大。
Checkpoint优点：可以很容易地进行recover，与使用lineage进行恢复相比，而使用checkpoint就可以直接恢复到之前的某一个状态
Checkpoint的缺点：占用额外的储存空间，如果没有及时做checkpoint的话会丢失数据。
3.Describe which applications are RDD suitable for and not suitable for?
1.RDD适合计算使对内存需要比较小的，需要进行迭代计算的应用。尤其适合应对批处理命令比较多的应用，在对同样的数据集进行相同的操作的情况下优势会比较的明显。
2.RDD不适合对内存比较大的，需要不断从存储器读取数据的应用，尤其是那些需要异步地，细粒度地修改共享数据的应用，会显著地降低计算的效率。





