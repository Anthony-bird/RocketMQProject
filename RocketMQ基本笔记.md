## RocketMQ基本笔记

### 1.安装

```shell
 cd /usr/local/  ，(先创建一个rocketmq的文件夹)安装到本地目录,在进入安装软件地址，移到指定文件夹位置，
 mv rocketmq-all-4.4.0-bin-release /usr/local/rocketmq/
 
 启动mqnamesrv：
 进入bin目录，在后台启动，nohup sh mqnamesrv &
 查看日志：
 tail -f ~/logs/rocketmqlogs/namesrv.log
 启动mqbroker:
 nohup sh mqbroker -n localhost:9876 &
 查看日志：
 tail -f ~/logs/rocketmqlogs/broker.log 
```

启动之后，查看日志是，发现如下错误，





![image-20201209003352700](C:\Users\PengYu\AppData\Roaming\Typora\typora-user-images\image-20201209003352700.png)



查看了nohup.out 

![image-20201209003519822](C:\Users\PengYu\AppData\Roaming\Typora\typora-user-images\image-20201209003519822.png)

修改了runserver.sh和(runbroker.sh)里面的配置，修改之后发现非常正确。

```JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:MetaspaceSize=128m  -XX:MaxMetaspaceSize=320m"```

用jps可以查看启动进程。

> 用本地虚拟机安装的时候找不到命令，使用 find / -name jps 可以验证，用 rpm -qa |grep -i jdk 检查到缺少openjdk-devel包，查看安装的版本 yum list *openjdk-devel*，安装这个包 yum install java-1.8.0-openjdk-devel.x86_64，之后就可以使用jps了。



### 2.关闭

```shell
# 1.关闭NameServer
sh mqshutdown namesrv
# 2.关闭Broker
sh mqshutdown broker
```

###  3.测试RocketMQ



发送

```sh
# 1.设置环境变量
export NAMESRV_ADDR=localhost:9876
# 2.使用安装包的Demo发送消息
sh tools.sh org.apache.rocketmq.example.quickstart.Producer
```

接收

```shell
# 1.设置环境变量
export NAMESRV_ADDR=localhost:9876
# 2.接收消息
sh tools.sh org.apache.rocketmq.example.quickstart.Consumer
```



### 4.配置集群

(刚开始配置的时候不分目录会出现问题，出了问题可以看后面的bug更改)

```sh
两台机器的IP  
192.168.43.167(虚拟机的)m1 s2
172.22.58.136(云服务器的)  m2 s1

vim /etc/hosts

# nameserver
192.168.43.167 rocketmq-nameserver1
172.22.58.136 rocketmq-nameserver2
# broker
192.168.43.167 rocketmq-master1
192.168.43.167 rocketmq-slave2
172.22.58.136 rocketmq-master2
172.22.58.136 rocketmq-slave1

#重启网卡
systemctl restart network

暴力配置法
# 关闭防火墙
systemctl stop firewalld.service 
# 禁止firewall开机启动
systemctl disable firewalld.service

# 查看防火墙的状态
firewall-cmd --state 
服务器安全部署
# 开放name server默认端口
firewall-cmd --remove-port=9876/tcp --permanent
# 开放master默认端口
firewall-cmd --remove-port=10911/tcp --permanent
# 开放slave默认端口 (当前集群模式可不开启)
firewall-cmd --remove-port=11011/tcp --permanent 
# 重启防火墙
firewall-cmd --reload

#配置环境
vim /etc/profile

在文件末尾配置
#set rocketmq
ROCKETMQ_HOME=/usr/local/rocketmq/rocketmq-all-4.4.0-bin-release
PATH=$PATH:$ROCKETMQ_HOME/bin
export ROCKETMQ_HOME PATH

使文件生效
source /etc/profile

#创建保存文件的路径
mkdir /usr/local/rocketmq/store
mkdir /usr/local/rocketmq/store/commitlog
mkdir /usr/local/rocketmq/store/consumequeue
mkdir /usr/local/rocketmq/store/index


```

配置broker文件

master1(本地虚拟机)

```sh
vim broker-a.properties(进入rocket文件夹conf里面的2m-2s-sync)
```



```sh
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store1
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store1/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store1/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store1/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store1/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store1/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=SYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```



slave2(本地虚拟机)

