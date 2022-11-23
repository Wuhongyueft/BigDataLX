# MapReduce预习报告

##MapReduce介绍##
MapReduce是一种用于在大型商用硬件集群中对海量数据实施可靠的、高容错的分布式计算框架，也是一种经典的并行计算模型。
###MapReduce核心思想###
它的核心思想是“分而治之”。所谓“分而治之”就是把一个复杂的问题，按照一定的分解方法分成等价的规模较小的若干部分，然后逐个解决。
##MapReduce编程模型##
使用MapReduce分析海量数据时，每个Mapreduce程序被初始化为一个工作任务，每个工作任务可以分为Map和Reduce两个阶段。其中Map阶段用于对原始数据进行处理，Reduce阶段用于对Map阶段的结果进行汇总，得到最终结果。从数据格式上看，map()函数接收的数据格式是键值对，产生的输出结果也是键值对，reduce()函数会将map()函数输出的键值对作为输入，把相同key值的value进行汇总，输出新的键值对。

通过下图对简单数据流模型进行说明。
![MapReduce简易数据流模型][1]

  [1]: http://www.itcast.cn/files/image/202112/20211206101032030.jpg
*** MapReduce简易数据流模型***
    1. 对原始数据处理成键值对<K1,V1>形式。
    2. 将解析后的键值对传给map()函数，map()函数会根据映射规则，将键值对<K1,V1>映射为一系列中间结果形式的键值对<K2,V2>
    3. 将中间的键值对<K2,v2>形成<K2,{V2,…}>形式传给reduce()函数处理，将具有相同key值的value合并在一起，产生新的键值对<K3,V3>，此时的键值对就是最终输出的结果。

根据工作原理可将MapReduce编程模型分为两类：MapReduce简单模型和MapReduce复杂模型。
***1. MapReduce简单模型***
对于某些任务来说，可能并不一定需要Reduce过程那么只需要map()函数处理就可以了。该模型只有Mapper过程，由Mapper产生的数据直接写入HDFS。
***2. MapReduce复杂模型***
对于大部分任务来说，都是需要Reduce过程，并且由于任务繁重，会启用多个Reducer来进行汇总。

##MapReduce数据流##
从MapReduce编程模型中可以发现数据以不同的形式在不同的节点之间流动，即经过本结点的分析处理，以另外一种形式进入下一个结点，从而得出最终结果。
MapReduce数据流处理过程主要分以下5个过程。

 **1. 分片、格式化数据源（InputFormat）**
 输入Map阶段的数据源，必须进行分片和格式化操作。Hadoop会为每一个分片构建一个Map任务，并由该任务运行自定义的map()函数，从而处理废片里的每一条记录。
然后将划分好的分片化为键值对<key,value>的数据。

 **2. Map过程**
 它处理每条输入记录并生成新的键值对，而Mapper生成的这个键值对与输入对完全不同。Mapper的输出也称为中间输出，写入本地磁盘。Mapper的输出不存储在HDFS上，因为这是临时数据，在HDFS上写入会创建不必要的副本（HDFS也是一个高延迟系统）。映射器的输出传递给组合器进行进一步处理。

 **3. Combiner过程**
 在Mapper阶段会产生大量相同的数据，势必会减低Reduce聚合阶段的执行效率。Combine()的作用就是对Map阶段输出的重复数据先做一次合并运算，然后把新的(key,value)做为Reduce()阶段的输入。

 **4. Shuffle阶段**
 Shuffle是MapReduce的核心，它用来确保每个Reduce的输入都是按键排序的，它的性能高低直接决定了整个MapReduce程序性能的高低。
整个Shuffle过程可以分为两个阶段，Mapper端的Shuffle和Reducer端的Shuffle。
1.map结果写入环形内存缓冲区，当内存不足以存储所有数据时，将数据批量溢写到磁盘。为了尽量减少IO消耗，所以在数据写入磁盘之前会先写入缓冲区，待缓冲区达到阈值后才批量将数据写入磁盘。
2.partition分区。在数据写入磁盘之前会先进行分区，一个分区对应一个reducer，期望数据在多个reducer之间达到均衡
3、排序（sort）和合并（combine）。数据经过分区之后，先按照key进行排序，如果用户指定了Combiner，再进行combine操作
4、溢写（spill）。经过排序和合并之后的数据会写入磁盘文件，每次spill都会产生一个文件。一个分区上的文件也叫一个segment。
5、归并（merge）。一个map最终会生成一个磁盘文件，由于多次spill会产生多个文件，所以需要将这些文件进行merge，最终形成一个有序的大文件。merge过程中有可能遇到相同key的数据，如果用户设置了Combiner，会执行combine操作
以上1-5是map阶段的shuffle，以下是reduce阶段的shuffle步骤
6、拷贝（copy）。当某个map完成后，reduce不断拉取map生成的文件到ruduce。和map阶段一样先将数据写入环形内存缓冲区，当达到阈值时，将数据批量溢写到磁盘
7、排序（sort）和归并（merge）。sort是伴随copy动作时执行的，由于map的输出是有序的，所以copy是进行sort消耗很低。当溢写数据到磁盘之前，如果用户设置了Combiner会先进行combine，然后将数据写入磁盘文件。当接受完map数据会生成多个溢写磁盘文件，将这些文件归并merge，合并成一个有序的大文件。

 **5.Reduce过程**
 1. Reduce会接收到不同的map任务传来的数据。并且每一个map传来的数据都是有序的。如果Reduce阶段接收的数据量相当小，则直接存储在内存中，如果数据量超过了该缓冲区大小的一定比例，则对数据合并后溢写到磁盘中。
 2. 随着溢写文件的增多，后台线程会将它们合并成一个更大的有序的文件。
 3. 合并的过程中会产生许多的中间文件（写入磁盘了）,但MapReuce会让写入磁盘的数据尽可能地少，并且最后一次合并的结果并没有写入磁盘，而是直接输入到reduce()函数。
 
 **MapReduce任务流运行流程**
 MapReduce的任务流程是从客户端提交任务开始，直到任务运行结束的一系列流程。MRv2是hadoop2中的MapReduce任务运行流程。在MRv2中，MapReduce运行时环境由Yarn提供，需要MapReduce相关服务和Yarn相关服务进行协同工作。
 ***MRv2基本组成***
