# Elasticsearch 商品搜索实践

Elasticsearch 在电商系统中主要用于商品搜索、订单搜索、日志分析等场景。它提供了强大的全文检索、聚合分析和实时搜索能力，是电商系统不可或缺的组件。

## 应用场景

### 1. 商品搜索

商品搜索是 Elasticsearch 在电商系统中最核心的应用场景，需要支持：
- 全文搜索（商品名称、描述）
- 精确匹配（商品编号、分类）
- 范围查询（价格区间、上架时间）
- 多条件组合查询
- 分面搜索（Faceted Search）
- 搜索建议（自动补全）
- 相关性排序

### 2. 订单搜索

支持用户和管理员快速检索订单：
- 订单号搜索
- 收货人搜索
- 时间范围搜索
- 订单状态过滤

### 3. 搜索推荐

- 热门搜索词
- 相关搜索
- 搜索纠错
- 个性化推荐

## 索引设计

### 商品索引结构

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "ik_max_word_pinyin": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["pinyin_filter", "word_delimiter"]
        },
        "ik_smart_pinyin": {
          "type": "custom",
          "tokenizer": "ik_smart",
          "filter": ["pinyin_filter"]
        }
      },
      "filter": {
        "pinyin_filter": {
          "type": "pinyin",
          "keep_first_letter": true,
          "keep_separate_first_letter": false,
          "keep_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "lowercase": true
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "product_id": {
        "type": "long"
      },
      "product_no": {
        "type": "keyword"
      },
      "product_name": {
        "type": "text",
        "analyzer": "ik_max_word_pinyin",
        "search_analyzer": "ik_smart_pinyin",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "category_id": {
        "type": "long"
      },
      "category_name": {
        "type": "keyword"
      },
      "category_path": {
        "type": "keyword"
      },
      "brand": {
        "type": "keyword"
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "original_price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "stock": {
        "type": "integer"
      },
      "sales": {
        "type": "integer"
      },
      "description": {
        "type": "text",
        "analyzer": "ik_max_word"
      },
      "main_image": {
        "type": "keyword",
        "index": false
      },
      "images": {
        "type": "keyword",
        "index": false
      },
      "attributes": {
        "type": "nested",
        "properties": {
          "name": {
            "type": "keyword"
          },
          "value": {
            "type": "keyword"
          }
        }
      },
      "tags": {
        "type": "keyword"
      },
      "status": {
        "type": "byte"
      },
      "is_hot": {
        "type": "boolean"
      },
      "is_new": {
        "type": "boolean"
      },
      "is_recommend": {
        "type": "boolean"
      },
      "created_at": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "updated_at": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "suggest": {
        "type": "completion",
        "analyzer": "ik_max_word_pinyin"
      }
    }
  }
}
```

### 创建索引

```bash
PUT /products
{
  "settings": {...},
  "mappings": {...}
}
```

## 数据同步

### 全量同步

#### 从 MySQL 同步到 Elasticsearch

```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk
import pymysql

es = Elasticsearch(['localhost:9200'])

# 连接 MySQL
conn = pymysql.connect(host='localhost', user='root', password='', database='ecommerce')
cursor = conn.cursor(pymysql.cursors.DictCursor)

# 查询商品数据
cursor.execute("SELECT * FROM products WHERE status = 1")
products = cursor.fetchall()

# 构建批量索引操作
actions = []
for product in products:
    action = {
        "_index": "products",
        "_id": product['product_id'],
        "_source": {
            "product_id": product['product_id'],
            "product_no": product['product_no'],
            "product_name": product['product_name'],
            "category_id": product['category_id'],
            "brand": product['brand'],
            "price": float(product['price']),
            "stock": product['stock'],
            "sales": product['sales'],
            "description": product['description'],
            "main_image": product['main_image'],
            "status": product['status'],
            "created_at": product['created_at'].strftime('%Y-%m-%d %H:%M:%S'),
            "suggest": {
                "input": [product['product_name'], product['brand']]
            }
        }
    }
    actions.append(action)

# 批量索引
success, failed = bulk(es, actions)
print(f"成功: {success}, 失败: {failed}")
```

### 增量同步

#### 使用 Canal 监听 MySQL Binlog

```python
# 监听 MySQL binlog 实时同步数据到 ES
from canal.client import Client
from elasticsearch import Elasticsearch

client = Client()
client.connect(host='127.0.0.1', port=11111)
client.subscribe(client_id='1001', destination='example', filter='ecommerce.products')

es = Elasticsearch(['localhost:9200'])

while True:
    message = client.get()
    entries = message['entries']
    
    for entry in entries:
        event_type = entry.header.eventType
        
        if event_type in ['INSERT', 'UPDATE']:
            # 更新或插入文档
            product_data = parse_entry(entry)
            es.index(index='products', id=product_data['product_id'], body=product_data)
        
        elif event_type == 'DELETE':
            # 删除文档
            product_id = entry.rowData[0].afterColumns[0].value
            es.delete(index='products', id=product_id)
    
    client.ack(message['id'])
