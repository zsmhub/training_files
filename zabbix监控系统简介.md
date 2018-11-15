## zabbix监控系统简介

### 为什么需要监控系统

公司服务器缺乏一个统一的监控报警平台，邮件报警监控凌乱，每次出故障都不能及时的发现或者根本未发现，或者没有图形界面的展示效果好。

### 监控系统需要做的事

对基本的硬件网络系统监控外，还需要对：hadoop集群的各个进程服务状态、服务性能，mfs同hadoop监控需求，nginx、mysql、php-fpm的性能和进程状态监控，mysql主从复制状态，docker容器状态和性能，web页面的性能监控等等。

###  监控系统的差异

好用功能强大的要收费，新型监控系统功能少不成熟，则中选择老牌开源监控系统。
###  zabbix的优势

老牌开源监控系统，监控与报警的统一平台，监控插件丰富扩展性强，网上资料多，系统稳定，支持分布式监控解决数目众多的机器监控问题，自带多种监控模版，支持自动发现功能，可以实现批量自动化添加和监控主机，lld的动态监控项的批量监控，支持的监控数据采集方式多样，比如 ： SNMP,IPMI,agentd,JMX。

### zabbix组件

一般由`zabbix server`，`zabbix agentd`，`zabbix proxy`，`database`，`web interface`组成。

组件功能：

1. `Zabbix Server`：负责接收agent发送的报告信息的核心组件，所有配置，统计数据及操作数据均由其组织进行；

2. `Database Storage`：专用于存储所有配置信息，以及由zabbix收集的数据；

3. `Web interface`：zabbix的GUI接口，通常与Server运行在同一台主机上；

4. `Proxy`：可选组件，常用于分布监控环境中，代理Server收集部分被监控端的监控数据并统一发往Server端；

5. `Agent`：部署在被监控主机上，负责收集本地数据并发往Server端或Proxy端；

注：`zabbix node`也是 `zabbix server`的一种 。

关系图：

![](http://i.imgur.com/TWTfPBS.png)



### 几种zabbix获取数据的方式

#### zabbix_trapper

zabbix获取数据有超时时间，如果一些数据执行时间较长就会出现异常,客户端自己提交数据给zabbix，交给`zabbix_trapper`来解决。
#### zabbix代理（主动）

`zabbix agentd`主动向server端发送数据。

#### zabbix代理（被动）

当监控的item比较多队列大，可以采用被动模式，简单的说就是agent主动的从server端去下载需要监控的item然后将数据上传到server。

### 监控实现

#### 部署

1. 使用ansible来分发和执行代理端的安装和配置脚本。
2. 安装和配置服务端和mysql服务器。
3. 在web页面配置discovery，通过IP段来批量接入服务器到监控中。
4. 主机的信息编辑配置。
5. 添加主机群到基础监控模版。
6. 规划好ansible的hosts组。

#### nginx、php-fpm、mysql的性能监控及mysql主从状态


1. 分别修改nginx、php-fpm的配置。
2. 用ansible发布配置文件到主机群。
2. 用ansible部署nginx、php-fpm、mysql的监控脚本到web服务器组。
3. mysql主从状态监控部署在从服务器上。
3. 导入模版，将主机群加入到模版。

#### docker的监控

1. docker监控首先要实现自动发现容器，利用zabbix的`low-level-discovery`就可以实现。
2. 首先代理端根据配置运行low discovery脚本获取到一份json格式的数据就包含容器名等等。
3. server端根据容器名的item，提供给代理端，代理端根据容器名获取相应容器的性能数据。
4. 部署同上。


#### hadoop的监控

1. 将hadoop集群的主机角色进行细分，然后利用ansible来根据群组来批量分发配置运行脚本等等。
2. 整个hdoop集群的进程状态利用上述方法实现。
3. hadoop集群的性能监控，获取hadoop namenode自带的web监控页面节点的一些数据。
4. 部署同上。

#### mfs的监控

1. 对mfs的挂载，mfs进程状态做了监控，还有master上面对整个集群的一些重要信息，比如：chunks总数、是否存在missing chunks等等进行了监控。



### 脚本来源

直接用，修改，自写。


### 遇到的坑

1. 使用zabbix_get在server端测试获取数据时有时候会出现获取不到值，或者不支持该key之类的报错。首先在server端需要su - zabbix然后执行get测试，因为部署的zabbix的代理端是以zabbix身份，所以有一些脚本中可能涉及的命令或者调用只能root用户才能成功获取到数据，对于这有两种解决方法：将该命令加入到/etc/sudoers中，或者将数据获取写到计划任务以root 身份运行写到文件，然后代理端从文件中取值。
2. 做docker监控时，github上有一个用c写的扩展，性能是自定义脚本效率的十倍上，苦苦的部署下来，发现并不能很正确的获取到值，无奈只能使用自定义脚本。
3. 针对zabbix的lld，开始有批量部署时发现数量一多起来，批量操作的速度好慢，网上有自写的脚本来替换。
4. 对mfs性能监控，网上搜索资料全无，官网看文档只有命令介绍也没有接口之类的，回想一下自带的监控页面可以获取到数据，然后是一个cgi脚本，然后就看代码，将html过滤掉，导出重要的代码，mfs使用python写了一个简单的web服务器和服务端，然后又一些不同的指令来获取不同的值，github上面的源码，找到那些指令的意思，然后去get值了，结果不知道是get太快还是怎么回事，cgiserver直接坏了，连带master也坏了，所以为了稳定只部署在master 节点上。

### 还差什么

1. 对内网服务器的监控,proxy的被动模式需要开启nat端口映射
2. 一系列触发器和报警
3. 是否可以用微信来替代邮件报警通知
4. 优化

#### 接下来解决

内网服务器监控：使用`zabbix proxy`的分布式结构来实现对内网机器的监控

架构图：

![](http://i.imgur.com/DmAKdf1.png)


工作过程：

`Zabbix proxy`是一个监控代理服务器，它收集监控到的数据，先存放在缓冲区，保存的时间可以通过配置文件设定，然后再传送到`Zabbix server`；监控代理需要一个单独的数据库。

`zabbix proxy`的好处：

1. 远程监控
2. 当监控的位置通信不便时
3. 当监控上千的设备时
4. 简化维护分布式监控，降低`zabbix server`的负载

`zabbix proxy`的主被动模式：

proxy主动发送数据给server的就是主动模式，proxy等待server的请求在发送数据的是被动模式。主动模式能够减轻server的压力。如果proxy架设在内网，内网网关需要开启nat端口映射。

proxy所支持的功能：

![](http://i.imgur.com/qMp7vOe.jpg)






