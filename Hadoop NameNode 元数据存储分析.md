## Hadoop NameNode 元数据存储分析

NameNode作为HDFS文件系统的命名空间的管理者，其将所有的文件和文件目录的元数据保存在一个文件系统树中。为了保证交互速度，这些元数据信息会保存在内存中，但同时也会定期将这些信息保存到硬盘上进行持久化存储，进行备份。这些信息保存的目录即为：HDFS-site.xml配置文件$dfs.namenode.name.dir$的current/子目录下。
以我在阿里云ECS部署的服务器为例，该目录下的文件如下图所示：
![current](https://github.com/liumihust/gitTset/blob/master/current.PNG)
主要的文件：   

命名空间镜像文件（fsimage）   

修改日志文件（edits）   

它们是恢复nameNode时重要的文件。   

NameNode 在执行 HDFS 客户端提交的写操作的时候，会首先把这些操作记录在 EditLog 文件之中，然后再更新内存中的文件系统镜像。内存中的文件系统镜像用于 NameNode 向客户端提供高效的读服务，而 EditLog 仅仅只是在数据恢复的时候起作用。记录在 EditLog 之中的每一个操作又称为一个事务，每个事务有一个整数形式的事务 id 作为编号。EditLog 会被切割为很多段，每一段称为一个 Segment，如上图所示。内存中的文件系统镜像也会定期保存在硬盘进行持久化，相当于一次checkpoint。   

日志文件文件的命名规则为edits_{该segment起始事务id}-{segment结束事务id}，如果还有日志正在写入，那么对应的segment的命名为edits_inprogress_{该segment起始事务id}-{segment结束事务id}。   
命名空间镜像文件的命名规则为fsimage_{备份的最后一个事务id}，该文件是序列化文件，不能直接被修改。   

有了这两个文件后，HDFS在重启时就可以根据这两个文件来进行状态恢复：首先将最新的checkpoint的元数据信息从fsimage中加载到内存，然后逐一执行edits修改日志文件中的操作记录以恢复到重启之前的最终状态。
备注：Hadoop的持久化过程是将上一次checkpoint以后最近一段时间的操作保存到修改日志文件edits中   

### SecondaryNameNode
仅仅依靠nameNode已经能够完成数据的备份，但是当应用长时间运行，基于这样的备份进行恢复的开销很大，因为nameNode需要按照修改日志文件里面的操作一步步进行恢复。于是HDFS增加了一个额外的SecondaryNameNode，专门在后台读取nameNode的fsimage并载入内存中根据edits的操作记录生成新的fsimage提交给nameNode。nameNode收到新的fsimage后就会删除旧的fsimage，edits重新开始。这样就能保证fsimage是最新的，edits都是最近的操作，因为edits不会很大，既节省空间又降低了恢复的开销。
