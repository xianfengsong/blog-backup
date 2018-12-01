title: Shell in Action（一）文本编辑-修改hosts
date: 2015-12-01 00:32:36
categories: 
tags: [shell,Linux]
---

　把自己工作环境换成linux之后总会遇到各种无语的问题，比如我在web开发时会经常要切换测试环境或者本地环境的hosts，但是在用firefox的hosts插件时发现每次修改都会卡死，最后忍无可忍打算写一个切换hosts环境的脚本，有问题欢迎指正~



##问题分析
　我们都知道hosts文件就长这样，#是注释符、ip和域名用空格分开
下面是测试文件testhosts，用DEV和TEST区分不同环境：

```
#DEV
74.125.207.84 accounts.a.com
74.125.207.83 accounts.b.com
#TEST
64.233.168.106 www.c.com
64.233.168.107 www.d.com
#END
```
**解决步骤**


　　1.读取用户要切换的环境
　　2.读取hosts文件，在指定的行前添加注释
　　3.维护一个值 保存hosts所处环境 提示用户当前hosts环境
<!--more-->
###一、修改host文件

##ed命令
　ed （edit）命令 可以逐行的修改文本，[[a]](#1)分为‘寻址’‘操作命令’‘文件名’三个部分
　ed [address]command textfile
在命令行 输入 `info ed` 查看ed完整说明 
**寻址**

| 选项        | 说明           |
|:------------- |:-------------| 
| number      | 第number行，从0开始 | 
|+number      | 从本行后第number行开始 | 
| -number      | 从本行前第number行开始 | 
|   ，    | 1,3表示从1到3行 一个逗号表示全文 | 
| /pattern/|下一个包含/partten/的行|
| \$partten\$|上一个包含partten的行|

**命令部分**

| 选项        | 说明           |
|:------------- |:-------------| 
| a  | add 追加文本 | 
|c | change| 
| d   | 删除当前指向的行 | 
| i   |insert | 
| wq   |和vim一样 write&quit | 
| s/pattern/replace/  |将符合pattern的替换为replace文本 | 
ed操作文本时会将文件拷贝到缓冲区，在编辑后写入文件，在命令行输入ed后程序会等待用户输入，我们用echo和管道向ed发送指令

**eg1:** 在testhosts第一行添加文本（句点表示退出add模式 相当于vim的ESC）
```shell
(echo '0a';echo '#hosts for test';echo '.';echo 'wq')|ed testhosts
```
**eg2:** 把.com替换为.com.cn
```shell
(echo ',s/com/com.cn/';echo '.';echo 'wq')|ed testhosts
```
##sed命令
　sed（stream edit），接受输入流并进行编辑，再把结果写到输出流；
由于sed是面向流的，在使用时我们要通过管道为sed指定输入流并将输出重定向到修改后的文件: 
`cat file|sed 'command' file > newfile`
或者使用"sed -i file"可以直接编辑并保存到文件 :
`sed -i 'command' file`

**sed**也是对文件逐行编辑，寻址方式和ed基本相同，但是不支持对匹配的地址进行+n/-n操作。

**eg**: 使用sed将testhosts文件DEV与TEST间的hosts注释掉
`sed -i '/#DEV/,/#TEST/s/ /#' testhosts`

###二、创建永久环境变量
    
　在使用环境变量要注意：

 - 当前命令行shell和使用sh或./执行脚本创建的子shell不在同一进程中
 - 普通自定义变量不会在子shell中生效，而全局环境变量可以在子shell中使用
 - 当前命令行定义的环境变量会在退出shell退出后失效
![举个例子][1]


在命令行输入:
``` 
root@song-pc:/# echo $$
30687
root@song-pc:/# export global="parent global"
```

创建脚本child.sh：

```
#!/bin/bash
echo $$;
echo $global;
export global="child global";
```
执行脚本，打印global
```
root@song-pc:/# ./child.sh 
30794
parent global
root@song-pc:/# echo $global
parent global
```
$变量表示当前shell的进程id，可见命令行执行的脚本属于另一个进程，而且子进程定义的环境变量在父进程中无效；
如果要脚本中对环境变量的修改生效,可以使用source命令（.命令）执行脚本，这用方式会在当前进程中执行脚本，但是我们定义的环境变量仍在shell关闭后失效；

**通过文件保存变量：**
linux系统下环境配置通常保存在这几个文件中[[b]](#2)：
- /etc/profile:System wide environment and startup programs, for login setup（所有用户可用）
- /etc/bashrc ：System wide functions and aliases（每一个运行bash shell的用户执行此文件.当bash shell被打开时,该文件被读取）
- ~/.bashrc：包含专用于你的bash shell的bash信息,当登录时以及每次打开新的shell时,该文件被读取.（每个用户都有一个.bashrc文件，在用户目录下）
 
为了保证记录hosts环境的变量不会失效，我选择修改 .bashrc的方式，在修改文件后执行 source命令保存，同时可以避免其他用户修改。

###三.编写脚本
**switch_hosts.sh:** 
```  
#!/bin/bash
#切换本机hosts环境
#2015-10-15 sxf 2.0 
file="/etc/hosts";
type=0;
env[1]="开发" env[2]="测试"  env[3]="线上";

echo "现在是 ${env[$HOSTS]} 环境";
read -p "选择切换到：1.开发  2.测试  3.线上 : " type;

if [ $HOSTS -eq $type ];
then
        echo "环境不变"
        return
fi
case $type in
1)
        echo "正切换到开发环境。。"
                sed -i '/#DEV/,/#END/s/^ /# /' $file;
                sed -i '/#DEV/,/#TEST/s/# / /' $file;
        ;;
2)
        echo "正切换到测试环境"
                sed -i '/#DEV/,/#END/s/^ /# /' $file;
                sed -i '/#TEST/,/#END/s/# / /' $file;
        ;;
3)
        echo "正切换到线上"
                sed -i '/#DEV/,/#END/s/^ /# /' $file;
        ;;
*)
        echo "输入错误"
        return
        ;;
esac
sed -i "s/HOSTS=$HOSTS/HOSTS=$type/" .bashrc;
source .bashrc;
cat $file;
service nscd restart;
```
为了使source .bashrc命令生效，要使用source switch_hosts.sh的方式执行，service nscd restart命令用来刷新DNS缓存；


[a] <span id = "1">http://biancheng.dnbcw.info/shell/242647.html</span>
[b] <span id = "2">http://blog.csdn.net/chenchong08/article/details/7833242</span>

  [1]: http://7xl4v5.com1.z0.glb.clouddn.com/example.jpg