```sh
vim broker-b-s.properties
```



```sh
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-b
#0 表示 Master，>0 表示 Slave
brokerId=1
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=11011
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store2
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store2/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store2/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store2/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store2/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store2/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```



master2(阿里云)

```sh
vim broker-b.properties
```



```sh
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-b
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=SYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```



slave1(阿里云)

```sh
vim broker-a-s.properties
```



```sh
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=1
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=11011
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```

### 4.启动集群

```sh
分别在两台机器上启动启动NameServer，
其次再启动broker集群内
master1：

nohup sh mqbroker -c /usr/local/rocketmq/rocketmq-all-4.4.0-bin-release/conf/2m-2s-sync/broker-a.properties &  (要进入bin目录下)


slave2：
nohup sh mqbroker -c /usr/local/rocketmq/rocketmq-all-4.4.0-bin-release/conf/2m-2s-sync/broker-b-s.properties &

master2：nohup sh mqbroker -c /usr/local/rocketmq/rocketmq-all-4.4.0-bin-release/conf/2m-2s-sync/broker-b.properties &

slave1:nohup sh mqbroker -c /usr/local/rocketmq/rocketmq-all-4.4.0-bin-release/conf/2m-2s-sync/broker-a-s.properties &

```

### 重要bug！！！(出错了必须看)

启动broker会出现的问题，第二个broker始终无法启动，

![image-20201209234731822](C:\Users\PengYu\AppData\Roaming\Typora\typora-user-images\image-20201209234731822.png)

我后来查看了nohup.out

发现一直显示我已启动MQ，很无语

![image-20201209234914946](C:\Users\PengYu\AppData\Roaming\Typora\typora-user-images\image-20201209234914946.png)

我查了原因之后，发现master和slavestorePathRootDir是一样的，会引起错误冲突，

需要把目录更改即可。

```sh
在两台主机上面分别更改
#重新创建保存文件的路径

mkdir /usr/local/rocketmq/store1
mkdir /usr/local/rocketmq/store1/commitlog
mkdir /usr/local/rocketmq/store1/consumequeue
mkdir /usr/local/rocketmq/store1/index

mkdir /usr/local/rocketmq/store2
mkdir /usr/local/rocketmq/store2/commitlog
mkdir /usr/local/rocketmq/store2/consumequeue
mkdir /usr/local/rocketmq/store2/index
```

重新创建目录之后，在进入配置文件把主从的slavestorePathRootDir和其他相关路径更改，之后重新启动，即可用jps命令检查到同时启动成功了。

### 5.集群监控平台搭建

在github上下载源码编译打包

### 

```sh
git clone https://github.com/apache/rocketmq-externals
cd rocketmq-console
mvn clean package -Dmaven.test.skip=true  打包之前要配置集群的地址

注意：打包前在rocketmq-console中配置namesrv集群地址：(两台机器的IP)
rocketmq.config.namesrvAddr=192.168.43.167:9876;172.22.58.136:9876
上传到机器
启动rocketmq-console：
java -jar rocketmq-console-ng-1.0.0.jar
```

客服端暂时访问不到，会出现关闭连接访问不到地址bug，使用了下面的命令，还没有解决，

nohup sh mqbroker -n 39.108.88.117:9876 -c conf/broker.conf autoCreateTopicEnable=true &

./mqadmin updateTopic -n 39.108.88.117:9876 -t stock -c DefaultCluster



经过几个小时的尝试之后，我果断放弃了硬钢，选者在本地再开一台虚拟机，重新配置集群，最后尝试在服务器端启动。

启动成功后，我们就可以通过浏览器访问`http://192.168.43.167:8080`进入控制台界面了，

