## 场景：
 上一篇详细说明了HDFS集群数据迁移的过程。如果阿里云ECS现在提供旧实例可以直接添加本地硬盘的服务（目前还不支持），而不用去申请新的实例。那问题就变成了HDFS单个实例内硬盘间的数据迁移。
 虽然目前的HDFS硬盘间的数据转移工作在社区比较成熟的解决方案，比如DiskBalancer（现在已经收入HDFS），它就是专门针对HDFS在单节点内使用多硬盘可能会导致硬盘使用不均的问题，通过统计出硬盘的占用率，进而进行数据迁移，以达到平衡，但是这样的解决方案不能直接应用到我们的场景。因为，1）我们需要进行整个硬盘数据的转移。2）DiskBalancerzhinengzai只能在相同介质的硬盘间转移数据，我们的ECS提供的硬盘类型往往非常多样。我们的场景需要一个新的解决方案。
 ## 方案
 ### HDFS数据在文件系统的组织
 我们都知道HDFS将数据按照默认的64M进行分块，并且对每块进行冗余备份存储。这些块实际存放的位置就是我们的配置文件里设置的dataNode的数据目录，以我的ECS服务器为例，数据存放的目录为
 /tmp/hadoop/tmp/dfs/data/current/BP-170152265-10.30.120.9-1499405595883/current/finalized/subdir0/subdir1/
 其中/tmp/hadoop/tmp/dfs/data/为用户自定义，后面的为hadoop自动创建。HDFS的成百上千的数据块就是以这样的形式组织的。这个组织结构被NameNode保存在内存中，当客户端查询或者修改数据的的时候，NameNode能高效的检索到数据块的分布。
 ### 数据迁移
 当我们添加了一块新的盘之后，将它挂载到文件目录，并且添加到dataNode的数据目录旧可以存放HDFS数据。当我们需要把旧盘的数据转移到新盘的时候，只需要将DataNode关闭，手动将/tmp/hadoop/tmp/dfs/data/muluia目录下的所有数据一级级目录都拷贝到新的目录下，需要注意的是，新旧盘的目录一定要保持完全一致，比如：
旧盘：/tmp/hadoop/tmp/dfs/data/current/BP-170152265-10.30.120.9-1499405595883/current/finalized/subdir0/subdir1/
新盘：/tmp/hadoop2/tmp/dfs/data/current/BP-170152265-10.30.120.9-1499405595883/current/finalized/subdir0/subdir1/
不然，DataNode就无法定位数据块了。
当所有的节点都进行了同样的操作后，将旧盘的目录从配置文件中去掉，重启所有的DataNode。
## 实验
实例配置：     
ECS规格：ecs.i1.xlarge      
CPU：4核      
内存：16G    
系统盘：高效云盘40G     
数据盘1：本地硬盘100G     
数据盘2：普通云盘100G
OS：CentOS 6.8 64位      
Hadoop版本：2.6.4 
集群规模：4

### 初始状态
最开始集群上有4个DataNode，node6-node9，其中node6上启动NameNode。每个datanode只挂载一个数据盘，如下为HDFS的初始状态：
```
17/07/07 13:35:11 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Configured Capacity: 439131176960 (408.97 GB)
Present Capacity: 416539900780 (387.93 GB)
DFS Remaining: 414368399360 (385.91 GB)
DFS Used: 2171501420 (2.02 GB)
DFS Used%: 0.52%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Live datanodes (4):

Name: 10.30.120.9:50010 (node6)
Hostname: node6
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 1085726134 (1.01 GB)
Non DFS Used: 5650424394 (5.26 GB)
DFS Remaining: 103046643712 (95.97 GB)
DFS Used%: 0.99%
DFS Remaining%: 93.86%
Last contact: Fri Jul 07 13:35:11 CST 2017


Name: 10.30.120.12:50010 (node7)
Hostname: node7
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 544660890 (519.43 MB)
Non DFS Used: 5647237734 (5.26 GB)
DFS Remaining: 103590895616 (96.48 GB)
DFS Used%: 0.50%
DFS Remaining%: 94.36%
Last contact: Fri Jul 07 13:35:11 CST 2017


Name: 10.30.120.22:50010 (node8)
Hostname: node8
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 270557198 (258.02 MB)
Non DFS Used: 5646807026 (5.26 GB)
DFS Remaining: 103865430016 (96.73 GB)
DFS Used%: 0.25%
DFS Remaining%: 94.61%
Last contact: Fri Jul 07 13:35:11 CST 2017


Name: 10.31.0.18:50010 (node9)
Hostname: node9
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 270557198 (258.02 MB)
Non DFS Used: 5646807026 (5.26 GB)
DFS Remaining: 103865430016 (96.73 GB)
DFS Used%: 0.25%
DFS Remaining%: 94.61%
Last contact: Fri Jul 07 13:35:11 CST 2017
```
### 挂载新盘
然后挂载新的硬盘，并添加到HDFS目录，在HDFS的配置文件添加如下内容，这是HDFS挂载多个硬盘的通用方法：

