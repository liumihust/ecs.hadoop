# HDFS集群数据迁移

## 场景：
用户已经租用了一定数量的阿里云ECS实例，并搭建了Hadoop或者Spark集群。以前ECS实例的硬盘都是采用云盘，相比如本地硬盘，云盘容错好但是带宽低。而Hadoop和Spark等大数据平台往往对硬盘IO要求很高，另一方面它们底层的存储都是采用HDFS，HDFS本身具有容错机制，因而对于这类的大数据平台采用本地硬盘会是一个更佳的选择。于是，阿里云ECS也新推出了本地硬盘实例。   
但是目前的本地硬盘不能直接添加到用户已有的ECS实例，而需要购买新的基于本地硬盘的实例。对于已经搭建了Hadoop或者Spark平台的用户来说，想要更新到最新的基于本地硬盘的实例，硬盘数据的迁移是一个非常关键的过程。   
于是，针对HDFS的硬盘数据迁移，我们做了深入的探究。下面是目前已经通过实验证实可行的迁移方案。

## 方案： 
### 1 DataNode迁移
#### 1.1 数据迁移
将新增的基于本地硬盘的实例（后面统称新实例）添加到现有的集群，新旧混合，然后创建一个文件oldInstances，记录所有旧dataNode的ip地址。放到自定义目录，比如$HADOOP_HOME$/etc/hadoop/。然后修改配置文件hdfs-site.xml，添加如下内容：
```
           <property>
               <name>dfs.hosts.exclude</name>
               <value>$HADOOP_HOME/etc/hadoop/oldInstances</value>
               <final>true</final>
           </property>
```

将修改后的配置文件及oldInstances分发到集群所有实例对应的目录。

在NameNode上执行:
```
hdfs dfsadmin –refreshNodes
```
HDFS会自动读取oldInstances里面所有的hostname，并将这些节点的dataNode数据迁移到其他新的的DataNode上。全部迁移结束后，这些旧的实例将处于Decommissioned状态，不参与HDFS的储存，可以直接杀死。这样就实现了数据从旧实例到新实例的迁移。值得说明的是，迁移后的旧实例的DataNode的数据并不会改变，这样，在迁移过程中出现任何差错，都不会导致原数据丢失。   
注：从新实例往旧实例迁移的时候，一定要切记，新实例的数量要大于当前HDFS所设置的副本数，默认为3，不然会迁移失败。这是HDFS目前尚未fix的一个issue，主要原因是，这种情况很极端，实际场景很少遇到，如果用户遇到了可以尝试先减少配置文件里设置的副本数，再迁移。详见:https://issues.apache.org/jira/browse/HDFS-1590

#### 1.2 Re-balance
考虑到迁移后的数据分布不一定均衡，可以用HDFS自带的均衡器进行re-balance：   
由于HDFS考虑到re-balance的IO可能会影响应用，所以默认的IO带宽限制得很低，我们也可以提升，
```
hdfs dfsadmin –setBalancerBandwidth 104857600 //比如100M   
```
开始re-balance，   
```
$HADOOP_HOME/sbin/start-balancer.sh -threshold 5 //5表示dataNode之间的硬盘占用率之差不超过5%。   
```
re-balance进程会在后台一直运行，直到达到用户要求的平衡阈值。

#### 1.3 阿里云ECS实例实验
ECS上最初有三台服务器，node1、node2、node3构成一个集群，其中node1为NameNode，其余为DataNode，数据块的副本数为2。   
我们在HDFS存储2.77G的数据，考虑到副本，实际的数据量应该是两倍，5.54G。如下所示，HDFS将数据均匀地分布在两个DataNode。
```
-------------------------------------------------------
Live datanodes (2):

Name: 10.30.209.243:50010 (node2)
Hostname: node2
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991955968 (2.79 GB)
Non DFS Used: 4106268672 (3.82 GB)
DFS Remaining: 35042275328 (32.64 GB)
DFS Used%: 7.10%
DFS Remaining%: 83.16%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:24:26 CST 2017


Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991955968 (2.79 GB)
Non DFS Used: 4070301696 (3.79 GB)
DFS Remaining: 35078242304 (32.67 GB)
DFS Used%: 7.10%
DFS Remaining%: 83.24%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:24:26 CST 2017
```
接下来，我们在ECS再申请两个新实例node4、node5作为dataNode，同时把node1也添加为DataNode，一开始三个新的DataNode没有数据。接下来按照前面的迁移操作，完成后的结果如下所示：

