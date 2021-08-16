# OpenStack

## 简述

OpenStack从一开始，就是为了云计算服务的。简单来说，它就是一个操作系统，一套软件，一套IaaS软件。



## 云计算

+ 概念：云化就是把每个人手中的独立资源集中起来，放在一个地方进行统一管理，然后动态分配给每个人使用。而云计算，就是把计算资源集中起来，这个计算资源，包括CPU、内存、硬盘等硬件，还有软件。
+ 云计算资源
  + 应用
  + 数据
  + 服务
+ 运作模式
  + laaS（基础设施即服务）：提供硬件相关的服务。比如虚拟化、服务器、存储和网络
  + PaaS（平台即服务）：提供平台或中间件的服务，比如操作系统和中间件
  + SaaS（软件及服务）：提供直接的应用服务，比如收发邮件，订购商品
+ 优势
  + 规模大：云计算的服务器是百万级别的
  + 高可靠性：云计算采用各种容灾措施
  + 灵活性：根据需求动态分配
  + 低成本：规模效应带来的低成本
+ 主流技术
  + openstack
  + k8s *



## openstack概述

+ 概述：laaS软件

+ 关键特性：虚拟化和云计算

+ 核心组件

  + Identity——Keystone：服务认证及授权
  + Image Service——Glance：存储和检索虚拟机磁盘镜像
  + OBject Storage——Swift：存储和检索无结构数据对象
  + Dashboard——Horizon：人机交互界面
  + Compute——Nova：提供虚拟机服务，包括创建和热迁移
  + Block Storage——Cinder：提供块存储服务
  + Network——Neutron：提供网络连接服务，创建和管理虚拟网络

+ 基于MySQL和RabbitMQ

+ 简要架构图

  ![openstack](D:\img\brain\openstack.png)

说明：关联最多的两个模块为：Horizon即与用户直接关联的UI层，负责调用其他模块提供的接口为用户提供相关的服务，而Keystone负责认证与服务授权，Openstack中的所有服务实习访问控制。而与Horizon最直接关联的为Nova，其负责虚拟机的整个生命周期管理，而拥有一个虚拟机还需要基于其运行的OS，所以我们需要到Glance上检索获取合适的OS镜像，而最终的镜像存储维护由Swift实现。有了拉取到的OS镜像需要调用Cinder的块服务部署配置OS，之后的可以基于Neutron来为创建的虚拟机提供网络连接配置。

+ 创建实例流程

  + stage1：user发出创建实例申请

  ```mermaid
  sequenceDiagram
  	User ->> keystone:获取认证信息
  	keystone ->> User:返回请求auth-token
  	User ->> nova-api:携带token发送boot instance请求
  	nova-api ->> keystone:验证token
  	keystone ->> nova-api:返回验证结果
  ```

  + stage2：创建物理主机，rpc简单表示为通过消息队列的调用

  ```mermaid
  sequenceDiagram
  	nova-api ->> MySQL:初始化新建虚拟机的数据库记录
  	nova-api ->> RabbitMQ:向scheduler请求是否具有创建虚机机的资源
  	RabbitMQ ->> nova-scheduler:侦听队列获取请求
  	nova-scheduler ->> MySQL:查询数据库中计算资源情况，更新物理主机信息
  	nova-scheduler ->> nova-compute:发出创建虚拟机请求(rpc)
  	nova-compute ->> nova-condictor:获取虚拟机信息(rpc)
  	nova-condictor ->> MySQL:查询信息
  	MySQL ->> nova-condictor:返回信息
  	nova-condictor ->> nova-compute:回送虚拟机信息(rpc)
  ```

  + stage3：配置虚拟机，VM为配置的虚拟化驱动

  ```mermaid
  sequenceDiagram
  	nova-compute ->> glance-api:请求镜像
  	glance-api ->> keystone:验证token
  	keystone ->> glance-api:返回结果
  	glance-api ->> nova-compute:返回镜像URL
  	nova-compute ->> neutron:请求虚拟机网络信息
  	neutron ->> keystone:验证token
  	keystone ->> neutron:返回结果
  	neutron ->> nova-compute:返回网络信息
  	nova-compute ->> cinder:请求虚拟机持久化存储信息
  	cinder ->> keystone:验证token
  	keystone ->> cinder:返回结果
  	cinder ->> nova-compute:返回持久化存储信息
  	nova-compute ->> VM:根据信息和资源通过驱动创建虚拟机
  ```

+ 术语
  + RPC(Remote Procedure Call)远程过程调用：一个节点请求另一个节点提供的服务
  + RESTful API(Representational State Transfer API)表述性状态传递应用程序接口：网络中client和server的一种交互的形式
  + CURD：数据库增删改查
