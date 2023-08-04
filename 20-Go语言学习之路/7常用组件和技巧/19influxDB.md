# influxDB

> https://www.liwenzhou.com/posts/Go/go_influxdb/

本文介绍了`influxDB`时序数据库及Go语言操作`influxDB`。

[InfluxDB](https://www.influxdata.com/)是一个开源分布式时序、事件和指标数据库。使用Go语言编写，无需外部依赖。其设计目标是实现分布式和水平伸缩扩展。

## 安装

### 下载

https://portal.influxdata.com/downloads/

这里需要注意因为这个网站引用了google的api所以国内点页面的按钮是没反应的，怎么办呢？

按照下图所示，按`F12`打开浏览器的控制台，然后点击`Elements`，按下`Ctrl/Command+F`搜索`releases/influxdb`，按回车查找自己所需版本的下载地址。![download influxDB](https://www.liwenzhou.com/images/Go/influxDB/influxdb_01.png)

Mac和Linux用户可以点击https://v2.docs.influxdata.com/v2.0/get-started/下载。

### 安装

将上一步的压缩包，解压到本地。

## influxDB介绍

### 名词介绍

| influxDB名词 | 传统数据库概念 |
| :----------: | :------------: |
|   database   |     数据库     |
| measurement  |     数据表     |
|    point     |     数据行     |

### point

influxDB中的point相当于传统数据库里的一行数据，由时间戳（time）、数据（field）、标签（tag）组成。

| Point属性 |                传统数据库概念                |
| :-------: | :------------------------------------------: |
|   time    |     每个数据记录时间，是数据库中的主索引     |
|   field   | 各种记录值（没有索引的属性），例如温度、湿度 |
|   tags    |       各种有索引的属性，例如地区、海拔       |

### Series

`Series`相当于是 InfluxDB 中一些数据的集合，在同一个 database 中，retention policy、measurement、tag sets 完全相同的数据同属于一个 series，同一个 series 的数据在物理上会按照时间顺序排列存储在一起。

[想要了解更多](http://blog.fatedier.com/2016/08/05/detailed-in-influxdb-tsm-storage-engine-one/)

## Go操作influxDB

### 安装

#### influxDB 1.x版本

```bash
go get github.com/influxdata/influxdb1-client/v2
```

#### influxDB 2.x版本

```bash
go get github.com/influxdata/influxdb-client-go
```

### 基本使用

```go
package main

import (
	"fmt"
	"log"
	"time"

	client "github.com/influxdata/influxdb1-client/v2"
)

// influxdb demo

func connInflux() client.Client {
	cli, err := client.NewHTTPClient(client.HTTPConfig{
		Addr:     "http://127.0.0.1:8086",
		Username: "admin",
		Password: "",
	})
	if err != nil {
		log.Fatal(err)
	}
	return cli
}

// query
func queryDB(cli client.Client, cmd string) (res []client.Result, err error) {
	q := client.Query{
		Command:  cmd,
		Database: "test",
	}
	if response, err := cli.Query(q); err == nil {
		if response.Error() != nil {
			return res, response.Error()
		}
		res = response.Results
	} else {
		return res, err
	}
	return res, nil
}

// insert
func writesPoints(cli client.Client) {
	bp, err := client.NewBatchPoints(client.BatchPointsConfig{
		Database:  "test",
		Precision: "s", //精度，默认ns
	})
	if err != nil {
		log.Fatal(err)
	}
	tags := map[string]string{"cpu": "ih-cpu"}
	fields := map[string]interface{}{
		"idle":   201.1,
		"system": 43.3,
		"user":   86.6,
	}

	pt, err := client.NewPoint("cpu_usage", tags, fields, time.Now())
	if err != nil {
		log.Fatal(err)
	}
	bp.AddPoint(pt)
	err = cli.Write(bp)
	if err != nil {
		log.Fatal(err)
	}
	log.Println("insert success")
}

func main() {
	conn := connInflux()
	fmt.Println(conn)

	// insert
	writesPoints(conn)

	// 获取10条数据并展示
	qs := fmt.Sprintf("SELECT * FROM %s LIMIT %d", "cpu_usage", 10)
	res, err := queryDB(conn, qs)
	if err != nil {
		log.Fatal(err)
	}

	for _, row := range res[0].Series[0].Values {
		for j, value := range row {
			log.Printf("j:%d value:%v\n", j, value)
		}
	}
}
```