##RocketMQ单机环境搭建（二）
>备注：
>笔者单机环境搭建过程参照：Quick Start-github-alibaba/RocketMQ
>本篇文章为Quick Start的解释和补充，为了您更好的理解，请参照Quick Start阅读
>传送门：https://github.com/alibaba/RocketMQ/wiki/quick-start
>
>作者：Nick



###安装环境

您需要安装以下软件：

* 64位操作系统，最好有Linux / Unix / Mac;
* 64bit JDK 1.6+;
* Maven 3.x
* Git
* screen(方便后台运行服务，不安装也可以)

笔者环境：centos7 通过yum一键安装，首先保证联网，ping通
```
yum install git // 安装git

yum install java //安装JDK

// 安装maven，如果执行yum install apache-maven提示报错，输入第一行命令-wget

wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo

yum -y install apache-maven

yum install screen  // screen安装

```

* yum默认安装的jdk是open jdk，经笔者测试open jdk同样可用，只要保证jdk唯一就行。
* 建议maven和jdk统一采用yum安装，因为maven依赖java环境，如果环境为oracle jdk而yum默认安装的maven依赖的是open jdk，后面会遇到冲突问题。
* screen的作用是方便后台运行namesrv或broker服务，其他文章里时常可以遇到`nohup`命令，或者`./xx服务 &`都是类似的作用。

###Clone & Build

打开终端，去某个有权限创建文件和文件夹的地方


1. git clone https://github.com/alibaba/RocketMQ.git
2. cd RocketMQ
3. bash install.sh

耐心等待，安装成功后可以看到devenv文件夹，内含所有必须的文件和库。其实这个文件是 /RocketMQ/target/alibaba-xxxx的快捷方式

###环境变量

确保正确设置以下环境变量：JAVA_HOME   
>jdk环境会对安装造成很大影响，请保证单一jdk使用环境

现在设置ROCKETMQ_HOME环境变量：
```
cd devenv  //进到devenv文件夹

echo "ROCKETMQ_HOME=`pwd`" >> ~/.bash_profile   //将当前路径赋予全局常量写到.bash_profile中，方便后面调用
```
使新环境变量生效：
```
source ~/.bash_profile
```

###启动RocketMQ name server和broker

在devenv目录下`cd bin`

* 启动Name Server服务(前文比喻的信号塔)
  `screen bash mqnamesrv`
  或者在bin目录，直接使用  `./mqnamesrv &`

如果您看到`The Name Server boot success. serializeType=JSON`，这意味着名称服务器成功启动。按`Ctrl + A`，然后按`D`断开会话。

* 启动Broker服务
```
screen bash mqbroker -n localhost:9876 
//-n 是mqbroker服务一个可选参数，意为选择nameserver地址，此处默认本机ip，端口9876
```

如果看到如下输出：
>broker[xxxxx-Lenovo，172.30.30.233:10911]启动成功。 serializeType = JSON和名称服务器是localhost：9876

表示Broker成功启动，Broker默认的端口10911，可以在broker.properties中修改

您还可以检查日志文件以验证如下：

`tail -f ~/logs/rocketmqlogs/broker.log`

检查是否有类似于续集文本的心跳事件：

>2016-xx-xx xx:xx:xx INFO BrokerControllerScheduledThread1 - 注册broker到名称服务器localhost：9876 OK

还可以输入命令`jps`查询当前java进程，此时应该可以看到NamesrvStartup,BrokerStartup，说明上述两个服务成功启动了

###Send & Receive Messages

在发送/接收消息之前，我们需要告诉客户端名称服务器所在的位置。为了简单起见，我们使用环境变量`NAMESRV_ADDR`，（producer/consumer初始化时会调用NAMESRV_ADDR，设置成环境变量或者代码里直接声明都可以）

`export NAMESRV_ADDR = localhost:9876`

现在我们准备好发送/接收消息。

`bash tools.sh com.alibaba.rocketmq.example.quickstart.Productcer`
你会看到几百条消息被发送给broker。

要使用刚刚在上一步中发送的消息，