```
           <property>
　　　　　　　　<name>dfs.datanode.data.dir</name>
　　　　　　　　<value>/tmp/hadoop/tmp/dfs/data,/tmp/hadoop2/tmp/dfs/data</value>
　　　　　　</property>
```
重启DataNode,可以看到HDFS集群的存储空间变成了800G
```
17/07/07 13:39:40 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Safe mode is ON
Configured Capacity: 861351968768 (802.20 GB)
Present Capacity: 817040698220 (760.93 GB)
DFS Remaining: 814869098496 (758.91 GB)
DFS Used: 2171599724 (2.02 GB)
DFS Used%: 0.27%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Live datanodes (4):

Name: 10.30.120.22:50010 (node8)
Hostname: node8
Decommission Status : Normal
Configured Capacity: 215337992192 (200.55 GB)
DFS Used: 270581774 (258.05 MB)
Non DFS Used: 11077251058 (10.32 GB)
DFS Remaining: 203990159360 (189.98 GB)
DFS Used%: 0.13%
DFS Remaining%: 94.73%
Last contact: Fri Jul 07 13:39:41 CST 2017


Name: 10.30.120.9:50010 (node6)
Hostname: node6
Decommission Status : Normal
Configured Capacity: 215337992192 (200.55 GB)
DFS Used: 1085750710 (1.01 GB)
Non DFS Used: 11079496266 (10.32 GB)
DFS Remaining: 203172745216 (189.22 GB)
DFS Used%: 0.50%
DFS Remaining%: 94.35%
Last contact: Fri Jul 07 13:39:41 CST 2017


Name: 10.30.120.12:50010 (node7)
Hostname: node7
Decommission Status : Normal
Configured Capacity: 215337992192 (200.55 GB)
DFS Used: 544685466 (519.45 MB)
Non DFS Used: 11077272166 (10.32 GB)
DFS Remaining: 203716034560 (189.73 GB)
DFS Used%: 0.25%
DFS Remaining%: 94.60%
Last contact: Fri Jul 07 13:39:41 CST 2017


Name: 10.31.0.18:50010 (node9)
Hostname: node9
Decommission Status : Normal
Configured Capacity: 215337992192 (200.55 GB)
DFS Used: 270581774 (258.05 MB)
Non DFS Used: 11077251058 (10.32 GB)
DFS Remaining: 203990159360 (189.98 GB)
DFS Used%: 0.13%
DFS Remaining%: 94.73%
Last contact: Fri Jul 07 13:39:41 CST 2017
```
可以看到每个DataNode的Configured Capacity变成了200G,但是总的数据量以及单个DataNode的数量都没变。再看看两个盘的实际使用率：
```
node6：
Filesystem     1K-blocks    Used Available Use% Mounted on
/dev/vda1       41152832 2454900  36600832   7% /
tmpfs            8166980       0   8166980   0% /dev/shm
/dev/vdb       107209760 1123560 100633608   2% /tmp/hadoop
/dev/vdd       103081248   61092  97777276   1% /tmp/hadoop2
-------------------------------------------------------------
node7
Filesystem     1K-blocks    Used Available Use% Mounted on
/dev/vda1       41152832 2114092  36941640   6% /
tmpfs            8166980       0   8166980   0% /dev/shm
/dev/vdb       107209760  593004 101164164   1% /tmp/hadoop
/dev/vde       103081248   61092  97777276   1% /tmp/hadoop2
-------------------------------------------------------------
node8
Filesystem     1K-blocks    Used Available Use% Mounted on
/dev/vda1       41152832 2114324  36941408   6% /
tmpfs            8166980       0   8166980   0% /dev/shm
/dev/vdb       107209760  325304 101431864   1% /tmp/hadoop
/dev/vde       103081248   61092  97777276   1% /tmp/hadoop2
-------------------------------------------------------------
node9
Filesystem     1K-blocks    Used Available Use% Mounted on
/dev/vda1       41152832 2113452  36942280   6% /
tmpfs            8166980       0   8166980   0% /dev/shm
/dev/vdb       107209760  325304 101431864   1% /tmp/hadoop
/dev/vdd       103081248   61092  97777276   1% /tmp/hadoop2
```
新加入的盘挂载在/tmp/hadoop2，目前没有存放数据。

