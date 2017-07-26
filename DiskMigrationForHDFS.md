# 开源HDFS硬盘间数据迁移工具
## 1 开发背景
### 1.1 场景
用户已经将HDFS的数据存放在一定数量的磁盘上，现在加入了新的一批磁盘，想把旧的磁盘上的数据全部迁移到新的硬盘上，比如阿里云ECS用户已经基于云盘搭建了HDFS的集群，现在想转移到性能更好、成本更低的本地硬盘上来。（注：我们的工具不仅仅面向云盘往本地盘迁移的场景，而是通用的HDFS硬盘间数据迁移工具）。    
以下面的场景为例：
![hdfs11.PNG](https://github.com/liumihust/gitTset/blob/master/hdfs11.PNG) 
我们想把旧盘的数据全部迁移到新加入的盘。目前的HDFS还不支持该功能，正如上篇博客所介绍那样，手动将目录树对应起来拷贝是一种解决方案，但是它有如下问题：    
1）前后冗长的目录树要完全一致，至少稍有偏差就会导致DataNode找不到数据块。     
2）只能一对一拷贝，也就是对等拷贝，拷贝前后每个DataNode的盘数要一致    
3）手动操作过多，麻烦。     
针对以上这些问题，我最近研究了下HDFS源码，然后开发出了一个HDFS硬盘间数据迁移的工具，只用在HDFS客户端输入一条命令行，HDFS就可以在后台完成数据迁移。值得一提的是，我们的迁移还可以保持数据分布的均衡，不仅仅解决一对一迁移，还解决了多对一、一对多、多对多等复杂迁移场景。
### 1.2 期望的效果
对于上面的例子，我们的工具可以达到的效果接近如下：
![hdfs12](https://github.com/liumihust/gitTset/blob/master/hdfs12.PNG) 
## 2 技术细节
我们的迁移过程分为两个阶段：计划阶段和执行阶段。
### 2.1 计划阶段（plan）
所谓计划阶段就是，根据现有的硬盘的实际情况，规划出如1.2的迁移计划。整个计划可以分解为一个个Step，每个Step负责将一定数量的某个源硬盘的数据迁移到目标盘，每个Step的数据迁移量都是工具自动计算详见2.1.1。Plan和Step大致的关系如下所示：
![hdfs20](https://github.com/liumihust/gitTset/blob/master/hdfs20.PNG) 
#### 2.1.1 迁移量的计算
当有多个新硬盘的时候，我们为了保证迁移后的数据分布保持平衡，我们在每一步（step）迁移操作的时候，都会按照硬盘的空间进行分配接收的数据量，这样全部迁移结束后，每个新硬盘的数据占用率都是均等的。具体计算如下：
![hdfs21](https://github.com/liumihust/gitTset/blob/master/hdfs21.PNG) 
这种分配方式可以确保最后的数据分布更均衡：
![hdfs22](https://github.com/liumihust/gitTset/blob/master/hdfs22.PNG) 
我们计算出这样的一个规划后，将该plan序列化成json的格式持久化保存到文件系统里面，为下一阶段做准备。
### 2.2 执行阶段（execute）
执行阶段就是将上一部生成的json读进来并反序列化成plan对象，将里面的每一个step生成对应的工作流，以此按照step的顺序执行，执行完毕就完成了所有的数据的迁移。这一阶段的流程图如下：
![hdfs23](https://github.com/liumihust/gitTset/blob/master/hdfs23.PNG) 
### 2.3 执行并行优化
2.2 的串行执行是最简单也最安全的方式。在HDFS里面同一个盘不能同时进行并发操作，因为不论是取数据块还是存放数据块都是通过迭代器实现的，一旦出现并发copy in或者copy out就会导致迭代器出错。所以目前的HDFS里面还没有并行拷贝的机制。
但是磁盘数据的拷贝往往就是针对多盘的场景，在多盘的场景下，没有冲突的“盘对”（Volume Pair）是可以独立进行操作的，只是在运行时发掘这样的并行特征并且能够低开销的调度有一定的难度。2.2的流水线Step中就存在可以并行执行的部分，我们的工作就是挖掘出这一系列Step中可以并行的部分，时刻做到最大的并行化，提高数据迁移的效率（这工作有点类似于程序模块间的并行化）。
![parallel1.PNG](https://github.com/liumihust/gitTset/blob/master/parallel1.PNG) 
### 2.4 开源地址
将工作分为两个子部分进行开源，因为磁盘迁移是特定场景下的需求，更像是一种工具，而并行化执行工作是针对HDFS一种通用的改进。     
磁盘迁移：https://github.com/liumihust/Disk-Migration-For-HDFS         
并行化执行：https://github.com/liumihust/Parallel-Block-Copy-HDFS
## 3 阿里云ECS实验
### 3.1 实验环境
ECS规格：ecs.i1.xlarge      
CPU：4核      
内存：16G    
系统盘：高效云盘40G     
数据盘1：普通云盘100G     
数据盘2：普通云盘100G       
数据盘3：普通云盘100G     
数据盘4：普通云盘100G       
OS：CentOS 6.8 64位     
### 3.2 实验一（One To One）
#### 3.2.1 初始状态
最开始只挂载了一个数据盘，约100G，如下：
```
Configured Capacity: 105555197952 (98.31 GB)
Present Capacity: 100122657936 (93.25 GB)
DFS Remaining: 95219621888 (88.68 GB)
DFS Used: 4903036048 (4.57 GB)
DFS Used%: 4.90%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Missing blocks (with replication factor 1): 0
Pending deletion blocks: 0

-------------------------------------------------
Live datanodes (1):

Name: 10.30.209.220:9866 (node1)
Hostname: node1
Decommission Status : Normal
Configured Capacity: 105555197952 (98.31 GB)
DFS Used: 4903036048 (4.57 GB)
Non DFS Used: 63830896 (60.87 MB)
DFS Remaining: 95219621888 (88.68 GB)
DFS Used%: 4.64%
DFS Remaining%: 90.21%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 22 12:02:25 CST 2017
Last Block Report: Sat Jul 22 11:53:13 CST 2017
```
HDFS里面存放了16个原始数据，共4.57G：
```
Found 16 items
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:56 /liumi/data1
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:57 /liumi/data10
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:57 /liumi/data11
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:57 /liumi/data12
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:57 /liumi/data13
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:57 /liumi/data14
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:58 /liumi/data15
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:58 /liumi/data16
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:56 /liumi/data2
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:56 /liumi/data3
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:56 /liumi/data4
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:56 /liumi/data5
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:56 /liumi/data6
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:56 /liumi/data7
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:56 /liumi/data8
-rw-r--r--   1 root supergroup  304062704 2017-07-22 11:57 /liumi/data9
```
#### 3.2.2 挂载新的盘
我们添加一块新的数据盘HDFS,
```
Configured Capacity: 211110395904 (196.61 GB)
Present Capacity: 200246804480 (186.49 GB)
DFS Remaining: 195343536128 (181.93 GB)
DFS Used: 4903268352 (4.57 GB)
DFS Used%: 2.45%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Missing blocks (with replication factor 1): 0
Pending deletion blocks: 0

-------------------------------------------------
Live datanodes (1):

Name: 10.30.209.220:9866 (node1)
Hostname: node1
Decommission Status : Normal
Configured Capacity: 211110395904 (196.61 GB)
DFS Used: 4903268352 (4.57 GB)
Non DFS Used: 126173184 (120.33 MB)
DFS Remaining: 195343536128 (181.93 GB)
DFS Used%: 2.32%
DFS Remaining%: 92.53%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 22 12:04:12 CST 2017
Last Block Report: Sat Jul 22 12:04:00 CST 2017
```
总的可用空间增加到了200G，但是新的盘没有数据：
```
[root@node1 hdfs]# df
Filesystem     1K-blocks    Used Available Use% Mounted on
/dev/vda1       41152832 8457916  30597816  22% /
tmpfs            4030552       0   4030552   0% /dev/shm
/dev/vdb       103081248 4850460  92987908   5% /tmp/hadoop1
/dev/vdc       103081248   61100  97777268   1% /tmp/hadoop2
```
下面就开始迁移了。
#### 3.2.3 硬盘数据迁移
##### 3.2.3.1 生成计划
在HDFS的客户端输入一下命令：
```
hdfs diskbalancer -plan node1 -type diskMigrate 
```
终端会输出计划存放的目录，记住这个json文件的目录，下一步要用到
##### 3.2.3.2 执行计划
输入如下命令：
```
hdfs diskbalancer -execute [json文件的目录]
```
后台就开始了数据迁移的工作，通过df命令就可以看到硬盘数据已经开始在变化，通过下面的命令可以查看迁移是否正式结束：
```
hdfs diskbalancer -query node1:9867
```
如果输出为
```
Result: PLAN_DONE
```
表示迁移结束。
查看迁移后的HDFS数据：
```
Configured Capacity: 211110395904 (196.61 GB)
Present Capacity: 200246788096 (186.49 GB)
DFS Remaining: 195343343616 (181.93 GB)
DFS Used: 4903444480 (4.57 GB)
DFS Used%: 2.45%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Missing blocks (with replication factor 1): 60
Pending deletion blocks: 0

-------------------------------------------------
Live datanodes (1):

Name: 10.30.209.220:9866 (node1)
Hostname: node1
Decommission Status : Normal
Configured Capacity: 211110395904 (196.61 GB)
DFS Used: 4903444480 (4.57 GB)
Non DFS Used: 126189568 (120.34 MB)
DFS Remaining: 195343343616 (181.93 GB)
DFS Used%: 2.32%
DFS Remaining%: 92.53%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 22 16:36:36 CST 2017
Last Block Report: Sat Jul 22 16:23:39 CST 2017
```
数据没有发生变化，再查看硬盘数据：
```
[root@node1 logs]# df
Filesystem     1K-blocks    Used Available Use% Mounted on
/dev/vda1       41152832 9341976  29713756  24% /
tmpfs            4030552       0   4030552   0% /dev/shm
/dev/vdb       103081248   62204  97776164   1% /tmp/hadoop1
/dev/vdc       103081248 4849548  92988820   5% /tmp/hadoop2
```
数据已经从第一个硬盘全部迁移到了第二个盘，第一个盘现在为空的！
至此迁移任务已经完成。
***************************************************************************************************************************************************************************************************************************************************************
当然到了这里我们也可以将数据从第二个盘全部迁移回第一个盘，重复前面的两个指令，执行完毕查看，结果如下：
```
[root@node1 logs]# df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/vda1       41152832 11719568  27336164  31% /
tmpfs            4030552        0   4030552   0% /dev/shm
/dev/vdb       103081248  4851336  92987032   5% /tmp/hadoop1
/dev/vdc       103081248    61108  97777260   1% /tmp/hadoop2
```
### 3.3 实验二（One To Multi）
我们的工具是通用型的，实验一只是通过一对一迁移案例对工具的使用进行说明，其实对于一对多、多对一、多对多的等复杂场景都适合。所以我们再添加一个一对多的场景。
#### 3.3.1 初始状态
新加入两个硬盘，都为空
```
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/vda1       41152832 14572020  24483712  38% /
tmpfs            4030552        0   4030552   0% /dev/shm
/dev/vdb       103081248  4850868  92987500   5% /tmp/hadoop1
/dev/vdc       103081248    61108  97777260   1% /tmp/hadoop2
/dev/vdd       103081248    61100  97777268   1% /tmp/hadoop3
```
#### 3.3.2 迁移
使用实验一完全一致的命令，迁移后，第一个盘数据全部被迁移到了新加入的两个盘，并且是均衡的。如下:
```
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/vda1       41152832 14572088  24483644  38% /
tmpfs            4030552        0   4030552   0% /dev/shm
/dev/vdb       103081248    62400  97775968   1% /tmp/hadoop1
/dev/vdc       103081248  2455276  95383092   3% /tmp/hadoop2
/dev/vdd       103081248  2455464  95382904   3% /tmp/hadoop3
```
### 3.4 实验三（Multi To Multi）
多对多场景的测试
#### 3.4.1 初始态
这次是共有四块硬盘，其中两个为旧盘，新加入两个盘：
```
[root@node1 hadoop]# df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/vda1       41152832 20989336  18066396  54% /
tmpfs            4030552        0   4030552   0% /dev/shm
/dev/vdb       103081248    61108  97777260   1% /tmp/hadoop1
/dev/vdc       103081248    61108  97777260   1% /tmp/hadoop2
/dev/vdd       103081248  2824728  95013640   3% /tmp/hadoop3
/dev/vde       103081248  3282928  94555440   4% /tmp/hadoop4
```
#### 3.4.2 迁移
使用实验一完全一致的命令，数据开始迁移，在执行期间可以查看迁移进度：
```
[root@node1 hadoop]# df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/vda1       41152832 20989360  18066372  54% /
tmpfs            4030552        0   4030552   0% /dev/shm
/dev/vdb       103081248  1689556  96148812   2% /tmp/hadoop1
/dev/vdc       103081248  1656160  96182208   2% /tmp/hadoop2
/dev/vdd       103081248  2824736  95013632   3% /tmp/hadoop3
/dev/vde       103081248    61124  97777244   1% /tmp/hadoop4
```
可以看到盘4将数据平均分给了盘1、盘2，即Step1、Step2完成，Step3、Step4正在执行。      
全部迁移结束后，盘3、盘4的数据全部迁移到了盘1、盘2：
```
[root@node1 hadoop]# df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/vda1       41152832 20989356  18066376  54% /
tmpfs            4030552        0   4030552   0% /dev/shm
/dev/vdb       103081248  3185896  94652472   4% /tmp/hadoop1
/dev/vdc       103081248  2923240  94915128   3% /tmp/hadoop2
/dev/vdd       103081248    61116  97777252   1% /tmp/hadoop3
/dev/vde       103081248    61116  97777252   1% /tmp/hadoop4
```
可以看到，迁移后数据分布非常均匀，和预想的一样。

### 3.5 实验四（并行化执行）
#### 3.5.1 初始状态
四个盘，前两个有数据，后两个为新加入的盘，没有数据。
```
[root@node1 hadoop]# df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/vda1       41152832 20989356  18066376  54% /
tmpfs            4030552        0   4030552   0% /dev/shm
/dev/vdb       103081248  3185896  94652472   4% /tmp/hadoop1
/dev/vdc       103081248  2923240  94915128   3% /tmp/hadoop2
/dev/vdd       103081248    61116  97777252   1% /tmp/hadoop3
/dev/vde       103081248    61116  97777252   1% /tmp/hadoop4
```
#### 3.5.2 迁移数据
```
[root@node1 hadoop]# df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/vda1       41152832 20990328  18065404  54% /
tmpfs            4030552        0   4030552   0% /dev/shm
/dev/vdb       103081248  2886628  94951740   3% /tmp/hadoop1
/dev/vdc       103081248  2490408  95347960   3% /tmp/hadoop2
/dev/vdd       103081248   493376  97344992   1% /tmp/hadoop3
/dev/vde       103081248   393824  97444544   1% /tmp/hadoop4

[root@node1 hadoop]# df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/vda1       41152832 20990504  18065228  54% /
tmpfs            4030552        0   4030552   0% /dev/shm
/dev/vdb       103081248  2155992  95682376   3% /tmp/hadoop1
/dev/vdc       103081248  1724704  96113664   2% /tmp/hadoop2
/dev/vdd       103081248  1391180  96447188   2% /tmp/hadoop3
/dev/vde       103081248  1196284  96642084   2% /tmp/hadoop4
```
可以看到, 四个盘同时在迁移数据
#### 3.5.3 结果
```
[root@node1 hadoop]# df
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/vda1       41152832 20991196  18064536  54% /
tmpfs            4030552        0   4030552   0% /dev/shm
/dev/vdb       103081248  1724624  96113744   2% /tmp/hadoop1
/dev/vdc       103081248  1592604  96245764   2% /tmp/hadoop2
/dev/vdd       103081248  1390288  96448080   2% /tmp/hadoop3
/dev/vde       103081248  1522388  96315980   2% /tmp/hadoop4
```
每个盘数据接近均等。









