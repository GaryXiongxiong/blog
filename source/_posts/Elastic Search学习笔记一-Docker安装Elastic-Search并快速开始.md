---
title: Elastic Search学习笔记一 - Docker安装 Elastic Search 并快速开始
date: 2020-09-08 16:59:34
tags:
- Elastic Search
categories: 
- blog
---
## Docker 安装 Elastic Search

#### 编写`Docker-compose.yml`

```yaml
version: '3'
services:
  es:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.9.0
    container_name: es
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms256m -Xmx512m
  kibana:
    image: kibana:7.9.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://es:9200
    depends_on:
      - es
```

#### 运行docker-compose

```bash
docker-compose up -d
```

#### 验证Kibana与Elastic Search

1. 前往 docker宿主机地址:5601，打开Kibana环境

2. 点击左下 Dev Tools

3. 运行`GET _cluster/health`

4. 确认ES健康信息

   ```json
   {
     "cluster_name" : "docker-cluster",
     "status" : "green",
     "timed_out" : false,
     "number_of_nodes" : 1,
     "number_of_data_nodes" : 1,
     "active_primary_shards" : 6,
     "active_shards" : 6,
     "relocating_shards" : 0,
     "initializing_shards" : 0,
     "unassigned_shards" : 0,
     "delayed_unassigned_shards" : 0,
     "number_of_pending_tasks" : 0,
     "number_of_in_flight_fetch" : 0,
     "task_max_waiting_in_queue_millis" : 0,
     "active_shards_percent_as_number" : 100.0
   }
   ```

## 基本Query

#### 插入/更新文档

```json
PUT /megacorp/_doc/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}

PUT /megacorp/_doc/2
{
    "first_name" :  "Jane",
    "last_name" :   "Smith",
    "age" :         32,
    "about" :       "I like to collect rock albums",
    "interests":  [ "music" ]
}

PUT /megacorp/_doc/3
{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         35,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}

PUT /megacorp/_doc/4
{
    "first_name" :  "Gary",
    "last_name" :   "Jiang",
    "age" :         26,
    "about":        "I like Coding",
    "interests":  [ "coding" ]
}
```

#### 根据id获取文档

```json
GET /megacorp/_doc/1
```

```json
{
  "_index" : "megacorp",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 6,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "first_name" : "John",
    "last_name" : "Smith",
    "age" : 25,
    "about" : "I love to go rock climbing",
    "interests" : [
      "sports",
      "music"
    ]
  }
}
```

#### 轻量搜索

```json
GET /megacorp/_search?q=last_name:Smith
```

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.44183272,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.44183272,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 0.44183272,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      }
    ]
  }
}
```

#### 查询表达式搜索

```json
GET /megacorp/_search
{
  "query": {
    "match": {
      "last_name": "Smith"
    }
  }
}
```

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.44183272,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.44183272,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 0.44183272,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      }
    ]
  }
}
```

#### 过滤器

```json
GET /megacorp/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "last_name": "Smith"
          }
        }
      ],
      "filter": [
        {
          "range": {
            "age": {
              "gt": 30
            }
          }
        }
      ]
    }
  }
}
```

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.44183272,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.44183272,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      }
    ]
  }
}
```

#### 全文搜索

```json
GET /megacorp/_search
{
  "query": {
    "match": {
      "about": "rock climbing"
    }
  }
}
```

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.0797646,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 1.0797646,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.42033815,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      }
    ]
  }
}
```

#### 短语搜索

```json
GET /megacorp/_search
{
  "query": {
    "match_phrase": {
      "about": "rock climbing"
    }
  }
}
```

```json
{
  "took" : 37,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0797647,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 1.0797647,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      }
    ]
  }
}
```