![image-20201212024613141](http://yuge-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/image-20201212024613141.png)

### 6.消息发送样例

导入依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.4.0</version>
</dependency>
```

消息生产者步骤分析

1.创建消息生产者producer，并制定生产者组名
2.指定Nameserver地址
3.启动producer
4.创建消息对象，指定主题Topic、Tag和消息体
5.发送消息
6.关闭生产者producer

消息消费者步骤分析

1.创建消费者Consumer，制定消费者组名
2.指定Nameserver地址
3.订阅主题Topic和Tag
4.设置回调函数，处理消息
5.启动消费者consumer



#### 6.1基本样例

 消息发送

发送同步消息

```java
/*
* 发送同步消息
* */
public class SyncProducer {
    public static void main(String[] args) throws Exception {
//        1.创建消息生产者producer，并制定生产者组名
        DefaultMQProducer producer = new DefaultMQProducer("group1");
//        2.指定Nameserver地址
        producer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
//        3.启动producer
        producer.start();
        for (int i =0;i< 10 ;i++){
            //        4.创建消息对象，指定主题Topic、Tag和消息体
            /**
             * 参数一：消息主题：Topic
             * 参数二：消息Tag
             * 参数三： 消息内容
             * */
            Message msg = new Message("base", "Tag1", ("Hello World" + i).getBytes());
            //        5.发送消息
            SendResult result = producer.send(msg);
            //发送状态
            SendStatus status = result.getSendStatus();
            //消息ID
            String msgId = result.getMsgId();
            //消息接收队列ID
            int queueId = result.getMessageQueue().getQueueId();
            System.out.println("发送结果:"+result);
            //线程睡1秒
            TimeUnit.SECONDS.sleep(1);
        }

//        6.关闭生产者producer
        producer.shutdown();
    }
}
```



发送异步消息

```java
/**
 * 发送异步消息
 * */
public class AsyncProducer {
    public static void main(String[] args) throws Exception{
        //        1.创建消息生产者producer，并制定生产者组名
        DefaultMQProducer producer = new DefaultMQProducer("group1");
//        2.指定Nameserver地址
        producer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
//        3.启动producer
        producer.start();
        for (int i =0;i< 10 ;i++){
            //        4.创建消息对象，指定主题Topic、Tag和消息体
            /**
             * 参数一：消息主题：Topic
             * 参数二：消息Tag
             * 参数三： 消息内容
             * */
            Message msg = new Message("base", "Tag2", ("Hello World" + i).getBytes());
            //        5.发送消息
            producer.send(msg, new SendCallback() {
                /**
                 * 发送成功回调函数
                 * sendResult
                 * */
                public void onSuccess(SendResult sendResult) {
                    System.out.println("发送结果"+sendResult);
                }
                /**
                 * 发送失败回调函数
                 * */
                public void onException(Throwable e) {
                    System.out.println("发送失败"+e);
                }
            });
          
            //线程睡1秒
            TimeUnit.SECONDS.sleep(1);
        }

//        6.关闭生产者producer
        producer.shutdown();
    }
}
```



发送单项消息

```java
/**
 * 发送单向消息
 * */
public class OneWayProducer {
    public static void main(String[] args) throws Exception{
        //        1.创建消息生产者producer，并制定生产者组名
        DefaultMQProducer producer = new DefaultMQProducer("group1");
//        2.指定Nameserver地址
        producer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
//        3.启动producer
        producer.start();
        for (int i =0;i< 10 ;i++){
            //        4.创建消息对象，指定主题Topic、Tag和消息体
            /**
             * 参数一：消息主题：Topic
             * 参数二：消息Tag
             * 参数三： 消息内容
             * */
            Message msg = new Message("base", "Tag3", ("Hello World,单项消息" + i).getBytes());
            //        5.发送消息
            producer.sendOneway(msg);

            //线程睡1秒
            TimeUnit.SECONDS.sleep(1);
        }

//        6.关闭生产者producer
        producer.shutdown();
    }
}
```



**消息消费**

分为负载均衡模式和广播模式，默认为负载均衡模式。

负载均衡模式

```java
/**
 * 消息的接受者
 * */
public class Consumer {
    public static void main(String[] args) throws Exception {
        //1.创建消费者Consumer，制定消费者组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");

        //2.指定Nameserver地址
        consumer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
        //3.订阅主题Topic和Tag
        consumer.subscribe("base","Tag3");
        //4.设置回调函数，处理消息
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg:msgs){
                    System.out.println(new String(msg.getBody()));
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
       // 5.启动消费者consumer
        consumer.start();
    }
}
```

广播模式



```java
/**
 * 消息的接受者
 * */
public class Consumer {
    public static void main(String[] args) throws Exception {
        //1.创建消费者Consumer，制定消费者组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");

        //2.指定Nameserver地址
        consumer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
        //3.订阅主题Topic和Tag
        consumer.subscribe("base","Tag1");
        //广播模式消费
        consumer.setMessageModel(MessageModel.BROADCASTING);
        //4.设置回调函数，处理消息
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg:msgs){
                    System.out.println(new String(msg.getBody()));
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
       // 5.启动消费者consumer
        consumer.start();
    }
}
```



#### 6.2顺序消息

按照消息发送的顺序进行消费，可以分为分区有序或者全局有序。当发送和消费参与的queue只有一个，则是全局有序；如果多个queue参与，则为分区有序。

 **顺序消息生产**

先创建一个实体类

```java
/**
 * 订单构建者
 * */
public class OrderStep {
    private long orderId;
    private String desc;

 get,set,toString...
  
    /**
     * 生成模拟订单数据
     */
    public static List<OrderStep> buildOrders() {
        List<OrderStep> orderList = new ArrayList<OrderStep>();
        //1039L 创建 付款 推送 完成
        //1065L创建 付款
        //7235L 创建 付款
        OrderStep orderDemo = new OrderStep();
        orderDemo.setOrderId(1039L);
        orderDemo.setDesc("创建");
        orderList.add(orderDemo);

        orderDemo = new OrderStep();
        orderDemo.setOrderId(1065L);
        orderDemo.setDesc("创建");
        orderList.add(orderDemo);

        orderDemo = new OrderStep();
        orderDemo.setOrderId(7235L);
        orderDemo.setDesc("创建");
        orderList.add(orderDemo);

        orderDemo = new OrderStep();
        orderDemo.setOrderId(1039L);
        orderDemo.setDesc("付款");
        orderList.add(orderDemo);

        orderDemo = new OrderStep();
        orderDemo.setOrderId(1065L);
        orderDemo.setDesc("付款");
        orderList.add(orderDemo);

        orderDemo = new OrderStep();
        orderDemo.setOrderId(7235L);
        orderDemo.setDesc("付款");
        orderList.add(orderDemo);

        orderDemo = new OrderStep();
        orderDemo.setOrderId(1039L);
        orderDemo.setDesc("推送");
        orderList.add(orderDemo);

        orderDemo = new OrderStep();
        orderDemo.setOrderId(1039L);
        orderDemo.setDesc("完成");
        orderList.add(orderDemo);

        return orderList;
    }
}
```



顺序消息生产

```java
public class Producer {
    public static void main(String[] args) throws Exception {
        // 1.创建消息生产者producer，并制定生产者组名
        DefaultMQProducer producer = new DefaultMQProducer("group1");
//        2.指定Nameserver地址
        producer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
//        3.启动producer
        producer.start();
        //构建消息集合
        List<OrderStep> orderSteps = OrderStep.buildOrders();
        //发送消息
        for (int i= 0; i <orderSteps.size();i++){
            String body = orderSteps.get(i)+"";
            Message message = new Message("OrderTopic", "Order", "i" + i, body.getBytes());
            /*
             * 消息对象
             * 参数二：消息队列选择器
             * 参数三：选者队列的业务标识
             * */
            SendResult sendResult= producer.send(message, new MessageQueueSelector() {
                /**
                 * mqs: 队列集合
                 * msg：消息对象
                 * arg：业务参数
                 * */

                public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                    long orderId =(Long)  arg;
                    long index = orderId % mqs.size();
                    return mqs.get((int) index);
                }
            },orderSteps.get(i).getOrderId());
            System.out.println("发送结果"+sendResult);
        }
        producer.shutdown();
    }

}
```



顺序消费消息

```java
public class Consumer {
    public static void main(String[] args) throws Exception {
        //1.创建消费者Consumer，制定消费者组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");

        //2.指定Nameserver地址
        consumer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
        //3.订阅主题Topic和Tag
        consumer.subscribe("OrderTopic","*");
        //4.注册消息监听器
        consumer.registerMessageListener(new MessageListenerOrderly() {
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
                for (MessageExt msg: msgs){
                    System.out.println("线程名称:【"+Thread.currentThread().getName()+ "】:"+new String(msg.getBody()));
                }
                return ConsumeOrderlyStatus.SUCCESS;
            }
        });
        //5.启动消费者
        consumer.start();
        System.out.println("消费者启动");
    }
}
```



#### 6.4延时消息

**发送延迟消息**



```java
public class Producer {
    public static void main(String[] args) throws Exception {
        // 1.创建消息生产者producer，并制定生产者组名
        DefaultMQProducer producer = new DefaultMQProducer("group1");
//        2.指定Nameserver地址
        producer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
//        3.启动producer
        producer.start();

        for (int i=0; i< 10; i++){
            //4.创建消息对象，指定主题Topic，Tag，和消息体
            Message msg = new Message("DelayTopic", "Tag1", ("HelloWord" + i).getBytes());
            /*
             * 消息对象
             * 参数二：消息队列选择器
             * 参数三：选者队列的业务标识
             * */
            //设定延时时间
            msg.setDelayTimeLevel(2);
            //发送消息
            SendResult result= producer.send(msg);
            //发送状态
            SendStatus status = result.getSendStatus();
            System.out.println("发送结果"+result);
            //线程睡眠1秒
            TimeUnit.SECONDS.sleep(1);
        }
        //关闭生产者
        producer.shutdown();
    }
}    
```



**启动消息消费者**



```java
public class Consumer {
    public static void main(String[] args) throws Exception {
        //1.创建消费者Consumer，制定消费者组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");

        //2.指定Nameserver地址
        consumer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
        //3.订阅主题Topic和Tag
        consumer.subscribe("DelayTopic","*");

        //4.设置回调函数，处理消息
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg:msgs){
                    System.out.println("消息ID:【"+msg.getMsgId()+"】,延迟时间："+(System.currentTimeMillis()-msg.getStoreTimestamp()));
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 5.启动消费者consumer
        consumer.start();
        System.out.println("消费者启动");
    }
}
```



**延迟使用限制**

```java
messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h"; // 一共有18个等级
```

#### 6.5批量消息

**发送批量消息**

发送消息不能超过4MB的，如果大于4MB需要进行分割



```java
public class Producer {
    public static void main(String[] args) throws Exception {
        // 1.创建消息生产者producer，并制定生产者组名
        DefaultMQProducer producer = new DefaultMQProducer("group1");
//        2.指定Nameserver地址
        producer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
//        3.启动producer
        producer.start();

        List<Message> msgs = new ArrayList<Message>();
        //4.创建消息对象，指定主题Topic，Tag，和消息体

            /*
             * 消息对象
             * 参数二：消息队列选择器
             * 参数三：选者队列的业务标识
             * */
        Message msg1 = new Message("BatchTopic", "Tag1", ("HelloWord" + 1).getBytes());
        Message msg2 = new Message("BatchTopic", "Tag1", ("HelloWord" + 1).getBytes());
        Message msg3 = new Message("BatchTopic", "Tag1", ("HelloWord" + 1).getBytes());

        msgs.add(msg1);
        msgs.add(msg2);
        msgs.add(msg3);
        //发送消息
        SendResult result= producer.send(msgs);
        //发送状态
        SendStatus status = result.getSendStatus();
        System.out.println("发送结果"+result);
        //线程睡眠1秒
        TimeUnit.SECONDS.sleep(1);

        //关闭生产者
        producer.shutdown();
    }
}
```



#### 6.6过滤消息

限制是一个消息只能有一个标签，在复杂的情况下，可以使用SQL表达式筛选消息。

**SQL基本语法**

RocketMQ只定义了一些基本语法来支持这个特性。

* 数值比较，比如：**>，>=，<，<=，BETWEEN，=；**
* 字符比较，比如：**=，<>，IN；**
* **IS NULL** 或者 **IS NOT NULL；**
* 逻辑符号 **AND，OR，NOT；**

常量支持类型为：

* 数值，比如：**123，3.1415；**
* 字符，比如：**'abc'，必须用单引号包裹起来；**
* **NULL**，特殊的常量
* 布尔值，**TRUE** 或 **FALSE**

只有使用push模式的消费者才能用使用SQL92标准的sql语句，接口如下：

```java
public void subscribe(finalString topic, final MessageSelector messageSelector)
```

**消息生产者**

发送消息时，你能通过`putUserProperty`来设置消息的属性

```java
DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
producer.start();
Message msg = new Message("TopicTest",
   tag,
   ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET)
);
// 设置一些属性
msg.putUserProperty("a", String.valueOf(i));
SendResult sendResult = producer.send(msg);

