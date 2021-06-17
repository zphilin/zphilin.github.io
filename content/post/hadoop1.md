---
title: "使用docker搭建hadoop集群"
date: 2016-11-19T01:37:56+08:00
#lastmod: 2021-06-17T01:37:56+08:00
draft: false
tags: ["hadoop"]
categories: ["大数据"]
author: "ifreer"

---
# 制作基于ubuntu的Hadoop镜像

> 文中使用的到的docker知识及命令可参考 Docker基础
> 
> 在开始之前需确保已安装好docker环境

# 安装JAVA

```
#由于hadoop运行需要java环境，在宿主机新建一个容器用于配置java及安装hadoop
docker run -it ubuntu
apt-get install software-properties-common python-software-properties 
add-apt-repository ppa:webupd8team/java
apt-get update 
apt-get install oracle-java7-installer 
java -version
```

# 安装Hadoop

```
cd ~
mkdir soft
cd soft
mkdir hadoop
wget http://mirrors.sonic.net/apache/hadoop/common/hadoop-2.6.5/hadoop-2.6.5.tar.gz
tar -zxvf hadoop-2.6.5.tar.gz

vim ~/.bashrc
export JAVA_HOME=/usr/lib/jvm/java-7-oracle 
export HADOOP_HOME=/root/soft/hadoop/hadoop-2.6.5 
export HADOOP_CONFIG_HOME=$HADOOP_HOME/etc/hadoop 
export PATH=$PATH:$HADOOP_HOME/bin 
export PATH=$PATH:$HADOOP_HOME/sbin

vim hadoop-env.sh
export JAVA_HOME=/usr/lib/jvm/java-7-oracle
```

# 配置Hadoop

分别配置 **core-site.xml**、**hdfs-site.xml**、**mapred-site.xml三个文件**

```
cd $HADOOP_CONFIG_HOME
mkdir tmp
mkdir namenode
mkdir datanode
#创建各节点的存放目录用于配置
```

**vim core-site.xml 配置configuration中的property 如下**

> &lt;configuration&gt;
> 
> &lt;property&gt;
> 
> &lt;name&gt;hadoop.tmp.dir&lt;\/name&gt;
> 
> &lt;value&gt;\/root\/soft\/hadoop\/hadoop-2.6.5\/tmp&lt;\/value&gt;
> 
> &lt;description&gt;A base for other temporary directories.&lt;\/description&gt;
> 
> &lt;\/property&gt;
> 
> &lt;property&gt;
> 
> &lt;name&gt;fs.default.name&lt;\/name&gt;
> 
> &lt;value&gt;hdfs:\/\/master:9000&lt;\/value&gt;
> 
> &lt;final&gt;true&lt;\/final&gt;
> 
> &lt;description&gt;The name of the default file system. A URI whose scheme and authority determine the FileSystem implementation. The  uri's scheme determines the config property \(fs.SCHEME.impl\) naming the FileSystem implementation class. The uri's authority is used to determine the host, port, etc. for a filesystem.
> 
> &lt;\/description&gt;
> 
> &lt;\/property&gt;
> 
> &lt;\/configuration&gt;

**vim hdfs.site.xml**

> &lt;configuration&gt;
> 
> &lt;property&gt;
> 
> &lt;name&gt;dfs.replication&lt;\/name&gt;
> 
> &lt;value&gt;2&lt;\/value&gt;
> 
> &lt;final&gt;true&lt;\/final&gt;
> 
> &lt;description&gt;Default block replication.
> 
> The actual number of replications can be specified when the file is created.
> 
> The default is used if replication is not specified in create time.
> 
> &lt;\/description&gt;
> 
> &lt;\/property&gt;
> 
> &lt;property&gt;
> 
> &lt;name&gt;dfs.namenode.name.dir&lt;\/name&gt;
> 
> &lt;value&gt;\/root\/soft\/hadoop\/hadoop-2.6.5\/namenode&lt;\/value&gt;
> 
> &lt;final&gt;true&lt;\/final&gt;
> 
> &lt;\/property&gt;
> 
> &lt;property&gt;
> 
> &lt;name&gt;dfs.datanode.data.dir&lt;\/name&gt;
> 
> &lt;value&gt;\/root\/soft\/hadoop\/hadoop-2.6.5\/datanode&lt;\/value&gt;
> 
> &lt;final&gt;true&lt;\/final&gt;
> 
> &lt;\/property&gt;
> 
> &lt;\/configuration&gt;

**cp mapred-site.xml.template mapred-site.xml**

**vim mapred-site.xml**

> &lt;configuration&gt;
> 
> &lt;property&gt;
> 
> &lt;name&gt;mapred.job.tracker&lt;\/name&gt;
> 
> &lt;value&gt;master:9001&lt;\/value&gt;
> 
> &lt;description&gt;The host and port that the MapReduce job tracker runs
> 
> at. If "local", then jobs are run in-process as a single map
> 
> and reduce task.
> 
> &lt;\/description&gt;
> 
> &lt;\/property&gt;
> 
> &lt;\/configuration&gt;

# 配置节点认证互信

```
#设置服务自启动
vim ~/.bashrc
/usr/sbin/sshd

ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa 
cd ~/.ssh
cat id_dsa.pub >> authorized_keys
#实验中，各节点可使用同一个公钥；也可后续生成不同的公钥，将各节点自己的公钥加入到其他节点的authorized_keys中
#注意authorized_keys文件非owner用户不能有w权限

exit 
docker commit -m 'hadoop installed' container_id ubuntu:hadoop
#提交保持hadoop镜像用于后续新建实例
```

# Hadoop集群实施

基于之前的hadoop的镜像，此时只需new（docker run）多个容器即可创建多个hadoop节点，然后修改其中的配置，配置为Master或Slave

# 新建hadoop节点

```
#在三个不同的终端标签页中分别执行以下命令
docker run -it -h master -p 50070:50070 ubuntu:hadoop
docker run -it -h slave1 ubuntu:hadoop
docker run -it -h slave2 ubuntu:hadoop
```

# 配置Hosts

```
#在各节点容器中执行ifconfig命令，得到具体ip，然后向各节点hosts都写入
vim /etc/hosts
172.17.0.4 master
172.17.0.5 slave1
172.17.0.6 slave2
```

# Master节点设置

```
cd $HADOOP_CONFIG_HOME
vim slaves #写入所有slave节点的主机名hostname
slave1
slave2
```

# 启动服务

```bash
#在master节点执行start-all.sh 若能看到以下类似信息表示成功
slave1: starting nodemanager, logging to /root/soft/hadoop/hadoop-2.6.5/logs/yarn-root-nodemanager-slave1.out
```

# 查看服务状态

> \#在各节点中执行jps命令查看服务进程
> 
> jps \#master中信息
> 
> 1762 NodeManager
> 
> 3592 Jps
> 
> 974 NameNode
> 
> 1563 SecondaryNameNode
> 
> 336 ResourceManager
> 
> 1418 DataNode

对于非Toolbox的Docker环境可直接在浏览器中访问master节点 [http:\/\/172.17.0.4:50070](http://172.17.0.4:50070)

如果是采用Toolbox安装的Docker，则访问[http:\/\/192.168.99.100:50070 ](http://192.168.99.100:50070)

此中情况需要做端口映射，见开始docker run处，如果没有可在宿主机执行，达到动态修改端口～

iptables -t nat -A PREROUTING -p tcp -m tcp --dport 50070 -j DNAT --to-destination 172.17.0.4:50070

![](/assets/屏幕快照 2016-11-19 下午10.21.04.png)


*ifreer@2016.11.19*


