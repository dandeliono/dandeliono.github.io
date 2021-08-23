# InfluxDB 通识篇
## 什么是时序数据库？

时序数据库全称时间序列数据库，英文名 Time Series DataBase，缩写 TSDB。

这种数据库专门用作处理时间序列数据。

那什么是时间序列数据呢？就是随着时间变化而源源不断产生的数据。

举个例子，Mac 电脑上的活动监视器就是一种时间序列数据，每隔几秒它都会获取电脑上各个部件最新的数据。

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-23%2015-08-12/30840c35-0dea-47e8-a08a-15161d032440.webp?raw=true)
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-23%2015-08-12/7ae745ab-abd9-4661-b647-d5186b50c7a1.webp?raw=true)
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-23%2015-08-12/c2f0b680-14fb-42fe-b433-6edef9c26c81.webp?raw=true)
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-23%2015-08-12/85c20a4a-abbc-4797-9e19-b38fab432766.webp?raw=true)
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-23%2015-08-12/8f6c70ea-6e11-412f-8209-7ccb54263442.webp?raw=true)

## 为什么需要时序数据库？

随着物联网和大数据时代的到来，全球每天产生的数据量大到令人难以想象。这些数据受到业务场景的限制分为不同的种类，每个种类对存储都有不同的要求。单凭传统的 RDBMS 很难完成各种复杂场景下的数据存储。

这时我们就需要根据不同的数据特性和业务场景的要求，选择不同的数据库。

一般选择使用哪个数据库，要从低响应时间（Low Response Time）、高可用性（High Availability）、高并发（High Concurrency）、海量数据（Big Data）和可承担成本（Affordable Cost）五个维度去权衡。

数据库的种类非常繁多，举几个常见的类型来对比一下各自的特点。

关系型数据库主流代表有 MySQL、Oracle 等，优点是具有 ACID 的特性，各方面能力都比较均衡。缺点是查询、插入和修改的性能都很一般。

KV 数据库主流带代表有 Redis、Memcached 等，优点是存储简单、读写性能极高。缺点是存储成本非常高，不适合海量数据存储。

文档型数据库最流行的是 MongoDB，相比 MySQL，数据结构灵活、更擅长存储海量数据，在海量数据的场景下读写性能很强。缺点是占用空间会很大。

搜索引擎数据库最流行的是 ElasticSearch，非常擅长全文检索和复杂查询，性能极强，并且天生集群。缺点是写入性能低、字段类型无法修改、硬件资源消耗严重。

而时序数据库，最初诞生的目的很大程度上是在对标 MongoDB，因为在时序数据库出现之前，存储时序数据这项领域一直被 MongoDB 所占据。

时序数据库一哥 InfluxDB 的公司 InfluxData，曾在 2018 年发表了一篇关于 [InfluxDB vs MongoDB 的博客](https://link.juejin.cn/?target=https%3A%2F%2Fwww.influxdata.com%2Fblog%2Finfluxdb-is-27x-faster-vs-mongodb-for-time-series-workloads%2F "https&#x3A;//www.influxdata.com/blog/influxdb-is-27x-faster-vs-mongodb-for-time-series-workloads/")。文中使用 InfluxDB v1.7.2 和 MongoDB v4.0.4 做对比，得出 InfluxDB 比 MongoDB 快 2.4 倍的结论。当然可信度有待考量。

总之，时序数据库的特点是：持续高性能写入、高性能查询、低存储成本、支持海量时间线、弹性。

## 为什么选择 InfluxDB？

虽然时序数据库早在 13 年左右就已经出现，但真正流行的时间非常晚，一直到了 17、18 年才稍微普及，即使到了今天，时序数据库在 DB Engiens 的数据库排名中仍然很落后，最靠前的 InfluxDB 也仅仅排在了 28 位。

选择 InfluxDB 的原因非常简单，因为它目前是最流行的时序数据库。

InfluxDB 之所以能够从众多时序数据库中脱颖而出，除了自身强大以外，活跃的社区、合理的商业模式和营销功不可没。

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-23%2015-08-12/9f2af904-d1c7-458c-95a0-9dab417ff67d.webp?raw=true)