producer.shutdown();
```

eg:

```java
public class Producer {
    public static void main(String[] args) throws Exception {
        // 1.创建消息生产者producer，并制定生产者组名
        DefaultMQProducer producer = new DefaultMQProducer("group1");
//        2.指定Nameserver地址
        producer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
//        3.启动producer
        producer.start();

        //4.创建消息对象，指定主题Topic，Tag，和消息体
        for (int i=0; i< 10; i++) {
            /*
             * 消息对象
             * 参数二：消息队列选择器
             * 参数三：选者队列的业务标识
             * */
            Message msg = new Message("FilterSQLTopic", "Tag1", ("HelloWord" + i).getBytes());

            msg.putUserProperty("i",String.valueOf(i) );
            //发送消息
            SendResult result = producer.send(msg);
            //发送状态
            SendStatus status = result.getSendStatus();
            System.out.println("发送结果" + result);
            //线程睡眠1秒
            TimeUnit.SECONDS.sleep(1);
        }
        //关闭生产者
        producer.shutdown();
    }
}
```



**消息消费者**

用MessageSelector.bySql来使用sql筛选消息

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");
// 只有订阅的消息有这个属性a, a >=0 and a <= 3
consumer.subscribe("TopicTest", MessageSelector.bySql("a between 0 and 3");
consumer.registerMessageListener(new MessageListenerConcurrently() {
   @Override
   public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
       return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
   }
});
consumer.start();
```



