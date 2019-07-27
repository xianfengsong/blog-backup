title: Mesos安装
date: 2017-12-26 20:05:46
categories: 容器技术
tags: [mesos]

---

系统环境：CentOS 6.6 java7

mesos安装

按官网教程下载&编译
http://mesos.apache.org/documentation/latest/building/

借助mussh同时操作多个机器（需要先建立互信）

示例：`./mussh -H iplist -c 'hostname -i'`;
-H 指定ip
-c 指定执行的命令
iplist 一行一个ip
cat iplist 
10.9.19.xx
10.9.19.xx
10.9.19.xx
<!--more-->

问题
1.执行configure时
configure: error: cannot find libcurl
-------------------------------------------------------------------
libcurl is required for mesos to build.
-------------------------------------------------------------------
 `yum install libcurl-devel `
2.libsubversion-1 is required for mesos to build.

` yum install -y subversion-devel`
这两个都是之前命令中安装的东西但是到了这里又需要重新安装，应该是yum安装失败的问题
编译过程太慢了

3.执行make时出现错误
>Building mesos-1.4.0.jar ...
Exception in thread "main" java.lang.UnsupportedClassVersionError: org/apache/maven/cli/MavenCli : Unsupported major.minor version 51.0

这个主要是因为你的java环境低于1.7造成的，可以尝试升级jdk版本，
但是**我在升级之后明明java  -version提示的1.7，编译还是失败**，后来查看Makefile文件发现，文件中已经写死了jdk路径，比如这种：
`CONFIGURE_ARGS =  'JAVA_HOME=/opt/soft/jdk/jdk1.6.0_45`
最后通过修改(注意有好几个Makefile文件)
`vim build/src/Makefile`
`1,$s/jdk1.6.0_45/jdk1.7.0_79/g`（执行替换）

配置有两种
一个是 在通过命令行启动时添加选项 eg:--option_name=value
一个是 通过设置环境变量 eg:MESOS_XXX(XXX就是OPTION_NAME）

mesos master配置
必填项
--work_dir=/var/lib/mesos/master
这两个在单master时是不需要的
--zk=
--quorum==

我们使用环境变量来配置
echo 'export MESOS_work_dir=/var/lib/mesos'>>/etc/profile
source /etc/profile
进入build目录
执行`./bin/mesos-master.sh --ip=127.0.0.1 >/dev/null &`
启动master

mesos agent配置
必填项
--master=
host:port 
zk://host1:port1,host2:port2,.../path zk://username:password@host1:port1,host2:port2,.../path file:///path/to/file (where file contains one of the above)
--work_dir=VALUE 

`./mussh -H iplist3 -c 'echo "export MESOS_master=10.252.81.25:5050">>/etc/profile'`
`./mussh -H iplist3 -c 'echo "export MESOS_work_dir=/var/lib/mesos/agent">>/etc/profile'
`
`./mussh -H iplist3 -c 'source /etc/profile'`（mussh执行source无效，还是要逐个刷新）
`./bin/mesos-agent.sh`启动agent

agent启动报错，无法在master注册
XX exited event for master XX
这时要检查master的hostname是不是配置的ip地址，如果不一致使用hostname "name"命令设置为对应ip

在master访问http://127.0.0.1:5050/#/agents
就能看到启动的agents
![启动成功][1]


  [1]: /images/mesos.png