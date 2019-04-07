---
title: ElasticSearch入门
tags: ElasticSearch
      搜索
article_header:
  type: cover
  image:
    src: 
key: ElasticSearch
sharing: true
show_author_profile: false
comment: true
aside:
  toc: true
---



![](/assets/huangImg/es35.png)

<!--more-->

`声明：文中内容引自慕课网学习教程`{:.error}

## 1. 应用场景

+ 海量数据分析引擎
+ 站内搜索引擎
+ 数据仓库

### 一线公司应用场景

+ 实时分析公众对文章的回应
+ GitHub-站内实时搜索
+ 百度-日志监控平台

## 2. 安装（确保系统JDK8环境配置好）

+ **单实例安装**

  + [elasticSearch下载地址](https://www.elastic.co/cn/downloads/elasticsearch)
  + ``wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.2.tar.gz``
    + **wget**命令可以直接通过http协议下载，当然我们也可以在网页下载
  + ``tar -vxf 解压``
  + ``cd elasticsearch-5.5.2``
    + 核心文件有
    + config，modules，plugins
  + 启动命令：``./bin/elasticseatch``
  + 出现下图界面，started启动成功, 默认9200端口

  ![](/assets/huangImg/es1.png)

  + 访问url，es默认返回的 ***json*** 数据格式

  ![](/assets/huangImg/es2.png)

+ **插件安装**

  + head插件，提供了友好的web界面，基本信息查看

  + [head插件下载地址](https://github.com/mobz/elasticsearch-head)

  + 下载完文件为``master.zip``文件，进行解压``unzip master.zip``

  + 进入解压完成的文件``cd elasticsearch-head-master``

  + ``node -v``可以看到版本号

  + ``npm install``安装

  + ``npm run start``服务启动

    ![](/assets/huangImg/es3.png)

  + 访问浏览器效果：

    ![](/assets/huangImg/es4.png)

**此时连接是未连接，因为我刚刚将es服务停了，现在需要配置几个东西。**

+ **修改配置**

  + 原因：es和head属于两个进程，其之间的访问具备跨域问题，所以需要配置修改

  + 进入elasticsearch-5.5.2 修改 yml

    ![](/assets/huangImg/es5.png)

  + 添加最后两行

    ![](/assets/huangImg/es6.jpg)


  + 启动 加 -d参数 ``./bin/elasticsearch -d`` 后台启动
  + 重新启动head``cd  elasticsearch-head-master``，``npm run start``
  + 刷新浏览器，连接状态green

+ **分布式安装视频**

  + [慕课视频-分布式安装](https://www.imooc.com/video/15766)



## 3. 基础概念

+ 集群和节点
  + master slave1 slave2 像这样的一组节点构成了一个集群。
+ 索引，文档，类型
  + 索引：含相同属性的文档集合。≈ MySQL database
  + 类型：索引可以定义一个或者多个类型，文档必须属于一个类型。=MySQL 表
  + 文档：可以被索引的基本数据单位。 = MySQL 一行记录
+ 实例解释
  + 假设现在有个信息查询系统，用ES来存储。
  + 这样分布，汽车索引，家具索引，图书索引
  + 图书索引分为很多类型，科普类，玄幻类
  + 科普类具体到每一本书籍，就是文档。最小的存储单位
+ 分片 备份
  + 分片
    + 一个索引有多个分片，每个分片都是一个Lucene的索引
    + 如果一个索引的数据量很大，比如图书索引数据量太大，硬盘压力大，就可以创建多个分片。
  + 备份
    + 拷贝一份分片就完成了分片的备份，除了主分片挂掉以后备份，还可以分担搜索的压力。
  + ES创建索引默认**5个分片，1个备份**
    + 分片只能在创建索引的时候指定
    + 备份可以后期动态修改

## 4. 基本用法

+ 创建索引

  ![](/assets/huangImg/Inkedes6_LI.jpg)

+ 查看页面

  ![](/assets/huangImg/es7.png)

**上图：0 1 2 3 4 五个分片。0粗线框是主分片，0细线是粗线的备份分片、、**

上面这种创建索引的方式，创建出来的索引是没有数据结构的，也就是没有mapping。

下面我们来演示创建具备mapping的索引。

通过***postman***来进行http请求。

+ 创建一个结构化的people索引

![](/assets/huangImg/es8.png)

+ **插入**

  + 指定文档ID插入

    ![](/assets/huangImg/es9.png)

    返回结果

    ![](/assets/huangImg/es10.png)

    可以在head界面浏览，看插入是否成功

    ​

  + 自动产生ID插入

    + 将上面的127.0.0.1:9200/people/man 后面的id去掉，send。
    + 返回数据的ID就是自动生成的



+ **修改操作**

  + 执行：

    ![](/assets/huangImg/es11.png)

  + 返回：

    ![](/assets/huangImg/es12.png)

  + 还是``update``操作，使用脚本

    ![](/assets/huangImg/es13.png)

    ![](/assets/huangImg/es14.png)

+ **删除操作**

  + delete请求删除 ***文档***

    ![](/assets/huangImg/es15.png)

  + delete请求删除***索引***      `危险操作，谨慎执行`{:.error}

    ![](/assets/huangImg/es16.png)



+ **查询操作**

  + 事先提供了一些数据，瓦力老师

    ![](/assets/huangImg/es17.png)

  + 简单查询

    ![](/assets/huangImg/es18.png)

  + 查询所有

    + ``http://localhost:9200/_search``   查询所有索引+所有类型

    + ``http://localhost:9200/book/_search``    在book索引中查询所有类型(类型又包含其文档)

      ![](/assets/huangImg/es19.png)

    + ``http://localhost:9200/book/novel/_search``  在book索引 novel类型下 查询所有文档 

## 5. 查询功能

列举一些查询语法：

+ ***匹配title属性***

```json
{
	"query":{
		"match":{
			"title":"elasticsearch"
		}
	}
}
```

+ ***自定义排序，按照日期***

```json
{
	"query":{
		"match":{
			"title":"elasticsearch"
		}
	},
	"sort":[
		{"publish_date":{"order":"desc"}}
	]
}
```

+ ***聚合查询***   (mysql中的group by)

```json
{
    "aggs":{
        "group_by_word_count":{
            "terms":{
                "field":"word_count"
            }
        }
    }
}
```

```json
{
    "aggs":{
        "group_by_word_count":{
            "terms":{
                "field":"word_count"
            }
        },
        "group_by_publish_date":{
            "terms":{
                "field":"publish_date"
            }
        }
    }
}
```

+ ***对某个字段进行一系列计算***

  发送：

  ![](/assets/huangImg/es21.png)

  返回：

  ![](/assets/huangImg/es23.png)

## 6. 高级查询（全篇检查）

+ 模糊查询（含有"elasticsearch"和 ''入门''的）

  ![](/assets/huangImg/es24.png)

+  完全包含查询（只查询出含有"elasticsearch"）

  match改成***match_phrase***

  ![](/assets/huangImg/es22.png)

+ 查询指定属性里有某个字段的（查询title或者author里面包含“瓦力”的）

  ![](/assets/huangImg/es25.png)

+ string语法查询

  查询语法

  ![](/assets/huangImg/es27.png)

  返回结果

  ![](/assets/huangImg/es26.png)

  还可以加上``field：["title","author"]``指定属性

## 7. 结构化查询（字段级别的查询）

+ **``6高级查询中的都是针对全篇检查的,下面我们针对结构化的数据来查询。``**

+ ![](/assets/huangImg/es28.png)

+ 查询 word_count 在1000-2000范围的（gte 大于等于/ gt大于）

  ![](/assets/huangImg/es29.png)

+ 过滤查询

  过滤相比query较快，会缓存数据，需要结合布尔类型来使用

  ![](/assets/huangImg/es30.png)

## 8. 复合查询

+ 固定分数查询

  ES查询出来的结果，会有一个匹配度的评分，_score，都不一样，我们可以把这个评分固定为一样的。

  ![](/assets/huangImg/es31.png)

  指定分数为 **2**

  ![](/assets/huangImg/es32.png)

+ boo+should查询

  match关系为***或***，满足一个即可

  ![](/assets/huangImg/es33.png)

+ boo+must

  match关系为***与***

  ![](/assets/huangImg/es34.png)

************************

## ending