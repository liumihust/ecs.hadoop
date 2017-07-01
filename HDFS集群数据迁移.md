HDFS集群数据迁移
=

场景：用户已经租用了一定数量的阿里云ECS实例，并搭建了Hadoop或者Spark集群。以前ECS实例的硬盘都是采用云盘，相比如本地硬盘，云盘容错好但是带宽低。而Hadoop和Spark等大数据平台往往对硬盘IO要求很高，另一方面它们底层的存储都是采用HDFS，HDFS本身具有容错机制，对于这类的大数据平台采用本地硬盘会是一个更佳的选择。于是，阿里云ECS也新推出了本地硬盘实例。   
但是目前的本地硬盘不能直接添加到用户已有的实例，而需要购买新的基于本地硬盘的实例。对于已经搭建了Hadoop或者Spark平台的用户来说，想要更新到最新的基于本地硬盘的实例，硬盘数据的迁移是一个关键的过程。   
于是，针对HDFS的硬盘数据迁移，我们做了深入的探究。下面是目前已经证实可行的迁移方案。

方案：   
1、DataNode迁移
-
将新增的基于本地硬盘的实例（后面统称新实例）添加到现有的集群，新旧混合，然后创建一个文件oldInstances，记录所有旧dataNode的ip地址。放到定义目录，比如$HADOOP_HOME$/etc/hadoop/。然后修改配置文件hdfs-site.xml，添加如下内容：
```
           <property>
               <name>dfs.hosts.exclude</name>
               <value>$HADOOP_HOME$/etc/hadoop/oldInstances</value>
               <final>true</final>
           </property>
      
```

将修改后的配置文件及oldInstances分发到集群所有实例对应的目录。

在nameNode上执行hdfs dfsadmin –refreshNodes，HDFS会自动读取oldInstances里面所有的hostname，并将这些节点的dataNode数据迁移到其他新的的dataNode上。全部迁移结束后，这些旧的实例将处于Decommissioned状态，不参与HDFS的储存，可以直接杀死。这样就实现了数据从旧实例到新实例的迁移。值得说明的是，迁移后的旧实例的dataNode的数据并不会改变，即在迁移过程中出现任何差错，都不会导致原数据丢失。   

考虑到迁移后的数据分布不一定均衡，可以用HDFS自带的均衡器进行re-balance：   
由于HDFS考虑到re-balance的IO可能会影响应用，所以默认的IO带宽限制得很低，我们也可以提升，   
hdfs dfsadmin –setBalancerBandwidth 104857600 //比如100M   
开始re-balance，   
$HADOOP_HOME$/sbin/start-balancer.sh -threshold 5 //5表示dataNode之间的硬盘占用率之差不超过5%。   
re-balance进程会在后台一直运行，直到达到用户要求的平衡阈值。

2、NameNode迁移（可选）
-
步骤1已经完成了HDFS的数据的迁移，但是HDFS的元数据和快照都在nameNode上，当然，如果用户不想迁移HDFS也可以，因为nameNode不存储数据，只是文件系统的管理者。如果要迁移nameNode，需要进行后面的步骤。   
NameNode作为HDFS文件系统的命名空间的管理者，其将所有的文件和文件目录的元数据保存在一个文件系统树中。为了保证交互速度，这些元数据信息会保存在内存中，但同时也会定期将这些信息保存到硬盘上进行持久化存储，这些信息保存的目录即为：$dfs.namenode.name.dir$/current/   
下图为该目录下的结构（我在ECS上的实验机子为例）：   
![current](https://github.com/liumihust/gitTset/blob/master/current.PNG)

主要的文件：   
命名空间镜像文件（fsimage）   
修改日志文件（edits）   
它们是恢复nameNode时重要的文件。   
所以我们如果迁移nameNode，就需要先将当前nameNode该目录下的文件全部拷贝到新的nameNode的对应的目录下。然后在新的nameNode上启动nameNode即可
