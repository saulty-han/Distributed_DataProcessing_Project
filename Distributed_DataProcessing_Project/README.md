## 实验目的
<font style="color:rgb(0,0,0);">通过实验分析MapReduce 和 Spark 基于yarn的Spark 实现的 PageRank 算法在不同运行环境和数据集规模下的运行性能差异，重点对比迭代完成时间、CPU 利用率和内存消耗占比。</font>

## 实验方法
### 实验环境
#### 实验主机
我们实验使用四台ubuntu 2核4G云主机，在配置之前，需要完成主机的本机免密登陆和几台主机的免密互联。

在实现**主机免密登陆**时，需要生成本机公钥，并将公钥发给机器本身，修改授权文件所在的文件和文件夹的权限。

在实现**多台机器免密互联**中，首先需要对主机名字进行修改，并修改每台机器的 \etc\hosts 文件，需要使用云主机的私有ip地址，并标识每台主机的主机名。

```plain
172.21.96.207 ecnu01
172.21.96.206 ecnu02
172.21.96.209 ecnu03
172.21.96.208 ecnu04
```

将从节点和客户端上的公钥发送给主节点上的授权文件，再将主节点的授权文件发送给另外三个节点。

其次进行JAVA环境配置，使用jdk1.8，修改全局配置文件 /etc/profile来配置环境变量。



#### Hadoop配置
使用Hadoop-2.10.1来完成实验。

修改/etc/hadoop/slaves 中的localhost为ecnu02和ecnu03；

修改core-site.xml，设置ecnu01为主节点、主节点的namenode存放路径tmp；

修改hdfs-site.xml，设置dfs的备份数量、namenode和datanode存储元数据的本地文件系统目录；

修改Hadoop-env.sh，将JAVA_HOME修改为jdk1.8的地址。初始化和启动dfs后成功完成。

以上hadoop-2.10.1文件夹内容在四个云主机中均相同。



#### MapReduce配置
WEB_UI的所有端口都需要在安全组中添加。

在 hadoop-2.10.1/etc/hadoop/ 中新建mapred-site.xml文件（按照模版来）并修改，指定MapReduce的运行框架为yarn，配置主机对历史作业进程的查看端口，并指定作业运行中产生的临时文件的暂存位置；

修改yarn-site.xml，设置NodeManager上运行的附属服务为mapreduce_shuffle，设置每使用 1MB 物理内存最多可用的虚拟内存数为默认值2.1。

以上hadoop-2.10.1文件夹内容在四个云主机中均相同。



#### 基于Yarn的Spark配置
使用Spark-2.4.7完成实验。

修改spark-2.4.7/conf 中的spark-env.sh文件，配置Hadoop路径和主节点主机名及端口号；

修改slaves文件，设置从节点为ecnu02、ecnu03；

修改spark-defaults.conf文件，指定记录spark运行过程中的日志信息；

修改hadoop-2.10.1/conf/etc/hadoop 中的 yarn-site.xml，调整yarn的内存管理，关闭内存检查功能，避免任务因内存超限被杀死，并设置yarn作业日志的访问地址，方便查看作业的标准输出和标准错误日志。

以上spark-2.4.7文件夹内容在四个云主机中均相同。



#### 数据集
web_Google.txt		875713个网页节点，大小约为80MB

graph_dataset.txt	100001个网页节点，大小约为800MB



### 实验过程
#### **启动Yarn、HDFS和Spark服务**
主节点执行：

```plain
~/hadoop-2.10.1/sbin/start-yarn.sh

~/hadoop-2.10.1/sbin/mr-jobhistory-daemon.sh start historyserver
~/hadoop-2.10.1/sbin/start-dfs.sh

~/spark-2.4.7/sbin/start-all.sh
~/spark-2.4.7/sbin/start-history-server.sh
```



#### **步骤零：构建程序输入参数**
MapReduce：输入路径 输出路径 网页节点数

Spark：网页节点数 输入路径 输出路径



#### <font style="color:rgb(0,0,0);">步骤一：集中式和分布式运行性能对比</font>
1. <font style="color:rgb(0,0,0);">在单机环境中使用MapReduce 和 Spark 运行 PageRank（使用小数据集）</font>

在客户端执行：

