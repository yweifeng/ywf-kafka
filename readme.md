# kafka集群

## 1 集群原理

TODO...

## 2 集群部署

### 2.1 服务器环境

centos7 、zookeeper-3.4.12、kafka_2.11-2.0.0

| 主机名 | ip地址          | kafka broke.id | zookeeper | myid |
| ------ | --------------- | -------------- | --------- | ---- |
| kafka1 | 192.168.111.128 | 1              | server.1  | 1    |
| kafka2 | 192.168.111.129 | 2              | server.2  | 2    |
| kafka3 | 192.168.111.130 | 3              | server.3  | 3    |

### 2.2 搭建zookeeper集群

#### 2.2.1 下载zookeeper安装包 [zookeeper-3.4.12.tar](https://links.jianshu.com/go?to=https%3A%2F%2Fpan.baidu.com%2Fs%2F1uvbXJvjqOetpUB8Y7aT05Q) 

#### 2.2.2 解压并设置环境变量

```shell
# 解压
tar -zxvf zookeeper-3.4.12.tar

# 重命名
mv zookeeper-3.4.12 zookeeper

# 修改环境变量
vim etc/profile
#set zookeeper environment
export ZK_HOME=/opt/zookeeper
export PATH=$ZK_HOME/bin:$PATH

# 重启环境变量
source /etc/profile
```

#### 2.2.3 修改配置文件

```shell
# 拷贝zoo_sample.cfg 并重命名为zoo.cfg
cd opt/zookeeper/conf
cp zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg

# 修改配置文件
#修改数据文件夹路径
dataDir=/opt/zookeeper/data
#在文件末尾添加
server.1=192.168.111.128:2888:3888
server.2=192.168.111.129:2888:3888
server.3=192.168.111.130:2888:3888
```

#### 2.2.4 创建数据文件夹

```shell
cd opt/zookeeper
mkdir data
```

#### 2.2.5 创建myid文件

```shell
# 192.168.111.128
echo 1 >> /opt/zookeeper/data/myid
# 192.168.111.129
echo 2 >> /opt/zookeeper/data/myid
# 192.168.111.130
echo 3 >> /opt/zookeeper/data/myid
```

#### 2.2.6 启动zookeeper

```shell
# 每个机子单独启动zookeeper
[root@hadoop-slave1 zookeeper]# cd bin/
[root@hadoop-slave1 bin]# ./zkServer.sh start

# 查看启动结果
[root@hadoop-slave1 bin]# zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Error contacting service. It is probably not running.

# cat zookeeper.out分析原因
2019-11-25 10:41:24,690 [myid:1] - WARN  [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:QuorumCnxManager@584] - Cannot open channel to 2 at election address /192.168.111.129:3888
java.net.ConnectException: 拒绝连接 (Connection refused)

#解决方法 
1、scp 拷贝 zookeeper到3台服务器。
2、修改myid。
3、scp 拷贝/etc/profile 到3太服务器。
4、source /etc/profile
5、三台服务器./zkServer.sh start启动  ./zkServer.sh status

#192.168.111.128
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: follower

#192.168.111.129
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: follower

#192.168.111.130
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: leader
```



### 2.3 kafka集群搭建

#### 2.3.1 下载 kafka安装包   [kafka_2.11-2.0.0 .tgz](https://links.jianshu.com/go?to=https%3A%2F%2Fpan.baidu.com%2Fs%2F1Flc6qthv6p1Dqq7mEISyIA)



#### 2.3.2 解压并设置环境变量

```shell
# 解压
tar -zxvf kafka_2.11-2.0.0 .tgz

# 重命名
mv kafka_2.11-2.0.0 kafka

# 设置环境变量
vim /etc/profile
export KAFKA_HOME=/opt/kafka
PATH=${KAFKA_HOME}/bin:$PATH

# 重启环境变量
source /etc/profile

# 修改hosts
source /etc/hosts
192.168.111.128 kafka1
192.168.111.129 kafka2
192.168.111.130 kafka3
```

#### 2.3.3 修改配置文件 server.properties 

```shell
vim /opt/kafka/config/server.properties
# 修改broker.id
broker.id=1
# 修改 listeners 
listeners=PLAINTEXT://kafka1:9092

# 修改 zookeeper连接地址
zookeeper.connect=192.168.111.128:2181,192.168.111.129:2181,192.168.111.130:2181
```

