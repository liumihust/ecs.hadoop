# HDFS集群数据迁移

## 场景：
用户已经租用了一定数量的阿里云ECS实例，并搭建了Hadoop或者Spark集群。以前ECS实例的硬盘都是采用云盘，相比如本地硬盘，云盘容错好但是带宽低。而Hadoop和Spark等大数据平台往往对硬盘IO要求很高，另一方面它们底层的存储都是采用HDFS，HDFS本身具有容错机制，因而对于这类的大数据平台采用本地硬盘会是一个更佳的选择。于是，阿里云ECS也新推出了本地硬盘实例。     
关于基于ECS云盘的HDFS和基于ECS本地硬盘的HDFS性能的详细评测实验在: https://github.com/liumihust/ecs.hadoop/blob/master/ECS_BenchMark.md      
但是目前的本地硬盘不能直接添加到用户已有的ECS实例，而需要购买新的基于本地硬盘的实例。对于已经搭建了Hadoop或者Spark平台的用户来说，想要更新到最新的基于本地硬盘的实例，硬盘数据的迁移是一个非常关键的过程。     
于是，针对HDFS的硬盘数据迁移，我们做了深入的探究。下面是目前已经通过实验证实可行的迁移方案。

## 方案： 
### 1 DataNode迁移
#### 1.1 数据迁移
##### 1.1.1 将新增的基于本地硬盘的实例（后面统称新实例）添加到现有的集群，新旧混合。
将新加入的实例的hostname追加到slaves文件，分发到新旧实例，然后在每个新实例上执行，
```
$HADOOP_HOME/sbin/hadoop-daemon.sh start datanode
```
不需要重启，新的DataNode即可加入集群。   
##### 1.1.2 删除旧实例
实例混合后，创建一个文件oldInstances，记录所有旧DataNode的ip地址。放到自定义目录，比如$HADOOP_HOME/etc/hadoop/。然后修改配置文件hdfs-site.xml，添加如下内容：
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

### 2 NameNode迁移（可选）
步骤1已经完成了HDFS的数据的迁移，数据已经全部迁移到了新的实例上。但是NameNode还没有迁移，HDFS的元数据和快照都在NameNode上，当然，如果新实例和旧实例都是在一个数据中心或者局域网里面，用户不迁移旧实例上的NameNode到新实例也可以，因为NameNode不存储数据，只是文件系统的管理者。如果要迁移NameNode，还需要进行后面的步骤。    
NameNode作为HDFS文件系统的命名空间的管理者，其将所有的文件和目录的元数据保存在一个文件系统树中。为了使与客户端的交互更高效，这些元数据信息都会保存在内存中，但同时也会定期将这些信息保存到硬盘上进行持久化存储，这些信息保存的目录即为：$dfs.namenode.name.dir$/current/  
关于NameNode的数据的详细介绍请看https://github.com/liumihust/ecs.hadoop/blob/master/Hadoop%20NameNode%20%E5%85%83%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8%E5%88%86%E6%9E%90.md   