```bash
cd ~/hadoop-2.10.1/

# 在MapReduce上运行小数据集
~/hadoop-2.10.1/bin/yarn jar ~/MapReduce.jar input/web_Google.txt output/ 875713

# 在基于yarn的spark上运行小数据集
~/spark-2.4.7/bin/spark-submit --master local \
--class cn.edu.ecnu.spark.examples.java.pagerank.PageRank \
/home/dase-dis/myApp/sparktest-1.0-SNAPSHOT.jar\
875713 \
~/dase-dis/spark_output/ \
/home/dase-dis/input/pagerank/web_Google.txt

```

2. <font style="color:rgb(0,0,0);">在分布式环境中的两个工作节点运行同样的程序（使用小数据集）：</font>

```bash
cd ~/hadoop-2.10.1/

# 在MapReduce上运行小数据集
~/hadoop-2.10.1/bin/yarn jar ~/MapReduce.jar input/web_Google.txt mapreduce_output_small/ 875713

# 在基于yarn的spark上运行小数据集
# 以client模式提交
~/spark-2.4.7/bin/spark-submit \
--deploy-mode client \
--master yarn \
--class cn.edu.ecnu.spark.examples.java.pagerank.PageRank \
875713 \
hdfs://ecnu01:9000/user/dase-dis/spark_output_small_1/ \
hdfs://ecnu01:9000/user/dase-dis/spark_input/web_Google.txt

# 以cluster模式提交
~/spark-2.4.7/bin/spark-submit \
--deploy-mode cluster \
--master yarn \
--class cn.edu.ecnu.spark.examples.java.pagerank.PageRank \
875713 \
hdfs://ecnu01:9000/user/dase-dis/spark_output_small_2/ \
hdfs://ecnu01:9000/user/dase-dis/spark_input/web_Google.txt
```



#### 步骤二：<font style="color:rgb(0,0,0);">不同数据规模在分布式环境下的对比</font>
<font style="color:rgb(0,0,0);">分别选择80MB和800MB规模的数据集，在MapReduce和Spark的分布式环境中运行 PageRank。</font>

运行小数据集的命令如步骤二所示，以下是运行大数据集的jar包提交命令：

```bash
cd ~/hadoop-2.10.1/

# 在MapReduce上运行大数据集
~/hadoop-2.10.1/bin/yarn jar ~/MapReduce.jar input/graph_dataset.txt mapreduce_output_big/ 100001

# 在基于yarn的spark上运行大数据集
# 以client模式提交
~/spark-2.4.7/bin/spark-submit \
--deploy-mode client \
--master yarn \
--class cn.edu.ecnu.spark.examples.java.pagerank.PageRank \
100001 \
hdfs://ecnu01:9000/user/dase-dis/spark_output_big_1/ \
hdfs://ecnu01:9000/user/dase-dis/spark_input/graph_dataset.txt

# 以cluster模式提交
~/spark-2.4.7/bin/spark-submit \
--deploy-mode cluster \
--master yarn \
--class cn.edu.ecnu.spark.examples.java.pagerank.PageRank \
100001 \
hdfs://ecnu01:9000/user/dase-dis/spark_output_big_2/ \
hdfs://ecnu01:9000/user/dase-dis/spark_input/graph_dataset.txt
```



#### 步骤三：<font style="color:rgb(0,0,0);">实时性能指标量化</font>
<font style="color:rgb(0,0,0);">记录程序开始运行到迭代结束的</font>**<font style="color:rgb(0,0,0);">总时间</font>**

<font style="color:rgb(0,0,0);">分别在两个工作节点上执行top命令每隔一段时间打印程序运行情况到`log`文件中，例如，执行以下命令即可每隔10s保存程序运行快照并写入cpu_usage.log文件中：</font>

```bash
top -b -d 10 > cpu_usage.log
```



#### **停止运行**
主节点执行：

```bash
~/hadoop-2.10.1/sbin/stop-yarn.sh
~/hadoop-2.10.1/sbin/mr-jobhistory-daemon.sh stop historyserver
~/hadoop-2.10.1/sbin/stop-dfs.sh

~/spark-2.4.7/sbin/stop-all.sh
~/spark-2.4.7/sbin/stop-history-server.sh
```



#### **删除输出文件**
```bash
./bin/hdfs dfs -rm -r <output>
```



