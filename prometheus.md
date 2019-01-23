## 1 Prometheus的介绍和使用
#### 1.1 介绍
prometheus一种应用于监控云服务的监控工具，类似于influxdb，使用的也是时间序列数据库（自带openTSDB），但是跟influxdb的区别也是有的：
- prometheus的应用场景多应用于监控云服务，跟docker的配合十分友好；
- prometheus的数据结构跟influxdb还是有区别的，sql不如influxdb强大；
- prometheus是一个pull的方式，而influxdb是push的方式，prometheus需要client处起一个http服务，提供metrics（默认）路由方法，供prometheus去抓取数据

#### 1.2 USAGE
```prometheus --config.file=prometheus.yml``` 启动prometheus ，默认启动的端口是9090，然后可以打开http://127.0.0.1:9090/去查看prometheus的web界面
- prometheus 的console 窗口可以输入很多[expression](https://prometheus.io/docs/prometheus/latest/querying/basics/) 语句来进行感兴趣的参数查询
- premetheus 也可以用grafana来配置，类似于influxdb跟grafana的配置

#### 1.3 配置文件的构成
简版的配置文件 ```prometheus.yml```，目前只用到了两部分配置：
- global：prometheus的总体配置，比如取数频率之类的
- scrape_configs: prometheus抓取的client参数配置
```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.多久抓取一次客户端的数据
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.多久重新加载一次报警的规则
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:   
      monitor: 'codelab-monitor' # 目前没有跟其他系统交互，先不用理会

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
# 这个是prometheus数据报警机制的规则，目前没用到，还没有研究
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # 这个就是要监控哪些节点的设置，目前需要的功能
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'    # prometheus自监控，监控自身的状态

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:         
      - targets: ['localhost:9090']

  - job_name: 'docker' # prometheus监控节点的状态，例子里是监控docker的节点
         # metrics_path defaults to '/metrics'
         # scheme defaults to 'http'.

    static_configs:  # 要在docker方面打开暴露给prometheus的接口，具体例子见https://github.com/Mabo-IoT/prometheus-example
      - targets: ['localhost:9323']
```
#### 1.4 告警配置(待续，先跳过，目前用不着)
通过命令行flag和配置文件config进行配置
- 命令行flag来配置不可变项，
- config file 定义了推送规则，推送路由和推送接收方
[visual editor](https://prometheus.io/webtools/alerting/routing-tree-editor/)可以来编写route tree

## 2 prometheus数据格式和查询
#### 2.1 数据格式
prometheus的数据格式（参考[openTSDB数据格式]([http://opentsdb.net/docs/build/html/user_guide/writing/index.html#data-specification)）相比influxdb来说比较简单，由四部分组成：
```
http_requests_total 1554621254    360        method=GET
<metric>        <timestamp> <value> <tagk1=tagv1[ tagk2=tagv2 ...tagkN=tagvN]>
```
- metric 一个时间序列集合的名字，如```http_requests_total```
- timestamp 每个point的时间戳
- value point的值
- tags 时间序列里的标签，如```http_requests_total{"method"="GET"}```，其中method就是标签名，get是标签值
总体对比influxdb的数据结构应该很好理解。

#### 2.2 查询
prometheus不同于influxdb的类sql查询方式，prometheus只有四种查询类型(参考[官方文档](https://prometheus.io/docs/prometheus/latest/querying/basics/))：
- **Instant vector** 即时查询，可以看做是一个timestamp下的不同timeseries的数据结合，这个数据集合下的point全部都是同一个时间点下的。比如
查询```http_requests_total```的结果，就是这个metric下各个tag的数据集合。
- **Range vector** 时间序列查询，可以看做是查询一条时间序列下的数据，类似与influxdb加```where time > now() - 5m```这样的时间序列下的数据集合。比如```http_requests_total[5m]```就是查询最近5分钟之内的http_requests_total时间序列所有points
- **Scalar** 查询一个浮点数值
- **String** 查询一个string值（目前prometheus整个官方应用还没有使用这个特性）
查询语句也能加一些函数功能，但是用的最多的是还是**instant vector**以及**range vector**

## 3 prometheus节点配置
prometheus的节点应该满足http server的一个默认```/metircs```路由
(待续)[https://prometheus.io/docs/concepts/metric_types/]


## 4 docker作为prometheus client节点配置
#### 4.1 collect docker metrcs with prometheus
- 1 configure docker，更改docker的配置文件
```
vim /etc/docker/daemon.json
```
```
{
  "metrics-addr" : "127.0.0.1:9323",
  "experimental" : true
}
```

- 3 启动docker和prometheus 
```
systemctl restart docker
```
```
prometheus --config.file=prometheus.yml
```


## 5 prometheus监控硬件信息
利用[node_exporter](https://github.com/prometheus/node_exporter)来监控硬件信息，具体策略可参考[官网教程](https://prometheus.io/docs/guides/node-exporter/)

## 6 prometheus作为grafana数据库
#### 6.1 grafana的设置
- 1 设置datasource为prometheus
- 2 正常流程配置prometheus
#### 6.2 prometheus的query语句
- instant vector 一个时间序列的集合，是同一个timestamp的不同sample集合（对比influxdb的point）
- range vector 一个时间序列集合，包含一段时间的data ponit（对比influxdb的timeseries理解）
    - 只有range vector可以画图
- scalar 一定数量的float point
- string string类型（目前没有使用）
#### 6.3 grafana prometheus query的配置
参考influxdb展现配置，具体查询语句见本文[prometheus query](####查询)

## 7 prometheus告警系统 Alertmanager(可跳过，目前用不到，可以了解)
alertmanager配合prometheus有以下几大功能：
- **Grouping** 将类型一致的告警当做一条告警发出，使其不会重复发出一类告警
- **Inhibition**  抑制一些报警的产生，如果已有同类报警的话
- **Silences** 一些特定的报警在特定的时间下不会报出
- **Client behavior** 一些高阶的应用
- **High Availability** 支持集群高可用性



 