下图为该目录下的结构（我在ECS上的实验机子为例）：   
![current](https://github.com/liumihust/gitTset/blob/master/current.PNG)

主要的文件：   
命名空间镜像文件（fsimage）   
修改日志文件（edits）   
它们是恢复NameNode时重要的文件。   
所以我们如果迁移NameNode，就需要先将当前NameNode该目录下的文件全部拷贝到新的NameNode的对应的目录下，即$dfs.namenode.name.dir/current/。 然后在新的实例上启动NameNode进程即可。这样新启动的HDFS集群就是NameNode关闭之前的状态。

### 3 阿里云ECS实例实验
旧实例配置：     
ECS规格：ecs.n1.large    
CPU：4核     
内存：8G     
系统盘：高效云盘40G     
数据盘：普通云盘100G     
OS：CentOS 6.8 64位     
Hadoop版本：2.6.4    

新实例配置：     
ECS规格：ecs.i1.xlarge      
CPU：4核      
内存：16G    
系统盘：高效云盘40G     
数据盘：本地硬盘100G     
OS：CentOS 6.8 64位      
Hadoop版本：2.6.4 

#### 3.1 DataNode迁移
ECS上最初有三台服务器，node3、node4、node5构成一个集群，由于系统盘和数据盘都是云盘，所以本实验直接将系统盘作为旧实例HDFS的数据盘，故三个DataNode总的Configured Capacity为120G。node3上同时启动NameNode、DataNode，node4、node5为DataNode，数据块的副本数为2。我们在HDFS存储约3.54G的数据。如下所示，HDFS将数据分布在三个DataNode。
```
-------------------------------------------------
Configured Capacity: 126421499904 (117.74 GB)
Present Capacity: 113898897624 (106.08 GB)
DFS Remaining: 106419224576 (99.11 GB)
DFS Used: 7479673048 (6.97 GB)
DFS Used%: 6.57%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Live datanodes (3):

Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 3739824236 (3.48 GB)
Non DFS Used: 4267552660 (3.97 GB)
DFS Remaining: 34133123072 (31.79 GB)
DFS Used%: 8.87%
DFS Remaining%: 81.00%
Last contact: Sat Jul 01 21:14:27 CST 2017


Name: 10.30.210.52:50010 (node5)
Hostname: node5
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 1682934423 (1.57 GB)
Non DFS Used: 4239074665 (3.95 GB)
DFS Remaining: 36218490880 (33.73 GB)
DFS Used%: 3.99%
DFS Remaining%: 85.95%
Last contact: Sat Jul 01 21:14:26 CST 2017


Name: 10.30.209.242:50010 (node4)
Hostname: node4
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2056914389 (1.92 GB)
Non DFS Used: 4015974955 (3.74 GB)
DFS Remaining: 36067610624 (33.59 GB)
DFS Used%: 4.88%
DFS Remaining%: 85.59%
Last contact: Sat Jul 01 21:14:27 CST 2017
```
接下来，我们在ECS再申请4个新实例node6、node7、node8、node9作为DataNode，并挂载本地硬盘到HDFS的数据目录，具体操作见附录，然后添加到集群，一开始4个新的DataNode没有数据，如下所示：
```
Configured Capacity: 565552676864 (526.71 GB)
Present Capacity: 530446290944 (494.02 GB)
DFS Remaining: 522966278144 (487.05 GB)
DFS Used: 7480012800 (6.97 GB)
DFS Used%: 1.41%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Live datanodes (7):

Name: 10.30.120.12:50010 (node7)
Hostname: node7
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 24576 (24 KB)
Non DFS Used: 5645987840 (5.26 GB)
DFS Remaining: 104136781824 (96.98 GB)
DFS Used%: 0.00%
DFS Remaining%: 94.86%
Last contact: Sat Jul 01 21:27:33 CST 2017


Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 3739938816 (3.48 GB)
Non DFS Used: 4267499520 (3.97 GB)
DFS Remaining: 34133061632 (31.79 GB)
DFS Used%: 8.87%
DFS Remaining%: 81.00%
Last contact: Sat Jul 01 21:27:33 CST 2017


Name: 10.31.0.18:50010 (node9)
Hostname: node9
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 24576 (24 KB)
Non DFS Used: 5645987840 (5.26 GB)
DFS Remaining: 104136781824 (96.98 GB)
DFS Used%: 0.00%
DFS Remaining%: 94.86%
Last contact: Sat Jul 01 21:27:31 CST 2017


Name: 10.30.210.52:50010 (node5)
Hostname: node5
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 1682993152 (1.57 GB)
Non DFS Used: 4239020032 (3.95 GB)
DFS Remaining: 36218486784 (33.73 GB)
DFS Used%: 3.99%
DFS Remaining%: 85.95%
Last contact: Sat Jul 01 21:27:32 CST 2017


Name: 10.30.120.9:50010 (node6)
Hostname: node6
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 24576 (24 KB)
Non DFS Used: 5645987840 (5.26 GB)
DFS Remaining: 104136781824 (96.98 GB)
DFS Used%: 0.00%
DFS Remaining%: 94.86%
Last contact: Sat Jul 01 21:27:31 CST 2017


Name: 10.30.120.22:50010 (node8)
Hostname: node8
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 24576 (24 KB)
Non DFS Used: 5645987840 (5.26 GB)
DFS Remaining: 104136781824 (96.98 GB)
DFS Used%: 0.00%
DFS Remaining%: 94.86%
Last contact: Sat Jul 01 21:27:32 CST 2017


Name: 10.30.209.242:50010 (node4)
Hostname: node4
Decommission Status : Normal
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2056982528 (1.92 GB)
Non DFS Used: 4015915008 (3.74 GB)
DFS Remaining: 36067602432 (33.59 GB)
DFS Used%: 4.88%
DFS Remaining%: 85.59%
Last contact: Sat Jul 01 21:27:33 CST 2017

```

按照1.1的迁移操作，完成后的结果如下所示：
```
Configured Capacity: 446611091456 (415.94 GB)
Present Capacity: 424019357912 (394.90 GB)
DFS Remaining: 409059745792 (380.97 GB)
DFS Used: 14959612120 (13.93 GB)
DFS Used%: 3.53%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Live datanodes (7):

Name: 10.30.120.12:50010 (node7)
Hostname: node7
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 1901743373 (1.77 GB)
Non DFS Used: 5646140147 (5.26 GB)
DFS Remaining: 102234910720 (95.21 GB)
DFS Used%: 1.73%
DFS Remaining%: 93.12%
Last contact: Sat Jul 01 21:33:15 CST 2017


Name: 10.29.254.31:50010 (node3)
Hostname: node3
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 3739938816 (3.48 GB)
Non DFS Used: 4267552768 (3.97 GB)
DFS Remaining: 34133008384 (31.79 GB)
DFS Used%: 8.87%
DFS Remaining%: 81.00%
Last contact: Sat Jul 01 21:33:15 CST 2017


Name: 10.31.0.18:50010 (node9)
Hostname: node9
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 1619296489 (1.51 GB)
Non DFS Used: 5649997591 (5.26 GB)
DFS Remaining: 102513500160 (95.47 GB)
DFS Used%: 1.48%
DFS Remaining%: 93.38%
Last contact: Sat Jul 01 21:33:13 CST 2017


Name: 10.30.210.52:50010 (node5)
Hostname: node5
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 1682993152 (1.57 GB)
Non DFS Used: 4239044608 (3.95 GB)
DFS Remaining: 36218462208 (33.73 GB)
DFS Used%: 3.99%
DFS Remaining%: 85.95%
Last contact: Sat Jul 01 21:33:14 CST 2017


Name: 10.30.120.9:50010 (node6)
Hostname: node6
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 1651115456 (1.54 GB)
Non DFS Used: 5648329280 (5.26 GB)
DFS Remaining: 102483349504 (95.45 GB)
DFS Used%: 1.50%
DFS Remaining%: 93.35%
Last contact: Sat Jul 01 21:33:13 CST 2017


Name: 10.30.120.22:50010 (node8)
Hostname: node8
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 2307542306 (2.15 GB)
Non DFS Used: 5647266526 (5.26 GB)
DFS Remaining: 101827985408 (94.83 GB)
DFS Used%: 2.10%
DFS Remaining%: 92.75%
Last contact: Sat Jul 01 21:33:14 CST 2017


Name: 10.30.209.242:50010 (node4)
Hostname: node4
Decommission Status : Decommissioned
Configured Capacity: 42140499968 (39.25 GB)
DFS Used: 2056982528 (1.92 GB)
Non DFS Used: 4015939584 (3.74 GB)
DFS Remaining: 36067577856 (33.59 GB)
DFS Used%: 4.88%
DFS Remaining%: 85.59%
Last contact: Sat Jul 01 21:33:15 CST 2017
```

可以看到，node3、node4、node5上的所有数据块都转移到了node6、node7、node8、node9(前面已经提到，迁移完成后，原有的DataNode的数据不会变)，且node3、node4、node5都已经处于Decommissioned状态。它们将不参与存储，事实上node3、node4、node5的数据已经没有作用，可以直接kill掉了。     
到此为止，我们已经实现了旧实例的硬盘数据往新实例硬盘的迁移。如果发现数据分布不均匀，还可以re-balance，
```
start-balancer.sh -threshold 1
```


#### NameNode迁移
前面的实验已经完成了HDFS数据的迁移，从node3、node4、node5全部迁移到node6、node7、node8、node9。接下来我们接着将NameNode从node3迁移到node6（新实例）      
修改配置文件core-site.xml：
``` 
               <name>fs.defaultFS</name>
               <value>hdfs://node6:9000/</value>
```
将修改后的配置文件发布到所有节点。     
将node3的$dfs.namenode.name.dir/ 下面的数据全部拷贝到node6对应的目录下，   
```
scp -r $dfs.namenode.name.dir/ node3:$dfs.namenode.name.dir/
```
在node6执行：   
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
如果启动没问题，我们将可以看到，node6、node7、node8、node9均启动了DataNode，node6启动了NameNode，在任意节点执行：
```
hdfs dfsadmin -report
```
可以看到如下结果，跟NameNode迁移前一致:
```
Configured Capacity: 439131176960 (408.97 GB)
Present Capacity: 416545062912 (387.94 GB)
DFS Remaining: 409065115648 (380.97 GB)
DFS Used: 7479947264 (6.97 GB)
DFS Used%: 1.80%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Live datanodes (4):

Name: 10.30.120.12:50010 (node7)
Hostname: node7
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 1901809664 (1.77 GB)
Non DFS Used: 5645991936 (5.26 GB)
DFS Remaining: 102234992640 (95.21 GB)
DFS Used%: 1.73%
DFS Remaining%: 93.12%
Last contact: Sat Jul 01 21:44:25 CST 2017


Name: 10.31.0.18:50010 (node9)
Hostname: node9
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 1619349504 (1.51 GB)
Non DFS Used: 5645991936 (5.26 GB)
DFS Remaining: 102517452800 (95.48 GB)
DFS Used%: 1.48%
DFS Remaining%: 93.38%
Last contact: Sat Jul 01 21:44:25 CST 2017


Name: 10.30.120.9:50010 (node6)
Hostname: node6
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 1651167232 (1.54 GB)
Non DFS Used: 5648138240 (5.26 GB)
DFS Remaining: 102483488768 (95.45 GB)
DFS Used%: 1.50%
DFS Remaining%: 93.35%
Last contact: Sat Jul 01 21:44:25 CST 2017


Name: 10.30.120.22:50010 (node8)
Hostname: node8
Decommission Status : Normal
Configured Capacity: 109782794240 (102.24 GB)
DFS Used: 2307620864 (2.15 GB)
Non DFS Used: 5645991936 (5.26 GB)
DFS Remaining: 101829181440 (94.84 GB)
DFS Used%: 2.10%
DFS Remaining%: 92.76%
Last contact: Sat Jul 01 21:44:25 CST 2017
```
唯一的区别就是Configured Capacity减少了约7G，因为旧的DataNode上的的数据已经被删除。

再查看HDFS里面的数据：
```
hdfs dfs -ls /liumihust/
```
结果：
```
[root@node6 ~]# hdfs dfs -ls /liumihust/
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
可以看到20块数据完好无损，至此，就实现了将HDFS从一个基于云盘的集群原封不动地迁移到了一个新的基于本地硬盘的集群。

the end

## 附录
HDFS挂载ECS实例本地盘或云盘 https://github.com/liumihust/ecs.hadoop/blob/master/HDFS%E6%8C%82%E8%BD%BDECS%E5%AE%9E%E4%BE%8B%E6%9C%AC%E5%9C%B0%E7%9B%98%E6%88%96%E4%BA%91%E7%9B%98.md



