
- [1.环境依赖](#1.环境依赖)
- [2.服务器分配](#2.服务器分配)
- [3.安装搭建](#3.安装搭建)
    - [3.1下载安装](#3.1下载安装)
    - [3.2配置文件](#3.2配置文件)
        - [zookeeper.properties](#zookeeper.properties)
        - [server.properties](#server.properties)
        - [producer.properties](#producer.properties)
        - [consumer.properties](#consumer.properties)
    - [3.3相关配置项解](#3.3相关配置项解)
- [4.常用命令集锦](#4.常用命令集锦)


# 1.环境依赖 
> Kafka的安装需要java环境
> 以前的kafka还需要zookeeper，新版的kafka已经内置了一个zookeeper环境，所以可以直接使用。
> 防火墙端口关闭，或者允许开放端口，具体端口详见配置文件中出现的所有端口。


# 2.服务器分配 #
|    IP      |   Nodes  |
|:----------:|:--------:|
|192.168.5.12|  node12  |
|192.168.5.13|  node13  |
|192.168.5.14|  node14  |


# 3.安装搭建 #
## 3.1下载安装 ##
切换到三台服务器的/usr/local目录下：
> wget http://www-us.apache.org/dist/kafka/2.0.0/kafka_2.12-2.0.0.tgz    
> tar -xzf kafka_2.11-2.0.0.tgz    
> mv kafka_2.11-2.0.0.tgz kafka

至此，在三个node上面下载完kafka，目录都为`usr/local/kafka`。    
## 3.2配置文件 ##
　　安装完成后需要配置`kafka/config`文件下面的配置文件，以下以node12节点为参考，其他节点修改即可。里面的属性不是绝对的，后期根据性能，需求等进行设置。
###zookeeper.properties
　　三个节点的这个配置文件相同，无需改动。设置完后，需要在`dataDir`文件夹下标志自己的id，即新建`myid`文件，写入**0**，以方便zookeeper集群能够分别彼此。同理在**node13**节点下`myid`为**1**，**node14**为**2**。
```js
dataDir=/usr/local/kafka/zookeeper #需要自行创建文件夹
dataLogDir=/usr/local/kafka/log/zookeeper #存储消息

clientPort=2181

maxClientCnxns=100

tickTime=2000
initLimit=10
syncLimit=5

server.0=192.168.5.12:2888:3888 #这两个端口号一个是不同broker相互通信的，另一个是为了failover后进行选举leader用的。
server.1=192.168.5.13:2888:3888
server.2=192.168.5.14:2888:3888
```
###server.properties

```js
broker.id=0
log.dirs=/usr/local/kafka/kafka-logs #自行创建文件夹
zookeeper.connect=192.168.5.12:2181,192.168.5.13:2181,192.168.5.14:2181
zookeeper.connection.timeout.ms=6000

port=9092

listeners=PLAINTEXT://192.168.5.12:9092　　＃自身IP:port,不同node需要更改成
advertised.listeners=PLAINTEXT://192.168.5.12:9092 #自己所在服务器的IP:port

#background.threads=10
#compression.type=producer
#delete.topic.enable=true
#leader.imbalance.check.interval.seconds=300
#leader.imbalance.per.broker.percentage=10

#num.network.threads=3
#num.io.threads=8

socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

num.partitions=1
num.recovery.threads.per.data.dir=1

offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1

log.flush.interval.messages=10000
log.flush.interval.ms=1000
log.retention.check.interval.ms=300000

log.cleanup.policy=delete #到期清理规则为delete
log.retention.hours=5  # 测试的5小时内清理，可根据实际需求设置
#log.segment.bytes=512
#log.retention.bytes=512

auto.create.topics.enable=false
delete.topic.enable=true

group.initial.rebalance.delay.ms=0
```
###producer.properties
三个节点相同，无需改动。
```js
bootstrap.servers=192.168.5.12:9092,192.168.5.13:9092,192.168.5.14:9092 
compression.type=none #不压缩
producer.type=sync  #同步
acks=-1 
#设置为同步,并且设置为-1，producer会在所有备份的partition
#收到消息时得到broker的确认，这个设置可以得到最高的可靠性保证。
```
###consumer.properties
三节点相同，无需改动。
```
bootstrap.servers=192.168.5.12:9092,192.168.5.13:9092,192.168.5.14:9092 
group.id=kaka #实际编码中的group.id可以根据用途随意设置。
zookeeper.connect=192.168.5.12:2181,192.168.5.13:2181,192.168.5.14:2181 
```
## 3.3相关配置项解释 


- auto.offset.reset值含义解释
    - earliest
　　当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费
    - latest
　　当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据
　　经测试，earliest与latest都是从已经提交的offset开始消费，提交了offset以后没有继续生产消息就等待，如果提交的offset后有消息的话则从提交的offset后面开始消费。不同的group_id则消费情况不同，再换一个group_id的话，则会从头开始消费，但是必须设置为earliest，否则还是会等待消费新产生的消息。
    - none
　　topic各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常
- auto.create.topics.enable
　　server.properties  auto.create.topics.enable = false，默认设置为true。如果设置为true，则produce或者fetch 不存在的topic也会自动创建这个topic。这样会给删除topic带来很多意向不到的问题。
- delete.topic.enable=true
　　server.properties 设置 delete.topic.enable=true, 如果没有设置 delete.topic.enable=true，则调用kafka 的delete命令无法真正将topic删除，而是显示（marked for deletion）

- 数据保存时间设置
　　log.cleanup.policy = delete启用删除策略(默认),也可设置为compact。
    - 直接删除，删除后的消息不可恢复。可配置以下两个策略：
        - 清理超过指定时间清理：
      log.retention.hours=16
        - 超过指定大小后，删除旧的消息：
      log.retention.bytes=1073741824

    - 压缩
      将数据压缩，只保留每个key最后一个版本的数据。
      首先在broker的配置中设置log.cleaner.enable=true启用cleaner，这个默认是true。
      在topic的配置中设置log.cleanup.policy=compact启用压缩策略。
- 生产者的同步异步生产消息
producer.type=sync
request.required.acks=-1
　　设置为同步,并且设置为-1，producer会在所有备份的partition收到消息时得到broker的确认，这个设置可以得到最高的可靠性保证。

# 4.常用命令集锦 #
```
#zookeeper启动
bin/zookeeper-server-start.sh config/zookeeper.properties
#zookeeper关闭
bin/zookeeper-server-stop.sh config/zookeeper.properties
#kafka的server（也可以叫broker）启动，关闭同zookeeper类似
bin/kafka-server-start.sh config/server.properties
#创建topic=“test” 在node14节点上，两个备份，三个分区。
#（注意：zookeeper是集群，在一个节点创建且三分区，其他节点自动同步）
bin/kafka-topics.sh --create --zookeeper 192.168.5.14:2181 --replication-factor 2 --partitions 3 --topic test
#生产者生产数据 （注意，不能使用localhost:9092，需要指定IP）
bin/kafka-console-producer.sh --broker-list 192.168.5.14:9092 --topic test
#消费者从头消费消息
bin/kafka-console-consumer.sh --bootstrap-server 192.168.5.14:9092 --topic test --from-beginning (不带--from-beginning就是从最后一个消息消费)

#展示所有topic列表：
bin/kafka-topics.sh --list --zookeeper 192.168.5.14:2181

#展示test topic信息：
bin/kafka-topics.sh --describe --zookeeper 192.168.5.14:2181 --topic test

#描述test的详细信息
[root@localhost kafka]# bin/kafka-topics.sh --describe --zookeeper 192.168.5.14:2181 --topic test
Topic:test	PartitionCount:3	ReplicationFactor:2	Configs:
	Topic: test	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: test	Partition: 1	Leader: 1	Replicas: 1,2	Isr: 2,1
	Topic: test	Partition: 2	Leader: 2	Replicas: 2,0	Isr: 2,0
解释：
Here is an explanation of output. The first line gives a summary of all the partitions, each additional line gives information about one partition.
Since we have there partition for this topic and two replication.
"leader" is the node responsible for all reads and writes for the given partition. Each node will be the leader for a randomly selected portion of the partitions.
"replicas" is the list of nodes that replicate the log for this partition regardless of whether they are the leader or even if they are currently alive.
"isr" is the set of "in-sync" replicas. This is the subset of the replicas list that is currently alive and caught-up to the leader.

#删除topic=topic1
bin/kafka-topics.sh  --delete --zookeeper 192.168.5.14:2181 --topic topic1
```
　　删除命令使用后，虽然将delete.topic.enable=true，但是仍然会出现：
![](https://i.imgur.com/vSXJVZU.png)
　　但是查看kafka-logs这个目录下的topics记录已经没有了topic1这个主题。
![](https://i.imgur.com/8L3iZ8j.png)

　　至此kafka三节点集群搭建完毕。