eg:

```java
public class Consumer {
    public static void main(String[] args) throws Exception {
        //1.创建消费者Consumer，制定消费者组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");
        //2.指定Nameserver地址
        consumer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
        //3.订阅主题Topic和Tag
        consumer.subscribe("FilterSQLTopic", MessageSelector.bySql("i>5"));

        //4.设置回调函数，处理消息
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg:msgs){
                    System.out.println("consumerThread="+Thread.currentThread().getName()+","+new String(msg.getBody()));
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 5.启动消费者consumer
        consumer.start();
        System.out.println("消费者启动");
    }
}    
```



**通过标签来限制**

消息生产者



```java
public class Producer {
    public static void main(String[] args) throws Exception {
        // 1.创建消息生产者producer，并制定生产者组名
        DefaultMQProducer producer = new DefaultMQProducer("group1");
//        2.指定Nameserver地址
        producer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
//        3.启动producer
        producer.start();

        //4.创建消息对象，指定主题Topic，Tag，和消息体
        for (int i=0;i< 3;i++) {
            /*
             * 消息对象
             * 参数二：消息队列选择器
             * 参数三：选者队列的业务标识
             * */
            Message msg = new Message("FilterTagTopic", "Tag2", ("HelloWord" + i).getBytes());

            //发送消息
            SendResult result = producer.send(msg);
            //发送状态
            SendStatus status = result.getSendStatus();
            System.out.println("发送结果" + result);
            //线程睡眠1秒
            TimeUnit.SECONDS.sleep(1);
        }
        //关闭生产者
        producer.shutdown();
    }
}
```

 消息消费者



