### 1. 查看服务器的CPU

```
查看每个物理cpu中的core个数：
cat /proc/cpuinfo |grep "cpu cores"|wc -l
```

### 2. 打印所有进程及其线程

```
##打印所有进程及其线程
pstree -p 
##打印某个进程的线程数
pstree -p {pid} | wc -l

##ps 后面加上H，能打印某个进程的所有线程
ps hH p {pid} | wc -l

##top命令后面跟-H，会打印出所有线程列表
top -H
top -H -p {pid}
##举例如下
top -H -p 27587
```