MRv2的基本思想是将JobTracker的两个主要功能，资源管理和作业调度/监视的功能拆分为独立的守护进程。设计思想是将MRv1中的JobTracker拆分成了两个独立的服务：一个全局的资源管理器ResourceManager(RM)和每个应用程序特有的ApplicationMaster(AM)。每个应用程序要么是单个作业，要么是DAG作业。MRv2基本组成如下。
1. 客户端：客户端用向于向Yarn集群提交任务，是MapReduce用户和Yarn集群通信的唯一途径。
2. MARAppMaster：它监控和调度一整套MR任务流程,每个MR任务只产生一个MAPAppMaster。
3. Map task和Reduce task:用户定义的Map函数和Reduce函数的实例化，在MRv2中，它们值运行在Yarn给定的资源限制下，由MRAppMaster和NodeManage协同管理和调度。
***Yarn基本组成***
从 YARN 的架构图来看，它主要由ResourceManager、NodeManager、ApplicationMaster和Container等以下几个组件构成。
*1. ResourceManager*
RM 是一个全局的资源管理器，负责整个系统的资源管理和分配。它主要由两个组件构成：
调度器（Scheduler）和应用程序管理器（Applications Manager，ASM）。
YARN 分层结构的本质是ResourceManager。这个实体控制整个集群并管理应用程序向基础计算资源的分配。ResourceManager将各个资源部分（计算、内存、带宽等）精心安排给基础 NodeManager（YARN 的每节点代理）。ResourceManager 还与 ApplicationMaster 一起分配资源，与 NodeManager一起启动和监视它们的基础应用程序
1. 调度器
调度器根据容量、队列等限制条件（如每个队列分配一定的资源，最多执行一定数量的作业等），将系统中的资源分配给各个正在运行的应用程序。该调度器是一个“纯调度器”，它不再从事任何与具体应用程序相关的工作。
2. 应用程序管理器
应用程序管理器负责管理整个系统中所有应用程序，包括应用程序提交、与调度器协商资源以启动ApplicationMaster、监控ApplicationMaster运行状态并在失败时重新启动它等。
*2. NodeManager*
NodeManager管理一个YARN集群中的每个节点。NodeManager提供针对集群中每个节点的服务，从监督对一个容器的终生管理到监视资源和跟踪节点健康。MRv1通过插槽管理Map和Reduce任务的执行，而NodeManager 管理抽象容器，这些容器代表着可供一个特定应用程序使用的针对每个节点的资源。YARN继续使用HDFS层。它的主要 NameNode用于元数据服务，而DataNode用于分散在一个集群中的复制存储服务。
1. 单个节点上的资源管理；
2. 处理来自ResourceManager上的命令；
3. 处理来自ApplicationMaster上的命令。
*3.ApplicationMaster（AM）*
 ApplicationMaster管理一个在YARN内运行的应用程序的每个实例。ApplicationMaster 负责协调来自 ResourceManager 的资源，并通过 NodeManager 监视容器的执行和资源使用（CPU、内存等的资源分配）。请注意，尽管目前的资源更加传统（CPU 核心、内存），但未来会带来基于手头任务的新资源类型（比如图形处理单元或专用处理设备）。从 YARN 角度讲，ApplicationMaster 是用户代码，因此存在潜在的安全问题。YARN 假设 ApplicationMaster 存在错误或者甚至是恶意的，因此将它们当作无特权的代码对待。
1. 负责数据的切分；
2. 为应用程序申请资源并分配给内部的任务；
3. 任务的监控与容错。
*4. Container*
对任务运行环境进行抽象，封装CPU、内存等多维度的资源以及环境变量、启动命令等任务运行相关的信息。比如内存、CPU、磁盘、网络等，当AM向RM申请资源时，RM为AM返回的资源便是用Container表示的。YARN会为每个任务分配一个Container，且该任务只能使用该Container中描述的资源。要使用一个YARN集群，首先需要来自包含一个应用程序的客户的请求。ResourceManager协商一个容器的必要资源，启动一个ApplicationMaster 来表示已提交的应用程序。通过使用一个资源请求协议，ApplicationMaster协商每个节点上供应用程序使用的资源容器。执行应用程序时，ApplicationMaster监视容器直到完成。当应用程序完成时，ApplicationMaster从ResourceManager注销其容器，执行周期就完成了。



 


