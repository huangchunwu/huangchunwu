---
title: habase应用查询慢，hbase shell查询快
date: 2020-07-02 23:49:48
tags: hbase
categories: 技术
---


## 问题现象

今天早上，我还在上班路上，测试老大在群里面喊，xx应用仿真环境访问不了，并且截图了log日志，我看了一下是dubbo服务访问超时

，第一反应是dubbo服务挂了，找运维重启，重启后无果，然后等我去了公司，看了详细日志，是dubbo接口响应时长达到6s，明明是测试通过的接口，接口性能不可能这样慢， 分析了下这个接口功能，是直连hbase查询，还是rowkey的get查询，应该是几十毫秒内响应。遇到此类，本来好好的，现在不行的问题，一般都是一脸问号，没办法，只能撸起袖子找原因了。

### 第一步：hbase数据量是否大了，接口rowkey查询性能问题。

我连到了hbase服务器上，执行`hbase shell` 

```shel
get "db.user_acc_profit","23342_1_9"

```

耗时 15ms。

```shell
count "db.user_acc_profit"
1200 rows
```

完蛋，hbase没有问题，才1200条数据。只能检查代码去，是否有慢查。

### 第二步：检查代码逻辑，是否有慢查

看了下代码，只是单rowkey查询，没有任何慢查，此时，我陷入了迷茫了，只能靠猜了，我先去开发环境，执行该接口，发现没有问题，只能说明仿真环境真的有问题，但是问题到底是什么，我不知道。



### 第三步：网络传输

经验告诉我，此时不要慌，先把问题捋清楚，

当前的现象是：

- 同样的代码，在开发执行hbase查询很快，在仿真很慢。

- 单个hbase shell查询很快，说明hbase本身没有问题

根据这2点分析，hbase 自身shell 命令查询很快，但是应用查询慢，说明问题出现在应用与hbase的交互上，是网络传输做了限制吗？

带着这个问题，询问了运维，答案是no。我怀疑是dns配置的问题，运维的回答是这台机器用的是本地host解析的ip。就此打断了我对网络的猜测。

### 第四步：hbase服务器健康状态

接着，我又问了管理hbase的运维老师，他打开监控页面，告诉我，hbase服务器很健康。我去hbase服务器上查了下cpu负载

`top` 了一下，cpu很正常。又查了一下服务器内存，`free -m`  ，内存也是足够的，又查了下磁盘，`df -h ` ,也正常。

由于hbase是集群的，服务器比较多，我只是抽查了几台，问题就出在抽查。

后来不经意，通过程序里面配置的几台hbase的服务ip，发现有一台服务器宕机了，询问了一下前面管hbase的运维老师，他告诉我这台机器运营商硬件故障一直没有恢复，也没办法恢复了，查到这里，仿佛破云见雾般的感觉，我捋顺了思路：hbase shell查询快，说明单机的hbase服务器快，而我之前忽视了程序访问的是一个hbase集群来查询的，如果单台机器挂了，每次发起查询命令的时候，都会从配置的集群列表里面选取一台去查询，然而某台机器挂了，就会重试，重试失败后，再选择其他健康服务器去接着查询，所以出现了habse查询响应慢的现象。

 然后，我找运维老师把程序的hbase配置给改了一下后，测试了下程序，查询速度恢复正常了。

## 总结

公司的服务器维护治理，存在问题，此类服务器停机没有及时更新应用配置。

我在思考，为啥我花了那么久才查找问题，不能第一时间抓住要害。

当我看到“shell 命令查询没问题，应用程序查询有问题”情形，我的脑回路没有立刻提取到有用的信息：“hbase shell查询的单机，应用查询是走的集群查询”，是因为我没有这个常识吗？不是的。也许这就是知识与实践结合的原因，最后送自己一句话，“你知道不代表你会”。