`bash tools.sh com.alibaba.rocketmq.example.quickstart.Consumer`

到此单机环境搭建成功，并成功启动了Producer和Consumer进行了消息收发的测试。

***

### 下面开始划重点

>nameserver端口默认为9876
>broker监听端口默认为10911

接着[（一）RockeMQ初步认知](www.xxxx.com)中的介绍的四个角色，根据本章实践复习和补充下：

**Producer：**

* 一般由业务系统产生消息，通过topic来标示

* 存在Producer Group的概念（后文涉及）

**Consumer：**

* 有两种实现方式，一种push Consumer，第二种是 pull Consumer。

* 本章启动的示例Consumer为push consumer。在consumer对象注册一个listener接口，一接收到消息，consumer对象立刻回调 Listener方法。
* push Consumer和pull Consumer的区别简单理解：两者基本原理相同，push是封装好了，傻瓜调用。pull需要自主实现，由应用控制。（深入研究后再行补充）

* 存在Consumer Group的概念（后文涉及）

**Broker**：

* 消息中转角色，负责存储消息，转发消息。

* 拥有Master、slave（主备）的概念，主备有同步复制、异步双写功能来保持数据同步（未深研）。标识：Master的BrokerId 为 0 ，Slave的BrokerId 非0。
  * 部署模式：
    * 单Master无Slave（脆弱）
    * 单Master多Slave（单点故障就瘫，开源版无主备切换功能）
    * 多Master无Slave（无单点故障，线上生产常用模式）
    * 多Master多Slave（无单点故障）

* Broker 与 Name Server 集群中的**所有节点建立长连接**，定时（心跳）注册 Topic 信息到所有 Name Server。

**Name Server**：

* 无状态节点，可集群部署，**节点之间无任何信息同步**（Broker与每个namesrv连接，可以保证信息同步性）

----------

##FAQ
1. 本地网络连接正常，（linux）联网失败，ping不通？
   **解决方案**：修改 /etc/resolv.conf    dns 文件 添加 nameserver 8.8.8.8   && 8.8.4.4

2. Directory '/var/run/screen' must have mode 777，执行screen命令提示权限不足？

   **现象描述**：
   ![FAQ-问题2报错截图](http://img.blog.csdn.net/20161129143534204)

   **解决方案**：`chmod -R 777 /var/run/screen` 赋予报错中的文件路径以777权限，`-R`对其子文件也生效

3. 启动 mqnamesrv / mqbroker服务报错，显示内存不够，需大于2G？
   **现象描述**：
   "_VM warning: INFO: OS::commit_memory(0x00000006c0000000, 2147483648, 0) faild; error='Cannot allocate memory' (errno=12)_"

   ![FAQ-问题3报错截图](http://img.blog.csdn.net/20161129143632784)

   **解决方案**：修改`/RocketMQ/devnev/bin/` 下的服务启动脚本  `runserver.sh` 、`runbroker.sh` 中对于内存的限制，​改成如下示例：

   ```
    JAVA_OPT="${JAVA_OPT} -server -Xms128m -Xmx128m -Xmn128m -XX:PermSize=128m -XX:MaxPermSize=128m"
   ```


4. UseCMSComactAtFullCollection is deprecated and will likely be removed in a future release.java.net.BindException: 地址正在使用？

   **问题原因**：上述情况是系统存在多个Jdk同时使用导致的，如本文开头安装maven时讲到的冲突问题

   **解决方案**：删除/停止多余jdk，__保证系统使用唯一jdk源__。

5. 启动Broker遇到提示cannot stat '/root/rmq_bk_gc.log':No such file or directory：

   ```
   cp:cannot stat '/root/rmq_bk_gc.log':No such file or directory
   OpenJDK 64-Bit Server VM warning: Cannot open file /root/tmpfs/logs/gc.log due to No such file or directory
   ```

   **解答**：经测试，上叙情况不会影响Broker的正常启动，出现此提示请耐心等待Broker服务启动。如图所示：

   ![FAQ-问题5截图](http://img.blog.csdn.net/20161129143751176)

6. 待补充，欢迎留言~~~

