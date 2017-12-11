# Docker Issue Network Delay

在用自定义Docker网络跑容器的时候发现一个问题：`Docker的自定义网络启动会延迟大概40秒！`

> 换句话说就是：
>
> * 如果你使用自定义网络在一个容器启动时想访问另外一个容器会失败！但是如果你先等待40秒再访问的话就一切正常！
> * 如果你使用自定义网络在一个容器启动时`ping`另外一个容器会卡住一段时间。

解决：加上启动脚本检测网络是否就绪！

可以用类似下面的脚本检测服务是否就绪，或者干脆检查dmesg消息也可以

```bash
until nc -z zk 2181; do echo "waiting for zk to be ready"; sleep 0.5; done
```


## 现象

在用自定义Docker网络跑`Kafka`的时候发现一个现象：`zk`服务正常，但是 `Kafka`始终报告连接不上`zk`

``````bash
$ docker run --net=br --ip=192.168.33.88 --name=zk -h=zk -d wurstmeister/zookeeper
$ docker run --net=br --ip=192.168.33.91 --name=kf1 -h=kf1 \
 -e KAFKA_ZOOKEEPER_CONNECT=zk \
 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kf1:9092 \
 -e KAFKA_BROKER_ID=1 \
 --link zk:zk \
 -it wurstmeister/kafka
``````

运行后始终报错：

``````
java.net.NoRouteToHostException: No route to host
	at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
	at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:717)
	at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:361)
	at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1141)
``````

而如果你不用自定义网络的话则一切正常！

``````bash
$ docker run --name=zk -h=zk -d wurstmeister/zookeeper
$ docker run --name=kf1 -h=kf1 \
 -e KAFKA_ZOOKEEPER_CONNECT=zk \
 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kf1:9092 \
 -e KAFKA_BROKER_ID=1 \
 --link zk:zk \
 -it wurstmeister/kafka
``````



## 分析

那么问题在哪里呢？经过跟踪后终于发现问题在于`Docker的自定义网络启动会延迟大概40秒`！需要等容器实例`dmesg`中出现下列消息的时才能正常访问网络中的其他容器实例，这个等待时间大概是40秒

```
[ 1077.847733] docker1: topology change detected, propagating
```



## 解决

问题找到了，搜索了一圈也没有找到怎么让这个延迟时间消失的解决方法，好吧，用土办法：既然是因为网络还没准备好，那就等它准备好！

可以用类似下面的脚本检测服务是否就绪，或者干脆检查dmesg消息也可以

```bash
until nc -z zk 2181; do echo "waiting for zk to be ready"; sleep 0.5; done
```

那么只要在Docker容器真正开始运行之前先运行上面的脚本检测网络是否就绪就可以了，查了一下`wurstmeister/zookeeper`正好有一个`CUSTOM_INIT_SCRIPT`参数可以干这个事，妥了！

``````bash
$ docker run --net=br --ip=192.168.33.88 --name=zk -h=zk -d wurstmeister/zookeeper
$ docker run --net=br --ip=192.168.33.91 --name=kf1 -h=kf1 \
 -e KAFKA_ZOOKEEPER_CONNECT=zk \
 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kf1:9092 \
 -e KAFKA_BROKER_ID=1 \
 -e CUSTOM_INIT_SCRIPT="until nc -z zk 2181; do echo 'waiting for zk to be ready'; sleep 1; done" \
 --link zk:zk \
 -it wurstmeister/kafka
``````

``````
waiting for kafka to be ready
waiting for zk service ready
......
[2017-12-11 09:04:51,046] INFO [Partition user_events-0 broker=1] No checkpointed highwatermark is found for partition user_events-0 (kafka.cluster.Partition)
[2017-12-11 09:04:51,047] INFO Replica loaded for partition user_events-0 with initial high watermark 0 (kafka.cluster.Replica)
[2017-12-11 09:04:51,050] INFO [Partition user_events-0 broker=1] user_events-0 starts at Leader Epoch 0 from offset 0. Previous Leader Epoch was: -1 (kafka.cluster.Partition)
[2017-12-11 09:04:51,087] INFO [ReplicaFetcherManager on broker 1] Removed fetcher for partitions user_events-0 (kafka.server.ReplicaFetcherManager)
[2017-12-11 09:04:51,087] INFO [Partition user_events-0 broker=1] user_events-0 starts at Leader Epoch 1 from offset 0. Previous Leader Epoch was: 0 (kafka.cluster.Partition)
``````

另外：

* 这个方法也适用于启动时需要依赖其他服务就绪的情况，比如等待数据库就绪等
* 或者某些容器服务初始化时间较长，另外的容器需要等它就绪等