既然时序数据库很好，为什么排名如此考后？原因是应用场景少，记录运维监控和物联网实时数据是时序数据库的用武之地。而大多数的系统使用 MySQL、MongoDB 和 Redis 这些主流数据库就可以很好地支撑。

除了 InfluxDB 以外，还有几个比较流行的时序数据库，比如基于 PostgreSQL 的 TimeScaleDB，目前排在 97 位。基于 HBase 的 OpenTSDB，排在 122 位。基于 Cassandra 的 KairosDB，目前排在 201 位。

## InfluxDB 简介

InfluxDB 是 InfluxData 公司在 2013 年开源的数据库，是为了存储物联网设备、DevOps 运维这类场景下大量带有时间戳数据而设计的。

InfluxDB 源码采用 Go 语言编写，在 InfluxDB OSS 的版本中，部署方式上又分为两个版本，单机版和集群版。单机版开源，目前在 [github](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Finfluxdata%2Finfluxdb "https&#x3A;//github.com/influxdata/influxdb") 上有 21k+ star。集群版闭源，走商业路线。

个人认为单机版的 InfluxDB 比较鸡肋。因为一旦选择使 InfluxDB，那么数据量肯定一定达到了某个很高的程度。这时候必须使用集群版。而在数据量不够高的情况下，InfluxDB 并不会比 MongoDB 或者 ElasticSearch 有更明显的优势。

考虑到学习成本、简化上手难度，InfluxDB1.x 采用了一种类似 SQL 的 InfluxQL 语言来操作数据。2019 年 1 月推出了 InfluxDB2.0 alpha 版本。受到 2018 年最流行的脚本语言 JavaScript 影响，推出了全新的查询语言 Flux。并在 2020 年底推出了 InfluxDB 2.0 正式版本，该版本又分为两个系列，云模式的 InfluxDB Cloud 和独立部署的 InfluxDB OSS。

Flux 不是绑定在 InfluxDB 上的查询脚本语言，它是一个独立的项目，图灵完备，便于处理数据，也可以用作 InfluxDB 以外。

由于 InfluxDB 的流行程度不高，而且 2.0 版本也推出不久，所以在国内搜索到的很多 InfluxDB 相关资料都是在讲述 1.x 的内容，参考意义不大，目前最好的学习路径是 [InfluxDB 官方文档](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.influxdata.com%2Finfluxdb%2Fv2.0%2F "https&#x3A;//docs.influxdata.com/influxdb/v2.0/")。本文内容全部基于 InfluxDB OSS 2.0 版本。

## TICK 与 InfluxDB

TICK 是 InfluxData 平台的组件集的缩写，分别代表四大组件：Telegraf（数据收集器）、InfluxDB（时序数据库）、Chronograf（可视化 UI）和 Kapacitor（监控服务）。

InfluxData 公司的愿景是帮助人们处理时序数据，仅依靠一个时序数据库是不够的，还需要解决因为时序数据自身产生的一系列问题。因此 InfluxData 决定设计并开发 TICK。

在早期 Kapacitor 的脚本语言是 TICKScript，但是并不好用，遭受到社区中很多人的诟病。因此出现了 Flux。Flux 的功能性比 InfluxQL 更强，比 TICKScript 更易用。

随着 Flux 的逐渐发展，InfluxDB 的能力范围也在逐步扩展。

## 基本概念

InfluxDB 的数据模型概念和 RDBMS 稍有不同，下面是和 MySQL 的概念对照表。

| InfluxDB    | MySQL    | 解释                    |
| ----------- | -------- | --------------------- |
| Buckets     | Database | 数据桶 - 数据库，即存储数据的命名空间。 |
| Measurement | Table    | 度量 - 表。               |
| Point       | Record   | 数据点 - 记录。             |
| Field       | Field    | 未设置索引的字段。             |
| Tag         | Index    | 设置了索引的字段。             |

