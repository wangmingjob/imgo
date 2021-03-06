imgo
==============
`imroc/imgo` 是一个支持集群的实时推送服务,基于[goim](https://github.com/Terry-Mao/goim),相比之下新增离线消息系统,以后会增加IM功能的支持.目前还不要用于生产环境,请耐心等待一段时间.

---------------------------------------
  * [特性](#特性)
  * [架构](#架构)
  * [安装](#安装)
  * [配置](#配置)
  * [例子](#例子)
  * [文档](#文档)
  * [更多](#更多)

---------------------------------------

## 特性
 * 轻量级、高性能
 * 支持单个、多个、单房间以及广播消息推送
 * 支持单个Key多个订阅者（可限制订阅者最大人数）
 * 支持安全验证（未授权用户不能订阅）
 * 多协议支持（websocket，tcp）
 * 支持离线消息(用户不在线也可推送，可等下次上线拉取离线消息)
 * 可拓扑的架构（comet、logic、router、job模块可动态无限扩展）
 * 基于Kafka做异步消息推送

##架构
TODO

### comet

comet 属于接入层，和客户端保持长连接，非常容易扩展，直接开启多个comet节点，修改配置文件中的base节点下的server.id修改成不同值（注意一定要保证不同的comet进程值唯一），前端接入可以使用LVS 或者 DNS来转发

### logic

logic 属于无状态的逻辑层，可以随意增加节点，使用nginx upstream来扩展http接口，内部rpc部分，可以使用LVS四层转发

### kafka

kafka 可以使用多broker，或者多partition来扩展队列

### router

router 属于有状态节点，logic可以使用一致性hash配置节点，增加多个router节点（目前还不支持动态扩容），提前预估好在线和压力情况

### job

job 根据kafka的partition来扩展多job工作方式，具体可以参考下kafka的partition负载

### store

store 用于离线消息持久化与拉取,暂时不支持集群,后续版本会增加集群支持.


## 安装

### Kafka(消息队列)

kafka自带zookeeper，它们都依赖java，所以如果没有请先安装java环境，建议1.7以上。
kafka的安装在官网已经描述的非常详细，在这里就不过多说明，安装、启动请查看[这里](http://kafka.apache.org/documentation.html#quickstart).


### Redis(离线消息系统store模块需要用到的)
```sh
$ wget http://download.redis.io/releases/redis-stable.tar.gz
$ tar -xvf redis-stable.tar.gz
$ cd redis-stable/src
$ make
$ make test
$ make install
$ cp ../redis.conf /etc/
$ cp redis-server /usr/local/bin/
$ redis-server /etc/redis.conf
```
* if following error, see FAQ 2
```sh
which: no tclsh8.5 in (/usr/lib64/qt-3.3/bin:/usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/sbin:/sbin:/home/geffzhang/bin)
You need 'tclsh8.5' in order to run the Redis test
Make[1]: *** [test] error 1
make[1]: Leaving directory ‘/data/program files/redis-2.6.4/src’
Make: *** [test] error 2！
```

### Golang(编译imgo各个模块)
1.下载
根据自己的系统下载对应的[安装包](http://golang.org/dl/)并解压。
比如：
```sh
$ wget http://www.golangtc.com/static/go/1.7/go1.7.linux-amd64.tar.gz
$ tar -xvf ggo1.7.linux-amd64.tar.gz -C /usr/local
```
2.配置GO环境变量
(这里我加在/etc/profile.d/golang.sh)
```sh
$ vi /etc/profile.d/golang.sh
# 将以下环境变量添加到profile最后面
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
export GOPATH=/data/apps/go
$ source /etc/profile
```

### 部署imgo
1.下载imgo及依赖包
```sh
$ yum install hg
$ go get -u github.com/imroc/imgo
$ mv $GOPATH/src/github.com/imroc/imgo $GOPATH/src/imgo
$ cd $GOPATH/src/imgo
$ go get ./...
```

2.安装store、router、logic、comet、job模块(配置文件请依据实际机器环境配置)
```sh
$ cd $GOPATH/src/imgo/router
$ go install
$ cp router.conf $GOPATH/bin/router.conf
$ cp router-log.xml $GOPATH/bin/
$ cd ../store/
$ go install
$ cp store.conf $GOPATH/bin/store.conf
$ cp store-log.xml $GOPATH/bin/
$ cd ../logic/
$ go install
$ cp logic-example.conf $GOPATH/bin/logic.conf
$ cp logic-log.xml $GOPATH/bin/
$ cd ../comet/
$ go install
$ cp comet-example.conf $GOPATH/bin/comet.conf
$ cp comet-log.xml $GOPATH/bin/
$ cd ../logic/job/
$ go install
$ cp job-example.conf $GOPATH/bin/job.conf
$ cp job-log.xml $GOPATH/bin/
```
到此所有的环境都搭建完成！

### 启动imgo
```sh
$ cd /$GOPATH/bin
$ nohup $GOPATH/bin/message -c $GOPATH/bin/store.conf 2>&1 > /data/logs/imgo/panic-store.log &
$ nohup $GOPATH/bin/router -c $GOPATH/bin/router.conf 2>&1 > /data/logs/imgo/panic-router.log &
$ nohup $GOPATH/bin/logic -c $GOPATH/bin/logic.conf 2>&1 > /data/logs/imgo/panic-logic.log &
$ nohup $GOPATH/bin/comet -c $GOPATH/bin/comet.conf 2>&1 > /data/logs/imgo/panic-comet.log &
$ nohup $GOPATH/bin/job -c $GOPATH/bin/job.conf 2>&1 > /data/logs/imgo/panic-job.log &
```
如果启动失败，默认配置可通过查看panic-xxx.log日志文件来排查各个模块问题.

### 测试

推送协议可查看[push http协议文档](https://github.com/imroc/imgo/blob/master/doc/push.md)

## 配置

TODO

## 例子

Websocket: [Websocket Client Demo](https://github.com/imroc/imgo/tree/master/examples/javascript)

Android: [Android](https://github.com/imroc/imgo-java-sdk)

iOS: [iOS](https://github.com/roamdy/goim-oc-sdk)

## 文档
[push http协议文档](https://github.com/imroc/imgo/blob/master/doc/push.md)推送接口

[添加token文档](https://github.com/imroc/imgo/blob/master/doc/token.md)添加token接口


##更多
TODO