## 实验结果
### 分布式部署下各节点执行进程的分析对比
#### 查看进程
![](https://cdn.nlark.com/yuque/0/2024/png/51590530/1734374995788-1e0649f8-05af-4331-8276-32b9bcb6a4e4.png) ![](https://cdn.nlark.com/yuque/0/2024/png/51590530/1734375012678-ea36f23b-a1cc-446a-bd30-6cb26dd0f359.png) 。

在YARN的Cluster模式和Client模式下，主要的区别在于Driver程序的运行位置和进程的分布。在**Cluster模式**中，Driver程序运行在YARN集群中的一个从节点上，资源管理和任务调度由ResourceManager负责，而执行任务的具体进程如YarnChild和MRAppMaster则在各个从节点上启动。客户端仅负责提交作业，不参与计算。在**Client模式**中，Driver程序运行在提交作业的客户端机器上，客户端不仅提交任务，还负责驱动作业的执行；而从节点上依然会启动YarnChild和MRAppMaster来执行具体任务。

两者的关键区别在于Cluster模式下，Driver程序是在从节点运行的，而Client模式中，Driver程序运行在客户端上。下面对Driver在Cluster模式和Client模式下运行的位置进行具体的比较。



#### **对比YARN Cluster和Client模式下，Driver运行的位置**
##### Cluster模式：
在YARN Cluster模式中，Driver运行在YARN集群中负责执行Application Master进程的从节点上（本程序运行在ecnu02节点上）。由于Driver运行在集群的从节点，而不是提交作业的客户端，因此在客户端无法显示Driver的详细输出，可以通过登录到该从节点的8042端口来查询作业执行过程中产生的stderr和stdout日志。  

![](https://cdn.nlark.com/yuque/0/2024/png/51590530/1734375483136-472cd808-052a-4ca7-8088-ee6a26d22b6a.png)

![](https://cdn.nlark.com/yuque/0/2024/png/51590530/1734375510191-e194a5ac-88e6-497d-89d2-2ee5cb96d82b.png)

![](https://cdn.nlark.com/yuque/0/2024/png/51590530/1734375524321-114204be-80fd-4bf5-b5ad-b92f5e02b137.png)

##### Client模式：
YARN Client 模式下，Driver运行在提交作业的客户端上，因此日志信息将直接在终端显示。

![](https://cdn.nlark.com/yuque/0/2024/png/51590530/1734375700784-65a4691c-7c20-48a8-8d2e-9857a1b7fc1e.png)

![](https://cdn.nlark.com/yuque/0/2024/png/51590530/1734375710016-b117eaec-4429-44de-b1d0-f2f976b4183a.png)



### 时间对比
| 时间 | 单机（数据集80MB） | 分布式（数据集80MB） | 分布式（数据集800MB） |
| --- | --- | --- | --- |
| <font style="color:#000000;">MapReduce</font> | 517s | 1099s | 7446s |
| <font style="color:#000000;">Spark</font> | 448s | 296s | 2295s |




#### 单机与分布式模式性能对比： 
+ MapReduce: 分布式比单机慢了约 2 倍，分布式的开销（任务调度、网络通信）超过了带来的并行处理优势 
+ Spark: 分布式比单机提高了 1.5 倍，分布式运行能充分利用集群资源来提高效率 

#### 分布式处理不同大小数据集对比： 
+ Spark在不同大小的数据集上，效率都优于MapReduce
+ MapReduce的性能瓶颈（磁盘I/O、网络带宽）随着数据集增加更显著，运行时间增加较大 



### %CPU对比
#### 80MB数据集单机与分布式CPU对比
![](https://cdn.nlark.com/yuque/0/2024/png/36182196/1734373416848-5dcf124f-4871-43d3-bd37-a3de4cd6f1c0.png)![](https://cdn.nlark.com/yuque/0/2024/png/36182196/1734373431671-94b58f6a-e626-495f-bf88-e894679e520b.png)

+ MapReduce单机vs分布式：在分布式环境下CPU利用率有所上升，但波动比单机更明显。分布式时在低负载（如3）表现出较低的资源利用，可能是由于某些节点闲置或任务不均衡导致。
+  Spark单机 vs 分布式：在分布式模式下CPU利用率大幅度提升，整体来说，Spark在处理任务时能够较好地利用CPU资源。 
+ MapReduce vs Spark：在单机和分布式情况下，Spark都优于MapReduce，在分布式情况下性能远超MapReduce，Spark有更稳定和高效的CPU利用率。

#### 分布式80MB和800MB CPU对比
数据集大小为80MB：

![](https://cdn.nlark.com/yuque/0/2024/png/36182196/1734373569658-762c661e-87be-4128-8705-aab66be272c2.png)

数据集大小为800MB：

![](https://cdn.nlark.com/yuque/0/2024/png/36182196/1734373593172-4ad386a2-ba09-450d-b5d0-e7475734864b.png)

+ MapReduce 80MB vs 800MB：在数据集增大后，MapReduce 能够充分利用分布式集群的资源，CPU利用率最高达到380%，平均170%左右，高于小数据集 
+ Spark 80MB vs 800MB：在两个数据集上，Spark都保持了稳定的CPU利用率（220%~320%之间）
+  MapReduce vs Spark：在 800MB 数据集上，MapReduce 的 CPU 利用率虽较高，但相较于 Spark，其整体的波动性较大，且 CPU 利用率在某些时刻表现出较低的效率（10%左右）



### %MEM对比
#### 80MB数据集单机与分布式MEM对比
数据集大小为80MB：

![](https://cdn.nlark.com/yuque/0/2024/png/36182196/1734373706747-96925fc6-9765-40a3-84bd-4ede541964ed.png)

数据集大小为800MB：

![](https://cdn.nlark.com/yuque/0/2024/png/36182196/1734373710649-aba75baa-3063-4357-bf67-fefd8586776c.png)

+ MapReduce 单机 vs 分布式：分布式环境下的内存使用增加，表现为任务在分布式节点之间的分配和协调增加了内存需求 
+ Spark 单机 vs 分布式：分布式环境下的内存使用增加 
+ MapReduce vs Spark：在单机和分布式时，Spark的内存使用都高于MapReduce。MapReduce较为波动，Spark更平稳，Spark 在分布式环境下利用更多的内存来处理数据并减少磁盘 I/O，从而实现了性能的提升

#### 分布式80MB和800MB MEM对比
数据集大小为80MB：

![](https://cdn.nlark.com/yuque/0/2024/png/36182196/1734373780868-cbef3f28-e0e7-4808-8743-f157e5422265.png)

数据集大小为800MB：

![](https://cdn.nlark.com/yuque/0/2024/png/36182196/1734373787969-21b2be71-1ba0-4e63-8701-c9fe34d19430.png)

+ MapReduce 80MB vs 800MB：随着数据量增加，MapReduce 的内存使用变得更加频繁且不稳定，数据处理中多个阶段的间歇性计算任务和中间存储会导致内存使用剧烈波动 
+ Spark 80MB vs 800MB：虽然数据量增加，但 Spark 依然维持较为平稳的内存使用（50MB左右) Map
+ Reduce vs Spark：MapReduce在较大数据集上有较为频繁的内存波动，这导致了虽然MapReduce的内存峰值大于Spark，但性能不如Spark



## 实验结论
+ Spark在Yarn的Cluster和Client模式下，Driver分别位于不同的节点内。Cluster下，位于运行选择的某一从节点内；Client下，位于提交作业的客户端内。
+ 进行相同大小的作业时，MapReduce 的执行过程分为多个阶段（Map、Shuffle、Reduce），并且每个阶段都需要进行磁盘IO，因此延迟较大，CPU利用率起伏不定；而Spark基于DAG的任务调度避免了大量读写，因此性能较高，也更稳定。
+ 在进行不同大小的作业时，CPU 利用率通常在适当规模的数据集上表现最佳，能够更充分地发挥系统资源的优势。过小的数据集可能导致资源浪费，而过大的数据集可能超出系统的处理能力。
+ 在单机集中式运行和分布式对比时，CPU利用率和MEM也会增加，说明系统资源得到了充分的利用。



## 实验分工
每个人在每项工作上都做出了不可磨灭的贡献。

蔡季妍：25%。负责实验环境配置、实验运行、实验结果整理分析、制作ppt，撰写实验报告。

谭冰：25%。负责实验代码打包、实验运行、实验结果整理分析、制作ppt，撰写实验报告。

谢林娜：25%。负责实验数据集获取、实验运行、实验结果整理分析、制作ppt，撰写实验报告。

李思琪：25%。负责实验环境配置、实验运行、实验结果整理分析、制作ppt，撰写实验报告。