## 安装部署

安装方式有好几种，这里介绍如何使用 Docker 进行安装。

首先从 Docker 拉取镜像：

```bash
docker pull influxdb
```

然后快速创建容器：

```bash
docker run -d --name influxdb -p 8086:8086 influxdb
```

启动成功后访问 [http://127.0.0.1:8086/](https://link.juejin.cn/?target=http%3A%2F%2F127.0.0.1%3A8086%2F "http&#x3A;//127.0.0.1:8086/") 就可以看到页面了。

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-23%2015-08-12/74f5800a-1f6c-434e-bf50-570e5e49bbb5.webp?raw=true)

之后填写初始化信息，完成初始化。

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-23%2015-08-12/a537041c-22ac-4aca-ac7e-3c3eaa2436fb.webp?raw=true)

## 基本配置

### 将数据持久化到 Docker 容器之外

首先创建一个目录。

```bash
mkdir influxdb_docker_data_volume && cd $_
```

在这个目录下运行启动命令，并添加 volume 参数。

```bash
docker run -d --name influxdb -p 8086:8086 --volume $PWD:/root/.influxdbv2 influxdb
```

这样容器中的数据就会存储到当前目录。

## 写数据

InfluxDB 写数据的方式有两种，一是使用不同语言的客户端库，二是使用 Telegraf 插件。

这里介绍使用客户端库来进行写数据。

在 Load Data 页面上，有 Sources 标签，其中又有 Client Libraries 和 Telegraf Plugins 两个分类。

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-23%2015-08-12/fcf6b1a6-8c2e-47a2-bc30-5c95537e2fef.webp?raw=true)

这里选择 Go 语言，点开后会有示例代码。

写数据最少需要 4 个基础信息。

1.  组织 ID（org ID）
2.  存储桶（bucket ID）
3.  身份认证令牌（authentication token）
4.  数据库地址（InfluxDB URL）

写数据的数据格式有两种，第一种是 InfluxDB Line Protocol 格式。

### InfluxDB Line Protocol

完整代码如下：

```go
package main

import (
	"fmt"

	influxdb2 "github.com/influxdata/influxdb-client-go/v2"
)

func main() {
	
	const token = "vgqVL_p-qbSpQO0DIzU4QcRgaEyQM-wcEmK2cOkDUHAiQYwLOba7qEZr9Xcq3YvZ2UH-ovu9RG7XkMwChO6eeA=="
	const bucket = "test"
	const org = "lzq"

	client := influxdb2.NewClient("http://127.0.0.1:8086", token)
	
	defer client.Close()

	
	writeAPI := client.WriteAPI(org, bucket)

	
	writeAPI.WriteRecord(fmt.Sprintf("stat,unit=temperature avg=%f,max=%f", 23.5, 45.0))
	writeAPI.WriteRecord(fmt.Sprintf("stat,unit=temperature avg=%f,max=%f", 22.5, 45.0))
	
	writeAPI.Flush()
}
```

InfluxDB Line Protocol 本质上是一个具有约定格式的字符串。通过这个字符串形成一个记录，这个字符串必须包含一个测量（measurement）和一组字段（field），同时可能会包含 N 个标签（tag）和一个时间戳（timestamp）。

如果不附带时间戳，那么 InfluxDB 会使用其主机的当前系统时间，单位默认为纳秒。

### Data Point

另一种是数据点（Data Point）的数据格式。

这种格式又分为两种风格的 API。

第一种风格是一个函数，传递 N 个参数。

示例代码：

```go
p := influxdb2.NewPoint("stat",
  map[string]string{"unit": "temperature"},
  map[string]interface{}{"avg": 24.5, "max": 45},
  time.Now())

writeAPI.WritePoint(p)
```

第二种风格是类似 DSL 的 API。

代码如下：

