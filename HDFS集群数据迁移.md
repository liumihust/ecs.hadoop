# HDFS集群数据迁移

## 场景：
用户已经租用了一定数量的阿里云ECS实例，并搭建了Hadoop或者Spark集群。以前ECS实例的硬盘都是采用云盘，相比如本地硬盘，云盘容错好但是带宽低。而Hadoop和Spark等大数据平台往往对硬盘IO要求很高，另一方面它们底层的存储都是采用HDFS，HDFS本身具有容错机制，因而对于这类的大数据平台采用本地硬盘会是一个更佳的选择。于是，阿里云ECS也新推出了本地硬盘实例。    
但是目前的本地硬盘不能直接添加到用户已有的ECS实例，而需要购买新的基于本地硬盘的实例。对于已经搭建了Hadoop或者Spark平台的用户来说，想要更新到最新的基于本地硬盘的实例，硬盘数据的迁移是一个非常关键的过程。    
于是，针对HDFS的硬盘数据迁移，我们做了深入的探究。下面是目前已经通过实验证实可行的迁移方案。

## 方案： 
### 1 DataNode迁移
#### 1.1 数据迁移
将新增的基于本地硬盘的实例（后面统称新实例）添加到现有的集群，新旧混合，然后创建一个文件oldInstances，记录所有旧DataNode的ip地址。放到自定义目录，比如$HADOOP_HOME/etc/hadoop/。然后修改配置文件hdfs-site.xml，添加如下内容：
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
HDFS会自动读取oldInstances里面所有的hostname，并将这些节点的DataNode数据迁移到其他新的的DataNode上。全部迁移结束后，这些旧的实例将处于Decommissioned状态，不参与HDFS的储存，可以直接杀死。这样就实现了数据从旧实例到新实例的迁移。值得说明的是，迁移后的旧实例的DataNode的数据并不会改变，这样，在迁移过程中出现任何差错，都不会导致原数据丢失。   
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
ECS上最初有两台服务器，node1、node2构成一个集群，其中node1上同时启动NameNode、DataNode，node2为DataNode，数据块的副本数为2。   
我们在HDFS存储约2.77G的数据，考虑到副本，实际的数据量应该是两倍，约5.54G。如下所示，HDFS将数据均匀地分布在两个DataNode。
```
-------------------------------------------------
Live datanodes (2):

Name: 10.30.209.220:50010 (node1)
Hostname: node1
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991864304 (2.79 GB)
Non DFS Used: 5321508368 (4.96 GB)
DFS Remaining: 33827127296 (31.50 GB)
DFS Used%: 7.10%
DFS Remaining%: 80.27%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:33:57 CST 2017


Name: 10.30.209.243:50010 (node2)
Hostname: node2
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991864304 (2.79 GB)
Non DFS Used: 4107343376 (3.83 GB)
DFS Remaining: 35041292288 (32.63 GB)
DFS Used%: 7.10%
DFS Remaining%: 83.15%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:33:57 CST 2017
```
接下来，我们在ECS再申请3个新实例node3、node4、node5作为DataNode添加到集群，一开始三个新的DataNode没有数据，如下所示：
```
-------------------------------------------------
Live datanodes (5):

Name: 10.30.209.220:50010 (node1)
Hostname: node1
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991864304 (2.79 GB)
Non DFS Used: 5321438736 (4.96 GB)
DFS Remaining: 33827196928 (31.50 GB)
DFS Used%: 7.10%
DFS Remaining%: 80.27%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:37:57 CST 2017


Name: 10.30.210.52:50010 (node5)
Hostname: node5
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 24576 (24 KB)
Non DFS Used: 4230496256 (3.94 GB)
DFS Remaining: 37909979136 (35.31 GB)
DFS Used%: 0.00%
DFS Remaining%: 89.96%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:37:55 CST 2017


Name: 10.30.209.242:50010 (node4)
Hostname: node4
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 24576 (24 KB)
Non DFS Used: 4012118016 (3.74 GB)
DFS Remaining: 38128357376 (35.51 GB)
DFS Used%: 0.00%
DFS Remaining%: 90.48%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:37:54 CST 2017


Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 24576 (24 KB)
Non DFS Used: 4070617088 (3.79 GB)
DFS Remaining: 38069858304 (35.46 GB)
DFS Used%: 0.00%
DFS Remaining%: 90.34%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:37:55 CST 2017


Name: 10.30.209.243:50010 (node2)
Hostname: node2
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991864304 (2.79 GB)
Non DFS Used: 4106638864 (3.82 GB)
DFS Remaining: 35041996800 (32.64 GB)
DFS Used%: 7.10%
DFS Remaining%: 83.16%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:37:57 CST 2017
```

接下来按照前面的迁移操作，完成后的结果如下所示：