### 拷贝数据
开始拷贝数据，将旧盘挂载目录的所有数据拷贝到新盘相同的目录：
```
mkdir -p /tmp/hadoop2/tmp/dfs/data/
cp -r /tmp/hadoop/tmp/dfs/data/current/  /tmp/hadoop2/tmp/dfs/data/
```
然后重启DataNode，查看各节点的存储空间如下：
```
Configured Capacity: 422220791808 (393.22 GB)
Present Capacity: 400495663980 (372.99 GB)
DFS Remaining: 398324162560 (370.97 GB)
DFS Used: 2171501420 (2.02 GB)
DFS Used%: 0.54%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Live datanodes (4):

Name: 10.30.120.22:50010 (node8)
Hostname: node8
Decommission Status : Normal
Configured Capacity: 105555197952 (98.31 GB)
DFS Used: 270557198 (258.02 MB)
Non DFS Used: 5431271410 (5.06 GB)
DFS Remaining: 99853369344 (93.00 GB)
DFS Used%: 0.26%
DFS Remaining%: 94.60%
Last contact: Fri Jul 07 13:49:42 CST 2017


Name: 10.30.120.9:50010 (node6)
Hostname: node6
Decommission Status : Normal
Configured Capacity: 105555197952 (98.31 GB)
DFS Used: 1085726134 (1.01 GB)
Non DFS Used: 5431300682 (5.06 GB)
DFS Remaining: 99038171136 (92.24 GB)
DFS Used%: 1.03%
DFS Remaining%: 93.83%
Last contact: Fri Jul 07 13:49:42 CST 2017


Name: 10.30.120.12:50010 (node7)
Hostname: node7
Decommission Status : Normal
Configured Capacity: 105555197952 (98.31 GB)
DFS Used: 544660890 (519.43 MB)
Non DFS Used: 5431284326 (5.06 GB)
DFS Remaining: 99579252736 (92.74 GB)
DFS Used%: 0.52%
DFS Remaining%: 94.34%
Last contact: Fri Jul 07 13:49:42 CST 2017


Name: 10.31.0.18:50010 (node9)
Hostname: node9
Decommission Status : Normal
Configured Capacity: 105555197952 (98.31 GB)
DFS Used: 270557198 (258.02 MB)
Non DFS Used: 5431271410 (5.06 GB)
DFS Remaining: 99853369344 (93.00 GB)
DFS Used%: 0.26%
DFS Remaining%: 94.60%
Last contact: Fri Jul 07 13:49:42 CST 2017
```
可以看到Configured Capacitybianw变为了400G,数据全部从旧盘转移到了新盘，而且数据分布也未发生任何变化。到此为止，我们就完成了数据的迁移。由于这种单实例内硬盘数据的迁移的方式并不会影响数据的命名空间和索引，所以NameNode不需要任何变动。
### 总结
1、相比于HDFS的集群迁移，硬盘间间的数据迁移过程相对简单，因为在HDFS看来，同一个实例不同的硬盘存放的数据属于同一个索引路径。只要数据块不在不同实例间转移，数据块的移动就不会影响索引。所以，我们能够采用直接拷贝的方式完成硬盘间的数据迁移。
2、我们迁移实验的后是先添加新的硬盘到HDFS集群，然后再从旧硬盘拷贝到新硬盘，然后再删除旧硬盘。这样做的目的是为了说明，说明HDFS挂载多个硬盘方式。如果只是单纯的迁移，只需要三部：
停止DataNode
拷贝旧硬盘的数据到新硬盘，并更换HDFS配置文件的数据目录
重启DataNode