```
-------------------------------------------------------
Live datanodes (5):

Name: 10.30.209.242:50010 (node4)
Hostname: node4
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 1754566896 (1.63 GB)
Non DFS Used: 4011880208 (3.74 GB)
DFS Remaining: 36374052864 (33.88 GB)
DFS Used%: 4.16%
DFS Remaining%: 86.32%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:31:38 CST 2017


Name: 10.30.209.243:50010 (node2)
Hostname: node2
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991955968 (2.79 GB)
Non DFS Used: 4106334208 (3.82 GB)
DFS Remaining: 35042209792 (32.64 GB)
DFS Used%: 7.10%
DFS Remaining%: 83.16%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:31:38 CST 2017


Name: 10.30.210.52:50010 (node5)
Hostname: node5
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2243908468 (2.09 GB)
Non DFS Used: 4230229132 (3.94 GB)
DFS Remaining: 35666362368 (33.22 GB)
DFS Used%: 5.32%
DFS Remaining%: 84.64%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:31:38 CST 2017


Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991955968 (2.79 GB)
Non DFS Used: 4070334464 (3.79 GB)
DFS Remaining: 35078209536 (32.67 GB)
DFS Used%: 7.10%
DFS Remaining%: 83.24%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:31:38 CST 2017


Name: 10.30.209.220:50010 (node1)
Hostname: node1
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 1985286012 (1.85 GB)
Non DFS Used: 5325684868 (4.96 GB)
DFS Remaining: 34829529088 (32.44 GB)
DFS Used%: 4.71%
DFS Remaining%: 82.65%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:31:38 CST 2017

```
不难看出，node2、node3的所有数据块都转移到了node1、node4、node5(注：node2、node3自身数据不会变)，且node2、node3都处于Decommissioned状态。它们将不再参与存储，我们继续验证，再向HDFS存入707M的数据，结果如下：
```
------------------------------------------------------
Live datanodes (5):

Name: 10.30.209.242:50010 (node4)
Hostname: node4
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2096789095 (1.95 GB)
Non DFS Used: 4012870041 (3.74 GB)
DFS Remaining: 36030840832 (33.56 GB)
DFS Used%: 4.98%
DFS Remaining%: 85.50%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:37:26 CST 2017


Name: 10.30.209.243:50010 (node2)
Hostname: node2
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991955968 (2.79 GB)
Non DFS Used: 4106346496 (3.82 GB)
DFS Remaining: 35042197504 (32.64 GB)
DFS Used%: 7.10%
DFS Remaining%: 83.16%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:32:35 CST 2017


Name: 10.30.210.52:50010 (node5)
Hostname: node5
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2649776149 (2.47 GB)
Non DFS Used: 4231405547 (3.94 GB)
DFS Remaining: 35259318272 (32.84 GB)
DFS Used%: 6.29%
DFS Remaining%: 83.67%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:37:26 CST 2017


Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991955968 (2.79 GB)
Non DFS Used: 4070346752 (3.79 GB)
DFS Remaining: 35078197248 (32.67 GB)
DFS Used%: 7.10%
DFS Remaining%: 83.24%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:32:53 CST 2017


Name: 10.30.209.220:50010 (node1)
Hostname: node1
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2733319804 (2.55 GB)
Non DFS Used: 5326780804 (4.96 GB)
DFS Remaining: 34080399360 (31.74 GB)
DFS Used%: 6.49%
DFS Remaining%: 80.87%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 15:37:26 CST 2017
```
node2、node3的数据没有发生任何变化，只有node1、node4、node5的数据在增加，事实上node2、node3已经可以直接kill掉了。
到此为止，我们已经实现了旧实例的硬盘数据往新实例硬盘的迁移。



### 2 NameNode迁移（可选）
步骤1已经完成了HDFS的数据的迁移，但是HDFS的元数据和快照都在NameNode上，当然，如果用户不想迁移旧实例上的NameNode也可以，因为NameNode不存储数据，只是文件系统的管理者。如果要迁移NameNode，还需要进行后面的步骤。   
NameNode作为HDFS文件系统的命名空间的管理者，其将所有的文件和文件目录的元数据保存在一个文件系统树中。为了保证交互速度，这些元数据信息会保存在内存中，但同时也会定期将这些信息保存到硬盘上进行持久化存储，这些信息保存的目录即为：$dfs.namenode.name.dir$/current/  
关于这部分的详细介绍请看https://github.com/liumihust/ecs.hadoop/blob/master/Hadoop%20NameNode%20%E5%85%83%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8%E5%88%86%E6%9E%90.md   

下图为该目录下的结构（我在ECS上的实验机子为例）：   
![current](https://github.com/liumihust/gitTset/blob/master/current.PNG)

主要的文件：   
命名空间镜像文件（fsimage）   
修改日志文件（edits）   
它们是恢复nameNode时重要的文件。   
所以我们如果迁移nameNode，就需要先将当前nameNode该目录下的文件全部拷贝到新的nameNode的对应的目录下，即$dfs.namenode.name.dir/current/。 然后在新的实例上启动nameNode进程即可。
