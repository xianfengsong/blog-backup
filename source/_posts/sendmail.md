title: Sendmail的一些坑
date: 2015-12-26 23:05:46
categories: 
tags: [邮件]

---
## 简介
Sendmail是一个linux邮件服务系统，可以使用它来搭建邮件服务器。

 关于邮件的几个名词：
>MTA(Mail Transfer Agent)                             邮件传送代理，运行在邮件服务器的程序，负责接收发送邮件  

>MUA(Mail User Agent) 用户端代理，提供查看编辑提交邮件的功能（如foxmail）  

  >MDA(Mail Delivery Agent)主要的功能就是将MTA接收的信件依照信件的流向（送到哪里）将该信件放置到本机账户下的邮件文件中（收件箱），或者再经由MTA将信件送到下个MTA。如果信件的流向是到本机，这个邮件代理的功能就不只是将由MTA传来的邮件放置到每个用户的收件箱，它还可以具有邮件过滤（filtering）与其他相关功能[^foot]。

<!--more-->

## 安装
我使用的是CentOS6.2 sendmail 8.14.4

```shell
#安装sendmail和配置工具sendmail-cf
yum install -y sendmail
yum install -y sendmail-cf
#SMTP认证服务
yum install -y saslauthd
```
## 配置sendmail
sendmail的配置文件主要是XXX.mc文件和XXX.cf文件，mc后缀文件可以看做是配置文件的人类阅读版，通过sendmail-cf工具将它生成为cf后缀的外星人版配置文件   
>vim /etc/mail/sendmail.mc

具体操作请参考--[传送门](http://www.centoscn.com/CentosServer/lighttpd/2013/0726/650.html)

## 常见问题
### 1.发出的邮件被其他MTA识别为垃圾

设置域名指向（上节链接），添加spf记录 [操作教程](https://www.renfei.org/blog/introduction-to-spf.html)
同时注意控制发信频率
### 2.在哪查看sendmail日志?
**日志位置**： /var/log/maillog  
**调整日志级别（详细程度）**： 修改配置文件define(`confLOG_LEVEL', `16')dnl 默认为9
或者调用命令时指定sendmail -O LogLevel=14
[日志级别说明](http://www-01.ibm.com/support/knowledgecenter/ssw_aix_71/com.ibm.aix.networkcomm/sendmail_debugflags.htm?lang=zh)
### 3.sendmail日志都是什么意思?
**注意sendmail并不保证发出的邮件一定会被发到收件人接收**，所以日志中的信息只是接收端MTA反馈的连接信息。
具体格式，慢慢读吧-----[sendmail日志格式](http://www.softpanorama.net/Mail/Sendmail/sendmail_logs_format.shtml)
### 4.sendmail性能参数
http://www.5dmail.net/html/2008-4-27/200842733006.htm  
### 5.一些相关的网站，文章
http://www.5dmail.net/  
**sendmail官网**https://www.sendmail.com/sm/open_source/
http://park.jobdeer.com/discussion/19/%E9%82%AE%E4%BB%B6%E5%8F%91%E9%80%81%E9%82%A3%E7%82%B9%E4%BA%8B
**看邮件头**http://www.qqexmail.net/tips/st_security_look_head.asp

[^foot]:http://wenku.baidu.com/link?url=iK74BiQS1clzFxTy4B4eBcG6u4gR9EMZ-CY58DwhrNqsJSYt6P9nk-aUzGPqXxgoTM5qAj51J4F-aPUhf3vaidDopNr6SrU3VgwPhWwHws_