```
-------------------------------------------------------
Live datanodes (5):

Name: 10.30.209.220:50010 (node1)
Hostname: node1
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991955968 (2.79 GB)
Non DFS Used: 5321428992 (4.96 GB)
DFS Remaining: 33827115008 (31.50 GB)
DFS Used%: 7.10%
DFS Remaining%: 80.27%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:49:12 CST 2017


Name: 10.30.210.52:50010 (node5)
Hostname: node5
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 1818255360 (1.69 GB)
Non DFS Used: 4230504448 (3.94 GB)
DFS Remaining: 36091740160 (33.61 GB)
DFS Used%: 4.31%
DFS Remaining%: 85.65%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:49:13 CST 2017


Name: 10.30.209.242:50010 (node4)
Hostname: node4
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2461280933 (2.29 GB)
Non DFS Used: 3878376795 (3.61 GB)
DFS Remaining: 35800842240 (33.34 GB)
DFS Used%: 5.84%
DFS Remaining%: 84.96%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:49:13 CST 2017


Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 1838105439 (1.71 GB)
Non DFS Used: 4070702241 (3.79 GB)
DFS Remaining: 36231692288 (33.74 GB)
DFS Used%: 4.36%
DFS Remaining%: 85.98%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:49:13 CST 2017


Name: 10.30.209.243:50010 (node2)
Hostname: node2
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991955968 (2.79 GB)
Non DFS Used: 4106571776 (3.82 GB)
DFS Remaining: 35041972224 (32.64 GB)
DFS Used%: 7.10%
DFS Remaining%: 83.16%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:49:12 CST 2017
```
可以看到，node1、node2上的所有数据块都转移到了node3、node4、node5(前面已经提到，迁移完成后，原有的DataNode的数据不会变)，且node1、node2已经处于Decommissioned状态。它们将不参与存储，我们继续验证，向HDFS存入707M的数据，结果：
```
-------------------------------------------------------
Live datanodes (5):

Name: 10.30.209.220:50010 (node1)
Hostname: node1
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991955968 (2.79 GB)
Non DFS Used: 5321498624 (4.96 GB)
DFS Remaining: 33827045376 (31.50 GB)
DFS Used%: 7.10%
DFS Remaining%: 80.27%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:51:12 CST 2017


Name: 10.30.210.52:50010 (node5)
Hostname: node5
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2243958998 (2.09 GB)
Non DFS Used: 4231796522 (3.94 GB)
DFS Remaining: 35664744448 (33.22 GB)
DFS Used%: 5.32%
DFS Remaining%: 84.63%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:51:13 CST 2017


Name: 10.30.209.242:50010 (node4)
Hostname: node4
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 3209240865 (2.99 GB)
Non DFS Used: 3880648415 (3.61 GB)
DFS Remaining: 35050610688 (32.64 GB)
DFS Used%: 7.62%
DFS Remaining%: 83.18%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:51:10 CST 2017


Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2160361733 (2.01 GB)
Non DFS Used: 4071681787 (3.79 GB)
DFS Remaining: 35908456448 (33.44 GB)
DFS Used%: 5.13%
DFS Remaining%: 85.21%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:51:13 CST 2017


Name: 10.30.209.243:50010 (node2)
Hostname: node2
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2991955968 (2.79 GB)
Non DFS Used: 4106575872 (3.82 GB)
DFS Remaining: 35041968128 (32.64 GB)
DFS Used%: 7.10%
DFS Remaining%: 83.16%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 17:51:12 CST 2017
```
node1、node2的数据没有发生任何变化，只有node3、node4、node5的数据在增加，事实上node1、node2的数据已经没有作用，可以直接kill掉了。
到此为止，我们已经实现了旧实例的硬盘数据往新实例硬盘的迁移。如果发现数据分布不均匀，还可以re-balance，
```
start-balancer.sh -threshold 1
```
分布相对于之前更平衡：
```
Name: 10.30.210.52:50010 (node5)
Hostname: node5
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2701537280 (2.52 GB)
Non DFS Used: 4230520832 (3.94 GB)
DFS Remaining: 35208441856 (32.79 GB)
DFS Used%: 6.41%
DFS Remaining%: 83.55%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 18:32:31 CST 2017


Name: 10.30.209.242:50010 (node4)
Hostname: node4
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2617966592 (2.44 GB)
Non DFS Used: 4012175360 (3.74 GB)
DFS Remaining: 35510358016 (33.07 GB)
DFS Used%: 6.21%
DFS Remaining%: 84.27%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 18:32:32 CST 2017


Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2160427008 (2.01 GB)
Non DFS Used: 4070670336 (3.79 GB)
DFS Remaining: 35909402624 (33.44 GB)
DFS Used%: 5.13%
DFS Remaining%: 85.21%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 18:32:31 CST 2017

```

