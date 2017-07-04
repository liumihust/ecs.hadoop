# HDFS挂载本地或云盘
通过 fdisk -l 命令可以查看当前实例的所有硬盘
```
[root@node2 dev]# fdisk -l

Disk /dev/vda: 42.9 GB, 42949672960 bytes
255 heads, 63 sectors/track, 5221 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000215a8

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *           1        5222    41942016   83  Linux

Disk /dev/vdb: 107.4 GB, 107374182400 bytes
16 heads, 63 sectors/track, 208050 cylinders
Units = cylinders of 1008 * 512 = 516096 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000
```
图中vdb即为我们的数据盘，若果有多个数据盘，命名规则为vdb,vdc,vdd...，现在就是要把它挂载到文件系统并添加到HDFS的数据目录
## 创建分区
fdisk /dev/vdb
```
[root@node2 dev]# fdisk /dev/vdb
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0x3c9c7376.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').

Command (m for help): n //第一次
Command action
   e   extended
   p   primary partition (1-4)
p //第二次
Partition number (1-4): 1 //第三次
First cylinder (1-208050, default 1): // 第四次
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-208050, default 208050): //第五次
Using default value 208050

Command (m for help): w  //第六次
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```
分区过程中会有很多的选项：     
第一次输入n，表示要新建分区     
第二次输入p，表示原始分区    
第三次输入1，这里只创建一个分区（可选）     
第四次，第五次均输入回车，表示整个盘创建该分区    
最后输入w，把我们的选项写入硬盘保存    

这时候查看设备可以看到/dev目录下vdb多了一个分区/dev/vdb1
## 格式化硬盘
根据实例的操作系统的文件系统格式，格式化硬盘，以我的CentOS 6.8为例，格式为EXT4    
mkfs -t ext4 -c /dev/vdb
## 挂载到HDFS数据目录
我的实例的DataNode数据目录为/tmp/hadoop/tmp/dfs/data，NameNode目录为/tmp/hadoop/tmp/dfs/name，那么，将硬盘挂载到/tmp/hadoop/目录：    
mount /dev/vdb /tmp/hadoop/      
这样，云盘（本地盘）就成了HDFS的数据盘。
```
[root@node2 ~]# df
Filesystem     1K-blocks    Used Available Use% Mounted on
/dev/vda1       41152832 1918336  37137396   5% /
tmpfs            4030552       0   4030552   0% /dev/shm
/dev/vdb       103081248   61044  97777324   1% /tmp/hadoop

```