```go

p = influxdb2.NewPointWithMeasurement("stat").
  AddTag("unit", "temperature").
  AddField("avg", 23.2).
  AddField("max", 45).
  SetTime(time.Now())

writeAPI.WritePoint(p)
```

## 删数据

InfluxDB 不支持修改数据，但是可以删除数据。

删数据的方式有两种，一种是 InfluxDB CLI，一种是 HTTP API。

### InfluxDB CLI

运行 `influx delete`  命令即可删除数据，需要附带参数。

1.  bucket：指定某个数据桶。
2.  start 和 stop：指定删除的数据时间戳范围。
3.  predicate：可选项，删除符合某种条件的数据。

示例：

```bash
influx delete --bucket example-bucket \
  --start 2020-03-01T00:00:00Z \
  --stop 2020-11-14T00:00:00Z
```

带有条件的示例：

```bash
influx delete --bucket example-bucket \
  --start '1970-01-01T00:00:00Z' \
  --stop $(date +"%Y-%m-%dT%H:%M:%SZ") \
  --predicate '_measurement="example-measurement" AND exampleTag="exampleTagValue"'
```

### HTTP API

和 CLI 的方式类似。调用 `api/v2/delete`  即可。

需要符合以下条件。

1.  请求方式（Method）：POST。
2.  请求头（Headers）：带有 `Authorization`  用于验证身份，请求格式为 `application/json`。
3.  查询参数（QueryParams）：org/orgID，指定组织。bucket/bucketID，指定数据桶。
4.  请求体（Body）：start：表示开始时间，stop：表示结束时间，predicate（可选）：表示删除条件。

请求示例：

```bash
curl --request POST http://localhost:8086/api/v2/delete/?org=example-org&bucket=example-bucket \
  --header 'Authorization: Token <YOURAUTHTOKEN>' \
  --header 'Content-Type: application/json' \
  --data '{
    "start": "2020-03-01T00:00:00Z",
    "stop": "2020-11-14T00:00:00Z"
  }'
```

我不建议在 InfluxDB 中对数据进行删除操作。

时序数据库更适合存储完整的原始数据，而经过分析和提炼后的价值更高的数据，可以存入 MongoDB 或者 MySQL。

## 读数据

InfluxDB 读数据的方式有 5 种。

1.  InfluxDB UI
2.  InfluxDB HTTP API
3.  Flux REPL
4.  InfluxDB CLI
5.  Client libraries

这里介绍第 5 种，使用客户端的方式来读取数据。

下面是代码演示：

```go
client := influxdb2.NewClient(url, token)
defer client.Close()
query := fmt.Sprintf("from(bucket:\"%v\")|> range(start: -1h) |> filter(fn: (r) => r._measurement == \"stat\")", bucket)

queryAPI := client.QueryAPI(org)

result, err := queryAPI.Query(context.Background(), query)
if err == nil {
    
    for result.Next() {
        
        if result.TableChanged() {
            fmt.Printf("table: %s\n", result.TableMetadata().String())
        }
        
        fmt.Printf("value: %v\n", result.Record().Value())
    }
    
    if result.Err() != nil {
        fmt.Printf("query parsing error: %\n", result.Err().Error())
    }
} else {
    panic(err)
}
```

其中关键的是第 3 行名为 query 的字符串变量，它是使用 flux 语法编写的一段脚本。

    from(bucket:"test")
      |> range(start: -1h)
      |> filter(fn: (r) => r._measurement == "stat")
    复制代码

`from`  表示从哪个数据源检索数据。

`range`  表示根据时间范围过滤数据。start: -1 表示当前时间减掉 1 个小时。

`filter`  表示自定义过滤条件，其中 fn 是一个函数，在函数内定义规则，语法和 JavaScript 中 Array 的 filter 函数极其类似。

`|>`  表示管道前移符，将数据通过管道的形式传递到下一个操作中。

 [https://juejin.cn/post/6947575345570643981](https://juejin.cn/post/6947575345570643981)
