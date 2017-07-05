# ECS实例HDFS性能对比测试
## 实验环境
云盘实例配置：     
ECS规格：ecs.n1.large    
CPU：4核     
内存：16G     
系统盘：高效云盘40G     
数据盘：普通云盘100G     
OS：CentOS 6.8 64位     
Hadoop版本：2.6.4    
集群规模：4台     

本地硬盘实例配置：    
ECS规格：ecs.i1.xlarge      
CPU：4核      
内存：16G    
系统盘：高效云盘40G     
数据盘：本地硬盘100G     
OS：CentOS 6.8 64位     
Hadoop版本：2.6.4    
集群规模：4台
## DFS IO Benchmark
测试HDFS的持续读写能力和随机读写能力。
![benchmark1](https://github.com/liumihust/gitTset/blob/master/benchmark1.PNG)
## MR Benchmark
测试MapReduce小作业的执行效率，比较每个作业的平均执行时间。
![benchmark](https://github.com/liumihust/gitTset/blob/master/benchmark2.PNG)
## NameNode Benchmark
测试NameNode的抗压能力，该测试频繁向HDFS发出请求，创建文件，读写文件，关闭文件等。
![benchmark](https://github.com/liumihust/gitTset/blob/master/benchmark3.PNG)
## Applications
常见的一些应用。
![benchmark](https://github.com/liumihust/gitTset/blob/master/benchmark4.PNG)