+ links：

  + RPC：https://www.jianshu.com/p/7d6853140e13
  + REST：
    + https://www.jianshu.com/p/84568e364ee8
    + https://zhuanlan.zhihu.com/p/97978097



## 部署方式

### RDO

+ 系统版本：CentOS7.8
+ 安装版本：train版
+ 配置步骤——all in one by packstack

1. 配置host文件，ip-域名-主机名

2. 配置网络：使用桥接模式，配置虚拟机OS的网络IP地址为静态地址，网络地址，网关和DNS服务器均与宿主机中相符。

   `桥接模式`：https://www.cnblogs.com/haoabcd2010/p/8683656.html

   `Linux网络配置`：https://blog.csdn.net/weixin_41072205/article/details/90488464

3. 配置内存和存储：从硬盘存储中创建一个新分区，并分配8G以上内存

   `Linux分区配置`：https://www.cnblogs.com/kerrycode/p/4612925.html

4. 更换yum源为国内源，配置可查看对应的使用说明

+ 问题
  + pip包安装
    + yum install python-pip
    + pytz
    + pyyaml
  + 循环导入（循环依赖）问题
    + yum downgrade leatherman
  + https服务问题
    + yum install httpd
    + SELinux：setenforce 0
  + mysql问题
    + 数据库链接时用户认证错误
  + 创建实例时一直schedule——MQ等待错误
    + 重启nova相关服务
    + 所有openstack服务的命名格式为openstack-*
  + 创建实例错误
    + 在VM中修改虚拟机配置支持CPU虚拟化
    + 开启虚拟化
  + links：
    + 循环依赖：https://blog.csdn.net/weixin_43905458/article/details/104898789
    + mysql密码变更：https://blog.csdn.net/weixin_34562922/article/details/116633739?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0.control&spm=1001.2101.3001.4242
    + nova密码配置：https://blog.csdn.net/Miracle1203/article/details/102730311
    + nova-compute启动管理等：https://www.cnblogs.com/goldsunshine/p/8675462.html
    + 开启主板虚拟化：

```ssh
# 运行脚本，通过packstack安装前运行
yum install python-pip
pip install pytz pyyaml

yum downgrade leatherman

yum install httpd
setenforce 0   #建议永久关闭selinux
systemctl start httpd
systemctl enable httpd

yum install tuned
systemctl start tuned
systemctl enable tuned

#虚拟机开启CPU虚拟化
```



+ 配置更改
  + 增加节点
    + 配置compute节点
    + 修改answerfile文件——COMPUTE_HOSTS
  + 采用SSL链接
    + 修改answerfile文件——Horizon_SSL






### Kolla

+ kolla：容器镜像构建
+ kolla-ansible：容器部署
+ kolla-kubernetes：容器部署与管理



其他安装部署方式：https://cloud.tencent.com/developer/article/1491077



## 运行使用

### 网络配置

+ 桥接器br-ex——连接外网

```shell
ovs-vsctl show
```

+ 配置过程

1. 创建外部网络，并划分子网和创建IP地址池

```sh
neutron net-create external_network --provider:network_type flat --provider:physical_network extnet  --router:external
neutron subnet-create --name public_subnet --enable_dhcp=False --allocation-pool=start=192.168.1.90,end=192.168.1.100 --gateway=192.168.1.1 external_network 192.168.1.0/24
```

2. 创建内部网络
3. 创建路由器，并配置接口连接保证内外网互通



### 磁盘管理

+ 由cinder从cinder-volumes划分
+ 步骤

1. 新建卷然后关联到Instance上扩展硬盘
2. 重启虚拟机
3. 创建分区
4. ...



### 镜像配置

+ 获取云端镜像
  + 官方Url：https://download.cirros-cloud.net/
+ 创建镜像



### 用户认证及管理

+ Browser登录：浏览器中访问本机ip地址即可跳转到dashboard界面

+ 命令行登录

```sh
#默认的登录配置文件路径为/root/keystonerc_admin
source /root/keystonerc_admin
```

+ 创建用户

1. 新建一个Project，并限制物理资源配置
2. 新建一个User，并分配到该Project，设置Role(权限的抽象表述)

+ 创建公私钥对
  + 私钥文件路径：/tmp/mozilla_root0
+ 创建安全组——配置类似防火墙





### 实例创建

+ 创建主机类型Flavor
+ 创建实例Instance
  + 创建时配置内容
    + 镜像
    + 主机类型
    + 网络
    + 安全策略
    + SSH Key
  + 分配外网IP
  + Console链接或SSH链接







## Encryption

+ 功能：加密存储或传输
+ 服务粒度：单用户粒度
+ 加密对象：
  + 对象存储对象
  + 网路数据
  + 块存储对象——Swift
+ 加密分类
  + 卷加密
  + 磁盘加密

