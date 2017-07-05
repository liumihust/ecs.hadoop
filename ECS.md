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
![benchmark1](https://github.com/liumihust/gitTset/blob/master/benchmark1.PNG)
## MR Benchmark
![benchmark](https://github.com/liumihust/gitTset/blob/master/benchmark2.PNG)
## NameNode Benchmark
![benchmark](https://github.com/liumihust/gitTset/blob/master/benchmark3.PNG)
## Applications
![benchmark](https://github.com/liumihust/gitTset/blob/master/benchmark4.PNG)
