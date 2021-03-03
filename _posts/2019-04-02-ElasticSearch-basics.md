---
layout: post
title:  "ElasticSearch 基础"
categories: tools
tags:  es
author: 刚子

---

* content
  {:toc}

总结ElasticSearch的基础知识点

## 一、简易教程

### 概念

### 1. 下载安装

下载[安装文件](https://www.elastic.co/downloads/past-releases/elasticsearch-5-5-3)到`/opt/elasticsearch`目录下面并解压

```bash
>cd /opt/elasticsearch
>tar -zxvf elasticsearch-5.5.3.tar.gz
```

### 2. 启动并访问

```bash
>cd /opt/elasticsearch/elasticsearch-5.5.3
>./bin/elasticsearch
##./bin/elasticsearch -d 后台启动
```

root账户启动会报错：can not run elasticsearch as root，创建独立的用户来启动

```bash
>groupadd esgroup
>useradd esuser -g esgroup
>passwd esuser
>chown -R esuser:esgroup elasticsearch/
```

root用户关闭防火墙

```bash
>vi /etc/selinux/config
SELINUX=disabled
>systemctl stop firewalld.service
>systemctl disable firewalld.service
>setenforce 0
>getenforce 0
```

es配置可以使用ip访问

```bash
>vi config/elasticsearch.yml
network.host: 192.168.237.129   #或者0.0.0.0允许所有人访问
>su root
>vim /etc/security/limits.conf
esuser hard nofile 65536
esuser soft nofile 65536
>vi /etc/sysctl.conf
vm.max_map_count=655360
>sysctl -p
```

### 3. 创建、删除索引

```bash
curl -X PUT 'http://localhost:9200/weather'
curl -X DELETE 'http://localhost:9200/weather'
```

> 一个索引可以有多个分片来完成存储，但是主分片的数量是在索引创建时就指定好的，且无法修改，所以尽量不要只为数据存储建立一个索引，否则后面数据膨胀时就无法调整了。笔者的建议是对于同一类型的数据，根据时间来分拆索引，比如一周建一个索引，具体取决于数据增长速度。

### 4. 添加文档

```bash
curl -XPUT "http://localhost:9200/movies/movie/1" -d'
{
    "title": "The Godfather",
    "director": "Francis Ford Coppola",
    "year": 1972,
    "genres": ["Crime", "Drama"]
}'

```

### 5. 搜索所有文档

```bash
http://localhost:9200/_search # 搜索所有索引和所有类型
http://localhost:9200/movies/_search # 在电影索引中搜索所有类型
http://localhost:9200/movies/movie/_search # 在电影索引中显式搜索电影类型的文档
```

### 6. 安装中文分词插件

```bash
>./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.3/elasticsearch-analysis-ik-5.5.3.zip
```

### 7. 创建索引，并对文档字段进行中文分词

```bash
curl -X PUT 'http://localhost:9200/accounts' -d '
{
  "mappings": {
    "person": {
      "properties": {
        "user": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "desc": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        }
      }
    }
  }
}'


curl -XPUT "http://localhost:9200/accounts/person/1" -d'
{
    "user": "zhangsan",
    "title": "管理员",
    "desc": "我是系统管理员"
}'

curl -XPUT "http://localhost:9200/accounts/person/2" -d'
{
    "user": "lisi",
    "title": "组长",
    "desc": "我是组长"
}'
```

### 8. 搜索指定字段

```bash
curl 'http://localhost:9200/accounts/person/_search'  -d '
{
  "query" : { "match" : { "desc" : "系统" }},
  "from": 1,
  "size": 1
}'

# from指定分页起始位置
# size表示每页几条数据
```

### 9. 查询索引信息

```bash
# 列出所有的索引
curl localhost:9200/_cat/indices?v
# 查看索引的文档数量
curl localhost:9200/_cat/count/accounts?v
# 查看文档字段信息
curl localhost:9200/accounts/person/_mapping
# 删除索引
curl  -X DELETE localhost:9200/accounts
```

[重要配置的修改](https://www.elastic.co/guide/cn/elasticsearch/guide/current/important-configuration-changes.html#_%E6%8C%87%E5%AE%9A%E5%90%8D%E5%AD%97)

## 二、查询DSL

* 添加雇员索引文档

```bash
curl -X PUT "localhost:9200/megacorp/employee/1" -H 'Content-Type: application/json' -d'
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
'
```

* 创建文档，非更新

```bash
PUT /website/blog/123?op_type=create
# or
PUT /website/blog/123/_create
```

* 检查文档是否存在

```bash
curl -i -XHEAD http://localhost:9200/website/blog/123
```

* QueryString搜索

```bash
curl -X GET "localhost:9200/megacorp/employee/_search?q=last_name:Smith"
```

* 查询表达式(DSL:domain-specific language)搜索指定字段

```bash
curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
'
```

* 过滤器--对查询结果进行进一步过滤

```bash
# 搜索姓氏为 Smith 的雇员，但这次我们只需要年龄大于 30 的
curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
'
```

* 全文搜索

```bash
curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
'
```

* 短语搜索--精确匹配内容中出现短语的文档

```bash
curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
'
```

* 高亮搜索

```bash
curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
'
```

更多高亮设置参考[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-highlighting.html#highlighting-settings)

* 数据分析--聚合、汇总

[分析](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_analytics.html#_analytics)

* 通过版本号更新数据

```bash
curl -X PUT "localhost:9200/website/blog/1?version=1" -H 'Content-Type: application/json' -d'
{
  "title": "My first blog entry",
  "text":  "Starting to get the hang of this..."
}
'
```

更多请参考[乐观并发控制](https://www.elastic.co/guide/cn/elasticsearch/guide/current/optimistic-concurrency-control.html#optimistic-concurrency-control)

### 案例

```bash
PUT _template/coupon_search_template
{
    "order":10,
    "index_patterns":[
        "coupon_search_*"
    ],
    "settings":{
        "index.number_of_replicas":"1",
        "index.number_of_shards":"2",
        "index.refresh_interval":"1s",
        "index.search.slowlog.level":"info",
        "index.search.slowlog.threshold.fetch.debug":"100ms",
        "index.search.slowlog.threshold.fetch.info":"500ms",
        "index.search.slowlog.threshold.fetch.trace":"50ms",
        "index.search.slowlog.threshold.fetch.warn":"1s",
        "index.search.slowlog.threshold.query.debug":"100ms",
        "index.search.slowlog.threshold.query.info":"500ms",
        "index.search.slowlog.threshold.query.trace":"50ms",
        "index.search.slowlog.threshold.query.warn":"1s",
        "index.unassigned.node_left.delayed_timeout":"1d"
    },
    "mappings":{
        "dynamic":false,
        "properties":{
            "ztoUserId":{
                "type":"keyword"
            },
            "couponSn":{
                "type":"keyword"
            },
            "mobile":{
                "type":"keyword"
            },
            "orderId":{
                "type":"keyword"
            },
            "receiveTime":{
                "type":"date",
                "format":"yyyy-MM-dd HH:mm:ss",
                "locale":"zh_CN"
            }
        }
    },
    "aliases":{
        "coupon_alias":{}
    }
}

PUT coupon_search_all

POST /coupon_search_all/_doc
{
"ztoUserId":"123456",

"couponSn":"CS0001",

"mobile":"13057271932",

"orderId":"O0001",

"receiveTime":"2020-01-01 08:00:00"
}

GET /coupon_search_all/_search

GET /coupon_search_all/_search
{
  "query": {
    "match": {
      "mobile": "13057271932"
    }
  }
}

GET /coupon_search_all/_search
{
  "query": {
    "wildcard": {
      "mobile": {
        "value": "1305727*"
      }
    }
  },
  "sort": [
    {
      "receiveTime": {
        "order": "desc"
      }
    }
  ]
}

GET /coupon_search_all/_search
{
  "query": {
    "prefix": {
      "mobile": {
        "value": "13057"
      }
    }
  },
  "sort": [
    {
      "receiveTime": {
        "order": "desc"
      }
    }
  ]
}

POST /coupon_search_all/_update/QAivYXcB89UkBtu30_Fl
{
  "doc":{
    "mobile":"13057271933"
  }
}
```

### 字段类型

#### date

```bash
#给example索引新增一个birthday字段，类型为date, 格式可以是yyyy-MM-dd 或 yyyy-MM-dd HH:mm:ss
#添加日期类型的映射
PUT test_label_supplier/docs/_mapping
{
  "properties": {
    "birthday": {
      "type": "date",
      "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis",
      "ignore_malformed": false,
      "null_value": null
    }
  }
}

```

## 三、Kibana

### 下载安装

从[下载地址](https://www.elastic.co/cn/downloads/past-releases/kibana-5-5-3)下载到`/opt/elasticsearch`目录下并解压

```bash
tar -zxvf kibana-5.5.3-linux-x86_64.tar.gz
```

设置任何人都可以访问

```bash
>vim config/kibana.yml
server.host: "0.0.0.0"
```

### 启动并访问

```bash
>./bin/kibana
>./bin/kibana &  #后面添加&代表后台启动，shell窗口执行exit命令后kibana会一直后台启动
```

### 查询语法

[query string syntax](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/query-dsl-query-string-query.html#query-string-syntax)

## 四、中文分词

### 英文分词示例

```bash
PUT test/doc/1
{
  "msg":"Eating an apple a day keeps doctor away"
}

# 使用单词eat无法搜索到包含eating的内容
POST test/_search
{
  "query":{
    "match":{
      "msg":"eat"
    }
  }
}

# 分析一下字段的分词规则，发现默认的standard分词器没有把eating切分为eat
POST test/_analyze
{
  "field": "msg",
  "text": "Eating an apple a day keeps doctor away"
}

# 新加一个字段，指定写分词器为english，写分词器一经指定就不能修改，如果修改的话只能重建索引
PUT test/_mapping/doc
{
  "properties": {
    "msg_english":{
      "type":"text",
      "analyzer": "english"
    }
  }
}

POST test/doc/2
{
  "msg":"Eating an apple a day keeps doctor away",
  "msg_english":"Eating an apple a day keeps doctor away"
}

# 这时候再分析一下字段，发现eat已经可以匹配eating
POST test/_analyze
{
  "field": "msg_english",
  "text": "Eating an apple a day keeps doctor away"
}

# 尝试搜索一下看看，搜索分词器不指定的话默认与写分词器一致，一般来讲不需要特别指定搜索分词器
POST test/_search
{
  "query":{
    "match":{
      "msg_english":{
        "query": "eat"
      }
    }
  }
}

# standard分词器添加三个过滤器之后效果与english分词器一样，stemmer指的是词干提取
POST _analyze
{
  "char_filter": [], 
  "tokenizer": "standard",
  "filter": [
    "stop",
    "lowercase",
    "stemmer"
  ],
  "text": "Eating an apple a day keeps doctor away"
}
```

### 安装使用中文分词插件ik

```bash
>./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.3/elasticsearch-analysis-ik-5.5.3.zip
```

* 指定索引中文档的分词器的类型为ik

```bash
# 试一下IK的分词器分词效果
POST _analyze
{
  "analyzer": "ik_max_word",
  "text": "我爱北京天安门"
}
# 或者
POST _analyze
{
  "char_filter": [], 
  "tokenizer": "ik_max_word",
  "filter": [],
  "text": "我爱北京天安门"
}

# 设置索引文档字段的分词器，注意只能设置一次，否则只能重新创建索引
PUT /test1
{
  "mappings": {
    "person": {
      "properties": {
        "user": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "desc": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        }
      }
    }
  }
}
PUT test1/person/1
{
  "user":"用户1",
  "title":"标题",
  "desc":"注意只能设置一次，否则只能重新创建索引"
}
GET test1/person/_search
{
  "query": {
    "match": {
      "desc": "注意"
    }
  }
}
# 查看字段分词器用的是哪个
GET test1/person/_mapping
```

### 自定义静态词库文件

```bash
# 创建自定义词库文件my.dic
>/opt/elasticsearch/elasticsearch-5.5.3/config/analysis-ik
>vim my.dic
我爱北京天安门

#修改ik配置文件，指定词库文件
>vim IKAnalyzer.cfg.xml

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典 -->
        <entry key="ext_dict">my.dic</entry>
         <!--用户可以在这里配置自己的扩展停止词字典-->
        <entry key="ext_stopwords"></entry>
        <!--用户可以在这里配置远程扩展字典 -->
        <!-- <entry key="remote_ext_dict">words_location</entry> -->
        <!--用户可以在这里配置远程扩展停止词字典-->
        <!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>

# 重启ES试一下分词效果
POST _analyze
{
  "char_filter": [], 
  "tokenizer": "ik_max_word",
  "filter": [],
  "text": "我爱北京天安门"
}
POST _analyze
{
  "analyzer": "ik_max_word",
  "text": "我爱北京天安门"
}
```

### 自定义动态词库文件

```bash
# 设置动态词库url地址
>cd /opt/tomcat/apache-tomcat-8.5.37/webapps/ROOT
>vim my.dic
我爱

#修改ik配置文件，指定词库文件
>vim IKAnalyzer.cfg.xml

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典 -->
        <entry key="ext_dict">my.dic</entry>
         <!--用户可以在这里配置自己的扩展停止词字典-->
        <entry key="ext_stopwords"></entry>
        <!--用户可以在这里配置远程扩展字典 -->
        <entry key="remote_ext_dict">http://192.168.255.131:8080/my.dic</entry>
        <!--用户可以在这里配置远程扩展停止词字典-->
        <!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>

# 重启ES试一下分词效果
POST _analyze
{
  "char_filter": [], 
  "tokenizer": "ik_max_word",
  "filter": [],
  "text": "我爱北京天安门"
}
# 修改自定义远程词库之后会有最多1分钟生效时间
```

### 维护ik自定义词库到数据库(未经实验)

```java
//提供head请求服务，让ES隔一分钟调用一次，判断词库是否发生变化
@RequestMapping(value="/es/getCustomDic",method=RequestMethod.HEAD)
public void getCustomDic(HttpServletRequest request,HttpServletResponse response) throws Exception{
    String latest_ETags=getLatest_ETags();
    String old_ETags=request.getHeader("If-None-Match");
    if(latest_ETags.equals("")||!latest_ETags.equals(old_ETags)){
        refreshETags();
        response.setHeader("Etag", getLatest_ETags());
    }
}

//相同的服务地址，get请求获取自定义词库字符串
@RequestMapping(value="/es/getCustomDic",method=RequestMethod.GET,produces = {"text/html;charset=utf-8"})
public String getCustomDic(HttpServletRequest request,HttpServletResponse response) throws Exception{
    String old_ETags=request.getHeader("If-None-Match");
    logger.info("get请求，old_ETags="+old_ETags);
    StringBuilder hotwordStr=new StringBuilder();
    //先让热词状态改为生效状态
    HWIceServiceClient.getServicePrx(HotWordIPrx.class).updateHotWordIsEffect("family");
    //说明第一次请求或者最新标示已经更新
    List<String> hotWord=HWIceServiceClient.getServicePrx(HotWordIPrx.class).getAllHotWords("family");
    logger.info("新的热词加入,个数为: "+hotWord.size());
    hotWord.forEach(str->{
        hotwordStr.append(str+"\r\n");
    });
    refreshETags();
    return hotwordStr.toString();
}
```

## 五、同义词

### 维护静态同义词词库

* 创建词库文件

```bash
>cd /opt/elasticsearch/elasticsearch-5.5.3/config
>mkdir analysis
>cd analysis
>vim synonyms.txt
西红柿,番茄,土豆,马铃薯
社保,公积金
```

* 创建索引，添加自定义分词器，指定同义词过滤器、同义词库文件地址

```bash
PUT /synonymtest
{
    "settings": {
    "index": {
      "analysis": {
        "analyzer": {
          "jt_cn": {
            "type": "custom",
            "use_smart": "true",
            "tokenizer": "ik_smart",
            "filter": ["jt_tfr","jt_sfr"],
            "char_filter": ["jt_cfr"]
          },
          "ik_smart": {
            "type": "ik_smart",
            "use_smart": "true"
          },
          "ik_max_word": {
            "type": "ik_max_word",
            "use_smart": "false"
          }
        },
        "filter": {
          "jt_tfr": {
            "type": "stop",
            "stopwords": [" "]
          },
          "jt_sfr": {
            "type": "synonym",
            "synonyms_path": "analysis/synonyms.txt" //这个是相对于${es_home}/config目录而言的地址
          }
        },
        "char_filter": {
            "jt_cfr": {
                "type": "mapping",
                "mappings": [
                    "| => \\|"
                ]
            }
        }
      }
    }
  }
}
```

* 给文档字段创建映射，指定自定义分词器

```bash
PUT /synonymtest/mytype/_mapping
{
    "mytype":{
        "properties":{
            "title":{
                "analyzer":"jt_cn",
                "term_vector":"with_positions_offsets",
                "boost":8,
                "store":true,
                "type":"text"
            }
        }
    }
}
```

* 添加数据

```bash
PUT /synonymtest/mytype/1
{
"title": "番茄"
}
PUT /synonymtest/mytype/2
{
"title": "西红柿"
}
PUT /synonymtest/mytype/3
{
"title": "我是西红柿"
}
PUT /synonymtest/mytype/4
{
"title": "我是番茄"
}
PUT /synonymtest/mytype/5
{
"title": "土豆"
}
PUT /synonymtest/mytype/6
{
"title": "aa"
}
```

* 搜索同义词

```bash
POST /synonymtest/mytype/_search?pretty
{
  "query": {
    "match_phrase": {
      "title": {
        "query": "西红柿",
        "analyzer": "jt_cn"
      }
    }
  },
"highlight": {
    "pre_tags": [
      "<tag1>",
      "<tag2>"
    ],
    "post_tags": [
      "</tag1>",
      "</tag2>"
    ],
    "fields": {
      "title": {}
    }
  }
}
```

### 维护动态同义词词库

* mysql创建同义词维护表

```bash
DROP TABLE IF EXISTS `synonym_config`;
CREATE TABLE `synonym_config` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `synonyms` varchar(128) DEFAULT NULL,
  `last_update_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8;

INSERT INTO `synonym_config` VALUES ('1', '西红柿,番茄,圣女果', '2019-04-01 16:10:16');
INSERT INTO `synonym_config` VALUES ('2', '馄饨,抄手', '2019-04-02 16:10:40');
INSERT INTO `synonym_config` VALUES ('5', '小说,笑说,晓说', '2019-03-31 17:23:36');
INSERT INTO `synonym_config` VALUES ('6', '你好,利好', '2019-04-02 17:27:06');
```

* 创建一个SpringBoot应用，用来提供同义词数据接口供ElasticSearch定期检查导入

[dynamic-synonym-website](https://github.com/QigangZhong/dynamic-synonym-website)

* 编译打包动态同义词ES插件

[elasticsearch-analysis-dynamic-synonym](https://github.com/QigangZhong/elasticsearch-analysis-dynamic-synonym)

```bash
mvn clean package
```

在`target>release`目录下找到xxx.zip，放到`${es_home}/plugins/dynamic-synonym/`下解压，重启ES

```bash
DELETE synonymtest

PUT synonymtest
{
    "settings":{
        "index":{
            "analysis":{
                "filter":{
                    "local_synonym":{
                      "type":"synonym",
                        "synonyms_path":"synonym.txt",
                        "interval":60
                    },
                    "http_synonym":{
                        "type":"dynamic_synonym",
                        "synonyms_path":"http://192.168.237.129:8080/synonym",
                        "interval":60
                    }
                },
                "analyzer":{
                    "ik_max_word_syno":{
                        "type":"custom",
                        "tokenizer":"ik_max_word",
                        "filter":[
                            "http_synonym"
                        ]
                    },
                    "ik_smart_syno":{
                        "type":"custom",
                        "tokenizer":"ik_smart",
                        "filter":[
                            "http_synonym"
                        ]
                    }
                }
            }
        }
    }
}

POST synonymtest/product/_mapping
{
  "product":{
      "properties":{
          "id":{
              "type":"long"
          },
          "name":{
              "type":"text",
              "analyzer":"ik_max_word_syno",
              "search_analyzer":"ik_max_word_syno"
          }
      }
  }
}

GET synonymtest/product/_mapping

PUT /synonymtest/product/1
{
  "id":111,
  "name":"番茄炒蛋"
}

PUT /synonymtest/product/2
{
  "id":222,
  "name":"西红柿炒蛋"
}

GET /synonymtest/product/_search

GET synonymtest/product/_search
{
  "query": {
    "match_phrase": {
      "name": {
        "query": "西红柿", 
        "analyzer": "ik_max_word_syno"
      }
    }
  }
}

POST synonymtest/_analyze
{
  "analyzer": "ik_max_word_syno", 
  "text": "西红柿"
}
```

## 六、扩容

[https://www.elastic.co/guide/cn/elasticsearch/guide/current/_scale_horizontally.html](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_scale_horizontally.html)

## 七、原理

### write、read原理

### refresh、flush

[ES中Refresh和Flush的区别](https://www.jianshu.com/p/15837be98ffd)

### 倒排索引、正排索引

## 八、Java SDK

[Java API](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/index.html)

## 问题

* translog是什么？

[elasticsearch 事务日志translog](https://www.cnblogs.com/fengda/p/10348606.html)

* match、match_phrase、term、bool区别

[Elasticsearch查询match、term和bool区别](https://www.cnblogs.com/fengda/p/10348616.html)

* keyword、text区别

[elasticsearch的keyword与text的区别](https://www.cnblogs.com/fengda/p/10348607.html)

* segment是什么？
* 创建doc的过程

协调节点hash取模`shard = hash(document_id) % (num_of_primary_shards)`，确定在哪个分片，分片所在节点写入Memory Buffer，默认1秒refresh一次到Filesystem Cache，写入translog，flush到磁盘（30分钟一次，或translog大于512M）。flush之后新的translog被创建老的删除，MemoryBuffer写入新segment然后清空，写入新的提交点。

* 删除/更新doc的过程

ES中的doc是不可变的，删除是在磁盘上segment对应的.del文件中做标记，更新一样，只是把老的version的doc标记删除，查询的时候标记删除的记录依然可以匹配查询，但是会被过滤掉。在

* 搜索的过程
* ES是如何做master选举的？集群脑裂怎么解决？

ZenDiscovery模块，nodeId排序，达到n/2+1

通过最少master候选节点配置来解决脑裂问题，集群最少3个节点

```json
PUT /_cluster/settings
{
    "persistent" : {
        "discovery.zen.minimum_master_nodes" : 2
    }
}

```

* 如何保证并发读写一致？

乐观锁版本号

对于读操作，可以设置replication为sync(默认)，这使得操作在主分片和副本分片都完成后才会返回；如果设置replication为async时，也可以通过设置搜索请求参数_preference为primary来查询主分片，确保文档是最新版本。

## 参考

[ElasticSearch权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)

[ElasticSearch-IK拓展自定义词库（2）：HTTP请求动态热词内容方式](https://my.oschina.net/jsonyang/blog/1782832)

[ES常见问题](https://www.cnblogs.com/heqiyoujing/p/11146178.html)

[ElasticSearch是什么？](https://yuzhouwan.com/posts/22654/)