### 2 NameNode迁移（可选）
步骤1已经完成了HDFS的数据的迁移，数据已经全部迁移到了新的实例上。但是NameNode还没有迁移，HDFS的元数据和快照都在NameNode上，当然，如果新实例和旧实例都是在一个数据中心或者局域网里面，用户不迁移旧实例上的NameNode到新实例也可以，因为NameNode不存储数据，只是文件系统的管理者。如果要迁移NameNode，还需要进行后面的步骤。   
NameNode作为HDFS文件系统的命名空间的管理者，其将所有的文件和文件目录的元数据保存在一个文件系统树中。为了保证交互速度，这些元数据信息会保存在内存中，但同时也会定期将这些信息保存到硬盘上进行持久化存储，这些信息保存的目录即为：$dfs.namenode.name.dir$/current/  
关于NameNode的数据的详细介绍请看https://github.com/liumihust/ecs.hadoop/blob/master/Hadoop%20NameNode%20%E5%85%83%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8%E5%88%86%E6%9E%90.md   

下图为该目录下的结构（我在ECS上的实验机子为例）：   
![current](https://github.com/liumihust/gitTset/blob/master/current.PNG)

主要的文件：   
命名空间镜像文件（fsimage）   
修改日志文件（edits）   
它们是恢复NameNode时重要的文件。   
所以我们如果迁移NameNode，就需要先将当前NameNode该目录下的文件全部拷贝到新的NameNode的对应的目录下，即$dfs.namenode.name.dir/current/。 然后在新的实例上启动NameNode进程即可。
##### ECS实验
前面的实验已经完成了HDFS数据的迁移，从node1、node2全部迁移到node3,、node4、node5。接下来我们接着将NameNode从node1迁移到node3（新实例）。     
修改配置文件core-site.xml：
``` 
               <name>fs.defaultFS</name>
               <value>hdfs://node3:9000/</value>
```
将修改后的配置文件发布到所有节点。     
将node1的$dfs.namenode.name.dir/current/ 下面的数据全部拷贝到node3对应的目录下，   
```
scp -r $dfs.namenode.name.dir/current/ node3:$dfs.namenode.name.dir/
```
在node3执行：   
```
hdfs namenode -format
```
中间会跳出选项
```
Re-format filesystem in Storage Directory /tmp/hadoop/tmp/dfs/name ? (Y or N) 
```
应选择 NO   

接着执行，   
```
$HADOOP_HOME/sbin/start-dfs.sh
```
如果启动没问题，我们将可以看到，node3、node4、node5均启动了DataNode，node3启动了NameNode，任意节点执行：
```
hdfs dfsadmin -report
```
可以看到如下结果，跟NameNode迁移前一致:
```
-------------------------------------------------
Live datanodes (3):

Name: 10.30.210.52:50010 (node5)
Hostname: node5
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2701537280 (2.52 GB)
Non DFS Used: 4231036928 (3.94 GB)
DFS Remaining: 35207925760 (32.79 GB)
DFS Used%: 6.41%
DFS Remaining%: 83.55%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 19:02:07 CST 2017


Name: 10.30.209.242:50010 (node4)
Hostname: node4
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2617962496 (2.44 GB)
Non DFS Used: 4012679168 (3.74 GB)
DFS Remaining: 35509858304 (33.07 GB)
DFS Used%: 6.21%
DFS Remaining%: 84.27%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 19:02:07 CST 2017


Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2160427008 (2.01 GB)
Non DFS Used: 4080926720 (3.80 GB)
DFS Remaining: 35899146240 (33.43 GB)
DFS Used%: 5.13%
DFS Remaining%: 85.19%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jul 01 19:02:07 CST 2017
```
再查看HDFS里面的数据：
```
hdfs dfs -ls /liumihust/
```
结果：
```
[root@node5 ~]# hdfs dfs -ls /liumihust/
17/07/01 19:20:08 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Found 20 items
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:31 /liumihust/data1
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:32 /liumihust/data10
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:33 /liumihust/data11
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:33 /liumihust/data12
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:33 /liumihust/data13
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:33 /liumihust/data14
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:33 /liumihust/data15
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:33 /liumihust/data16
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:50 /liumihust/data17
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:50 /liumihust/data18
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:50 /liumihust/data19
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:31 /liumihust/data2
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:51 /liumihust/data20
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:31 /liumihust/data3
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:32 /liumihust/data4
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:32 /liumihust/data5
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:32 /liumihust/data6
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:32 /liumihust/data7
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:32 /liumihust/data8
-rw-r--r--   2 root supergroup  185540433 2017-07-01 17:32 /liumihust/data9

```
可以看到数据完好无损，至此，就实现了将HDFS从一个集群原封不动地迁移到了一个新的集群。

the end