```

#### 使用 Logstash 同步

```conf
# logstash-mysql-es.conf
input {
  jdbc {
    jdbc_driver_library => "/path/to/mysql-connector-java.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/ecommerce"
    jdbc_user => "root"
    jdbc_password => "password"
    schedule => "*/5 * * * *"  # 每5分钟同步一次
    statement => "SELECT * FROM products WHERE updated_at > :sql_last_value ORDER BY updated_at"
    use_column_value => true
    tracking_column => "updated_at"
    tracking_column_type => "timestamp"
  }
}

filter {
  mutate {
    remove_field => ["@version", "@timestamp"]
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "products"
    document_id => "%{product_id}"
  }
}
```

## 搜索查询

### 基础搜索

#### 全文搜索
```json
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "苹果手机",
      "fields": ["product_name^3", "description", "brand^2"],
      "type": "best_fields",
      "operator": "and"
    }
  }
}
```

#### 精确匹配
```json
GET /products/_search
{
  "query": {
    "term": {
      "product_no": "PROD20231207001"
    }
  }
}
```

#### 范围查询
```json
GET /products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 1000,
        "lte": 5000
      }
    }
  }
}
```

### 组合查询

#### Bool 查询（多条件组合）
```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "product_name": "手机"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "status": 1
          }
        },
        {
          "range": {
            "price": {
              "gte": 2000,
              "lte": 8000
            }
          }
        },
        {
          "terms": {
            "brand": ["Apple", "Samsung", "Huawei"]
          }
        }
      ],
      "should": [
        {
          "term": {
            "is_hot": true
          }
        },
        {
          "term": {
            "is_recommend": true
          }
        }
      ],
      "minimum_should_match": 1
    }
  },
  "from": 0,
  "size": 20,
  "sort": [
    {
      "sales": {
        "order": "desc"
      }
    },
    {
      "_score": {
        "order": "desc"
      }
    }
  ]
}
```

### 嵌套查询（属性筛选）

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "nested": {
            "path": "attributes",
            "query": {
              "bool": {
                "must": [
                  {
                    "term": {
                      "attributes.name": "颜色"
                    }
                  },
                  {
                    "term": {
                      "attributes.value": "黑色"
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}
```

### 聚合查询（分面搜索）

#### 价格区间聚合
```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 1000 },
          { "from": 1000, "to": 3000 },
          { "from": 3000, "to": 5000 },
          { "from": 5000 }
        ]
      }
    }
  }
}
```

#### 品牌聚合
```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "brands": {
      "terms": {
        "field": "brand",
        "size": 20
      }
    }
  }
}
```

#### 分类聚合
```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": {
        "field": "category_name",
        "size": 10
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

### 搜索建议（自动补全）

```json
GET /products/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "苹果",
      "completion": {
        "field": "suggest",
        "size": 10,
        "skip_duplicates": true
      }
    }
  }
}
```

### 高亮显示

```json
GET /products/_search
{
  "query": {
    "match": {
      "product_name": "苹果手机"
    }
  },
  "highlight": {
    "fields": {
      "product_name": {
        "pre_tags": ["<em class='highlight'>"],
        "post_tags": ["</em>"]
      },
      "description": {
        "fragment_size": 150,
        "number_of_fragments": 3
      }
    }
  }
}
```

## 排序策略

### 综合排序（相关性 + 销量 + 价格）

```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query": "手机",
          "fields": ["product_name", "description"]
        }
      },
      "functions": [
        {
          "field_value_factor": {
            "field": "sales",
            "modifier": "log1p",
            "factor": 0.1
          }
        },
        {
          "gauss": {
            "price": {
              "origin": "3000",
              "scale": "2000"
            }
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

## 性能优化

### 1. 索引优化

```json
PUT /products/_settings
{
  "index": {
    "refresh_interval": "30s",  // 降低刷新频率
    "number_of_replicas": 1,    // 合理设置副本数
    "translog": {
      "durability": "async",    // 异步刷盘
      "sync_interval": "30s"
    }
  }
}
```

### 2. 查询优化

- 使用 filter 代替 query（filter 有缓存）
- 避免深度分页，使用 search_after
- 合理使用 _source 过滤
- 控制返回字段数量

### 3. 批量操作

```python
from elasticsearch.helpers import bulk

# 批量索引
actions = [
    {
        "_index": "products",
        "_id": i,
        "_source": {...}
    }
    for i in range(1000)
]

bulk(es, actions)
```

### 4. 路由优化

```json
# 按分类路由，同一分类的数据在同一分片
PUT /products/_doc/1?routing=category_100
{
  "product_name": "iPhone 15",
  "category_id": 100
}
```

## 监控与运维

### 查看集群健康状态
```bash
GET /_cluster/health
```

### 查看索引统计信息
```bash
GET /products/_stats
```

### 查看慢查询日志
```json
PUT /products/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.query.info": "5s",
  "index.search.slowlog.threshold.query.debug": "2s"
}
```

## 最佳实践

1. **合理设计索引结构**：根据查询需求设计字段类型
2. **使用别名**：便于索引重建和零停机迁移
3. **定期清理过期数据**：避免索引过大
4. **监控集群状态**：及时发现和解决问题
5. **做好容量规划**：预估数据量和查询QPS
6. **使用模板管理索引**：统一索引配置
7. **分片策略**：避免分片过多或过少

## 参考资料

- [Elasticsearch官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [ElasticSearch实践](../../05~搜索引擎/ElasticSearch/README.md)