```java
public class Consumer {
    public static void main(String[] args) throws Exception {
        //1.创建消费者Consumer，制定消费者组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");

        //2.指定Nameserver地址
        consumer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
        //3.订阅主题Topic和Tag
        consumer.subscribe("FilterTagTopic","Tag1 || Tag2");

        //4.设置回调函数，处理消息
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg:msgs){
                    System.out.println("consumerThread="+Thread.currentThread().getName()+","+new String(msg.getBody()));
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 5.启动消费者consumer
        consumer.start();
        System.out.println("消费者启动");
    }
}
```



#### 6.7 事务消息



**流程**

![image-20201214231722740](http://yuge-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/image-20201214231722740.png)

分为两个流程：正常事务消息的发送及提交、事务消息的补偿流程。

1.事务消息发送及提交：

(1) 发送消息（half消息）。

(2) 服务端响应消息写入结果。

(3) 根据发送结果执行本地事务（如果写入失败，此时half消息对业务不可见，本地逻辑不执行）。

(4) 根据本地事务状态执行Commit或者Rollback（Commit操作生成消息索引，消息对消费者可见）

2.事务补偿

(1) 对没有Commit/Rollback的事务消息，从服务端发起一次“回查”

(2) Producer收到回查消息，检查回查消息对应的本地事务的状态

(3) 根据本地事务状态，重新Commit或者Rollback

其中，补偿阶段用于解决消息Commit或者Rollback发生超时或者失败的情况。



##### 创建事务性生产者

```java
public class Producer {
    public static void main(String[] args) throws Exception {
        // 1.创建消息生产者producer，并制定生产者组名
        TransactionMQProducer producer = new TransactionMQProducer("group5");
//        2.指定Nameserver地址
        producer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");

        //添加事务的监听器
        producer.setTransactionListener(new TransactionListener() {
            /**
             * 在该方法中执行本地事务*/
            public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
                if (StringUtils.equals("TagA",msg.getTags())){
                    return LocalTransactionState.COMMIT_MESSAGE;
                }else if (StringUtils.equals("TagB",msg.getTags())){
                    return LocalTransactionState.ROLLBACK_MESSAGE;
                }else if (StringUtils.equals("TagC",msg.getTags())){
                    return LocalTransactionState.UNKNOW;
                }
                return LocalTransactionState.UNKNOW;
            }

            /**
             * 在该方法时MQ进行消息事务状态回查
             * */
            public LocalTransactionState checkLocalTransaction(MessageExt msg) {
                System.out.println("消息的Tag:"+msg.getTags());
                return LocalTransactionState.COMMIT_MESSAGE;
            }
        });
//        3.启动producer
        producer.start();

        String[ ] tags= {"TagA","TagB","TagC"};
        //4.创建消息对象，指定主题Topic，Tag，和消息体
        for (int i=0;i< 3;i++) {
            /*
             * 消息对象
             * 参数二：消息队列选择器
             * 参数三：选者队列的业务标识
             * */
            Message msg = new Message("TransactionTopic", tags[i], ("HelloWord" + i).getBytes());

            //发送消息
            SendResult result = producer.sendMessageInTransaction(msg,null);
            //发送状态
            SendStatus status = result.getSendStatus();
            System.out.println("发送结果" + result);
            //线程睡眠1秒
            TimeUnit.SECONDS.sleep(1);
        }
        //关闭生产者
//        producer.shutdown();
    }
}
```



##### 实现事务的消费者



```java
public class Consumer {
    public static void main(String[] args) throws Exception {
        //1.创建消费者Consumer，制定消费者组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");

        //2.指定Nameserver地址
        consumer.setNamesrvAddr("192.168.43.167:9876;192.168.43.168:9876");
        //3.订阅主题Topic和Tag
        consumer.subscribe("TransactionTopic","*");

        //4.设置回调函数，处理消息
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg:msgs){
                    System.out.println("consumerThread="+Thread.currentThread().getName()+","+new String(msg.getBody()));
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 5.启动消费者consumer
        consumer.start();
        System.out.println("消费者启动");
    }
}
```



**使用限制**



1. 事务消息不支持延时消息和批量消息。
2. 为了避免单个消息被检查太多次而导致半队列消息累积，我们默认将单个消息的检查次数限制为 15 次，但是用户可以通过 Broker 配置文件的 `transactionCheckMax`参数来修改此限制。如果已经检查某条消息超过 N 次的话（ N = `transactionCheckMax` ） 则 Broker 将丢弃此消息，并在默认情况下同时打印错误日志。用户可以通过重写 `AbstractTransactionCheckListener` 类来修改这个行为。
3. 事务消息将在 Broker 配置文件中的参数 transactionMsgTimeout 这样的特定时间长度之后被检查。当发送事务消息时，用户还可以通过设置用户属性 CHECK_IMMUNITY_TIME_IN_SECONDS 来改变这个限制，该参数优先于 `transactionMsgTimeout` 参数。
4. 事务性消息可能不止一次被检查或消费。
5. 提交给用户的目标主题消息可能会失败，目前这依日志的记录而定。它的高可用性通过 RocketMQ 本身的高可用性机制来保证，如果希望确保事务消息不丢失、并且事务完整性得到保证，建议使用同步的双重写入机制。
6. 事务消息的生产者 ID 不能与其他类型消息的生产者 ID 共享。与其他类型的消息不同，事务消息允许反向查询、MQ服务器能通过它们的生产者 ID 查询到消费者。