#### 2.3.4 拷贝到其他服务器

```shell
# 拷贝kafka到其他服务器(192.168.111.129、192.168.111.130)
scp -r kafka root@192.168.111.129:/opt
# 拷贝/etc/hosts到其他服务器
scp hosts root@192.168.111.129:/etc
# 拷贝/etc/profile到其他服务器
scp /etc/profile root@192.168.111.129:/etc
# 重启环境变量
source /etc/profile
# 修改server.properties broker.id
192.168.111.129 broker.id=2
192.168.111.130 broker.id=3
```

#### 2.3.4 启动kafka

```shell
# 各服务器单独启动kafka
bin/kafka-server-start.sh -daemon config/server.properties 
```

#### 2.3.5 创建并查看topic

```shell
# 选择一台服务器创建topic 192.168.111.128  topic命名不要用_
kafka-topics.sh --create --zookeeper 192.168.111.128:2181 --replication-factor 3 --partitions 1 --topic topic-ywf

# 查看topic
 bin/kafka-topics.sh --describe --zookeeper 192.168.111.128:2181 --topic topic-ywf
	
# 修改partition数目
bin/kafka-topics.sh --zookeeper 192.168.111.128:2181 -alter --partitions 3 --topic topic-ywf

# 查看topic 生产消息情况
bin/kafka-console-consumer.sh --bootstrap-server 192.168.111.128:9092 --topic topic-ywf --from-beginning

# 查看topic 消费情况
bin/kafka-consumer-groups.sh --group test-consumer-group --describe --bootstrap-server 192.168.111.128:9092
```



### 2.4 kafka监控搭建

#### 2.4.1 安装[kafka Tool](http://www.kafkatool.com/download2/kafkatool_64bit.exe)



## 3 Springboot集成kafka

### 3.1 配置host

```
windows: C:\Windows\System32\drivers\etc\hosts
```

### 3.2 建立项目，修改pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.ywf</groupId>
    <artifactId>kafka</artifactId>
    <version>1.0-SNAPSHOT</version>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.2.RELEASE</version>
    </parent>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>

        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>2.8.2</version>
        </dependency>

    </dependencies>
</project>
```

### 3.3 建立传输实体类

```java
package com.ywf;

import lombok.Data;

import java.util.Date;

@Data
public class Message {
    private Long id;    //id

    private String msg; //消息

    private Date sendTime;  //时间戳

}

```

### 3.4 创建KafkaSender

```java
package com.ywf;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import java.util.Date;
import java.util.UUID;

@Component
@Slf4j
public class KafkaSender {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    private Gson gson = new GsonBuilder().create();

    //发送消息方法
    public void send() {
        Message message = new Message();
        Long id = System.currentTimeMillis();
        message.setId(id);
        message.setMsg(UUID.randomUUID().toString());
        message.setSendTime(new Date());
        log.info("+++++++++++++++++++++  message = {}", gson.toJson(message));
        kafkaTemplate.send("topic-ywf", String.valueOf(id), gson.toJson(message));
    }
}
```

### 3.5 创建KafkaReceiver

```java
package com.ywf;

import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

import java.util.Optional;

@Component
@Slf4j
public class KafkaReceiver {
    @KafkaListener(topics = {"topic-ywf"})
    public void listen(ConsumerRecord<?, ?> record) {

        Optional<?> kafkaMessage = Optional.ofNullable(record.value());

        if (kafkaMessage.isPresent()) {

            Object message = kafkaMessage.get();

            log.info("----------------- record =" + record);
            log.info("------------------ message =" + message);
        }

    }
}
```

### 3.6 创建KafkaApplication，并启动

```java
package com.ywf;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

@SpringBootApplication
public class KafkaApplication {
    public static void main(String[] args) {

        ConfigurableApplicationContext context = SpringApplication.run(KafkaApplication.class, args);

        KafkaSender sender = context.getBean(KafkaSender.class);

        for (int i = 0; i < 3; i++) {
            //调用消息发送类中的消息发送方法
            sender.send();

            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```



### 3.7 启动kafka Tool 查看

