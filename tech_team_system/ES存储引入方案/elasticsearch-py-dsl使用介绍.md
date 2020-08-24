## 1. elasticsearch-dsl-py

[elasticsearch-dsl-py](https://github.com/elastic/elasticsearch-dsl-py) 是由 elastic 官方提供的，是对 [elasticsearch-py](https://github.com/elastic/elasticsearch-py) 更高级的封装。

**示例**

1. 使用 elasticsearch-py 查询数据

```shell
from elasticsearch import Elasticsearch

# 初始化客户端
client = Elasticsearch()

response = client.search(
    index="my-index",  # 查询的索引名称
    body={
      "query": {
        "bool": {   # 使用 bool 查询
          "must": [{"match": {"title": "python"}}],   # title包含"python"
          "must_not": [{"match": {"description": "beta"}}],   # description不包含"beta"
          "filter": [{"term": {"category": "search"}}]  # 过滤category为"search"的记录
        }
      },
      "aggs" : {
        # 对 tags 字段进行聚合，获取每类 tag 中 lines 字段的最大值
        "per_tag": {
          "terms": {"field": "tags"},
          "aggs": {
            "max_lines": {"max": {"field": "lines"}}
          }
        }
      }
    }
)

# 取结果
for hit in response['hits']['hits']:
    print(hit['_score'], hit['_source']['title'])

for tag in response['aggregations']['per_tag']['buckets']:
    print(tag['key'], tag['max_lines']['value'])
```

- 优点
  - 直接使用原生语法，简单粗暴

- 缺点
  - body 结构复杂，嵌套层级深，编写时容易出错，且后续修改繁琐。等价于徒手写SQL



2. 使用 elasticsearch-py-dsl 查询数据

```python
from elasticsearch import Elasticsearch
from elasticsearch_dsl import Search

client = Elasticsearch()

s = Search(using=client, index="my-index") \  # 指定使用的client，以及索引名称
    .filter("term", category="search") \  # 过滤category为"search"的记录
    .query("match", title="python")   \   # title包含"python"
    .exclude("match", description="beta")  # description不包含"beta"

# 聚合
s.aggs.bucket('per_tag', 'terms', field='tags') \
    .metric('max_lines', 'max', field='lines')

# 支持聚合多个字段
# s.aggs.bucket('per_tag', 'terms', field='tags')
#  .bucket('per_category', 'terms', field='category')
#  .metric('max_lines', 'max', field='lines')

# 执行
response = s.execute()

# 取数据
for hit in response:
    print(hit.meta.score, hit.title)

for tag in response.aggregations.per_tag.buckets:
    print(tag.key, tag.max_lines.value)
```

- 优点
  - 使用函数式声明风格生成DSL，符合Django开发者认知
  - 取数据时使用`.xxx` 而不是 `["xxx"]`，写起来更爽



当然，通过elasticsearch-py-dsl，也能仅生成DSL，而不进行查询。直接调用 Search 对象的 `to_dict` 方法即可

```python
In [24]: print(json.dumps(s.to_dict(), indent=2))
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "category": "search"
          }
        },
        {
          "bool": {
            "must_not": [
              {
                "match": {
                  "description": "beta"
                }
              }
            ]
          }
        }
      ],
      "must": [
        {
          "match": {
            "title": "python"
          }
        }
      ]
    }
  },
  "aggs": {
    "per_tag": {
      "terms": {
        "field": "tags"
      },
      "aggs": {
        "per_category": {
          "terms": {
            "field": "category"
          },
          "aggs": {
            "max_lines": {
              "max": {
                "field": "lines"
              }
            }
          }
        }
      }
    }
  }
}
```





3. 索引初始化

```python
from datetime import datetime
from elasticsearch_dsl import Document, Date, Integer, Keyword, Text, connections

# 初始化 es 客户端
connections.create_connection(hosts=['localhost'])

# 定义数据的 Schema
class Article(Document):
    title = Text()
    body = Text()
    tags = Keyword()
    published_from = Date()
    lines = Integer()

    class Index:
        name = 'blog'   # 索引名称
        settings = {
          "number_of_shards": 2,  # 分片数量
        }

    def save(self, ** kwargs):
        self.lines = len(self.body.split())
        return super(Article, self).save(** kwargs)

    def is_published(self):
        return datetime.now() > self.published_from

# 根据定义的Schema，为es初始化索引和mapping
Article.init()
```

可以看到，已经在ES创建了对应的索引和mapping


```json
// curl localhost:9200/blog?pretty

{
  "blog" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "body" : {
          "type" : "text"
        },
        "lines" : {
          "type" : "integer"
        },
        "published_from" : {
          "type" : "date"
        },
        "tags" : {
          "type" : "keyword"
        },
        "title" : {
          "type" : "text"
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1592710536924",
        "number_of_shards" : "2",
        "number_of_replicas" : "1",
        "uuid" : "62mmerFpTPivO4kJxqN0yg",
        "version" : {
          "created" : "7060299"
        },
        "provided_name" : "blog"
      }
    }
  }
}
```



4. 数据持久化

```python
# 创建文档
article = Article(
    meta={'id': 42},
    title='Hello world!',
    tags=['test'],
    body=''' looong text ''',
    published_from=datetime.now(),
)
# 保存文档
article.save()

# 根据ID查询文档
article = Article.get(id=42)
print(article.is_published())

# 根据条件查询文档
articles = Article.search().query("match", title="hello").execute()
for article in articles:
    print(article.title)
```

数据插入成功，可以通过Rest API查询到

```json
// curl localhost:9200/blog/_search?pretty

{
  "took" : 1096,
  "timed_out" : false,
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "blog",
        "_type" : "_doc",
        "_id" : "42",
        "_score" : 1.0,
        "_source" : {
          "title" : "Hello world!",
          "tags" : [
            "test"
          ],
          "body" : " looong text ",
          "published_from" : "2020-06-21T11:45:50.424401",
          "lines" : 2
        }
      }
    ]
  }
}
```



从上面可以看出，通过Python Class 定义 Schema，并基于这个Class进行数据持久化和数据查询。一切都是那么的熟悉。



## 2. Django Elasticsearch DSL

[Django Elasticsearch DSL](https://github.com/django-es/django-elasticsearch-dsl) 是 elasticsearch-dsl-py 的一个轻度封装，以更好地适配 Django 工程。

使用介绍

1. **首先要定义一个Django Model，这个就是我们平时用来操作MySQL的ORM**

   ```python
   # models.py
   
   from django.db import models
   
   
   # Create your models here.
   class AnomalyRecord(models.Model):
       """
       异常记录表
   
       1. 经过检测算法判定后，生成异常记录
       2. 其它系统接入的异常记录
       """
   
       anomaly_id = models.CharField(verbose_name="异常ID", primary_key=True, max_length=64)
       source_time = models.DateTimeField(verbose_name="数据上报时间", db_index=True)
       create_time = models.DateTimeField(verbose_name="记录创建时间", auto_now_add=True, db_index=True)
       strategy_id = models.IntegerField(verbose_name="关联策略ID", db_index=True)
       origin_alarm = models.TextField(verbose_name="原始的异常内容(JSON)")
       count = models.IntegerField(verbose_name="异常汇总数量", default=1)
       event_id = models.CharField(verbose_name="关联事件ID", max_length=64, blank=True, default="", db_index=True)
   
   ```

   

2. **新建文件 `documents.py`，并关联 `AnomalyRecord`** 

```python
# -*- coding: utf-8 -*-
import json

from django_elasticsearch_dsl import Document, fields
from django_elasticsearch_dsl.registries import registry
from .models import AnomalyRecord


@registry.register_document
class AnomalyRecordDocument(Document):

    class Index:
        # ES 索引名称
        name = 'anomaly_record'
        # ES 索引的配置
        settings = {'number_of_shards': 1,
                    'number_of_replicas': 0}

    class Django:
        # 关联的 Django Model 类，会根据 Model 的字段自动推导出 ES 的 Mapping 配置
        model = AnomalyRecord

        # 希望保存到 ES 的字段
        fields = [
            'anomaly_id',
            'source_time',
            'create_time',
            'strategy_id',
            # 'origin_alarm',
            'count',
            'event_id',
        ]

        # 是否忽略 Model 的保存和更新操作，默认为 False
        # ignore_signals = False

        # 每次保存和更新时，是否对索引进行 refresh 操作，默认是 True
        # auto_refresh = True

    # 可以根据需要，重写某些字段的类型，而不使用自动推导
    origin_alarm = fields.ObjectField()

    def prepare_origin_alarm(self, instance):
        """
        prepare_xxx
        在某个字段被保存到 ES 之前，可以做一些前置工作，比如将 json 字符串进行反序列化
        """
        return json.loads(instance.origin_alarm)

```



3. **索引初始化**

   执行 django 命令对索引进行初始化

   ```
   python manage.py search_index --rebuild
   ```
   
   查询 ES，可以看到对应的索引和mapping都已经初始化好
   
   ```json
   // curl "localhost:9200/anomaly_record?pretty"
   {
     "anomaly_record" : {
       "aliases" : { },
       "mappings" : {
         "properties" : {
           "anomaly_id" : {
             "type" : "text"
           },
           "count" : {
             "type" : "integer"
           },
           "create_time" : {
             "type" : "date"
           },
           "event_id" : {
             "type" : "text"
           },
           "origin_alarm" : {
             "type" : "object"
           },
           "source_time" : {
             "type" : "date"
           },
           "strategy_id" : {
             "type" : "integer"
           }
         }
       },
       "settings" : {
         "index" : {
           "creation_date" : "1592738680534",
           "number_of_shards" : "1",
           "number_of_replicas" : "0",
           "uuid" : "QXNLLjybQyWTSAz0ejPPOw",
           "version" : {
             "created" : "7060299"
           },
           "provided_name" : "anomaly_record"
         }
       }
     }
   }
   ```
   
   
   
   
   
4. **数据持久化**

   Django Elasticsearch DSL 监听了 Django DB 的 signal，直接对 Model 对数据进行保存或更新，都会触发 ES  Document 的插入和更新。

   ```python
   import datetime
   from home.models import AnomalyRecord
   
   AnomalyRecord.objects.create(
       anomaly_id='787c0f67e3743291fb8f97dc99df6e28.1592628763.341.340.2',
       strategy_id=341,
       event_id='787c0f67e3743291fb8f97dc99df6e28.1592628643.341.340.2',
       source_time=datetime.datetime(2020, 6, 20, 4, 5, 43),
       origin_alarm='{"data": {"time": 1592628763, "value": "", "values": {"time": 1592628763, "value": ""}, "dimensions": {"bk_target_ip": "20.1.1.76", "bk_target_cloud_id": 0, "bk_topo_node": ["biz|5", "module|65", "set|15"]}, "record_id": "787c0f67e3743291fb8f97dc99df6e28.1592628763"}, "anomaly": {"2": {"anomaly_id": "787c0f67e3743291fb8f97dc99df6e28.1592628763.341.340.2", "anomaly_time": 1592628775, "anomaly_message": "Ping\\u4e0d\\u53ef\\u8fbe"}}}'
   )
   ```

   请注意，只要 Model 关联了 Document 类，任何的数据写入操作，都是**双写**的（即同时写入 MySQL 和 ES）

   此时，ES也能查到对应的数据了

   ```json
   // curl "localhost:9200/anomaly_record/_search?pretty"
   
   {
     "took" : 1,
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
       "max_score" : 1.0,
       "hits" : [
         {
           "_index" : "anomaly_record",
           "_type" : "_doc",
           "_id" : "787c0f67e3743291fb8f97dc99df6e28.1592628763.341.340.2",
           "_score" : 1.0,
           "_source" : {
             "origin_alarm" : {
               "data" : {
                 "time" : 1592628763,
                 "value" : "",
                 "values" : {
                   "time" : 1592628763,
                   "value" : ""
                 },
                 "dimensions" : {
                   "bk_target_ip" : "20.1.1.76",
                   "bk_target_cloud_id" : 0,
                   "bk_topo_node" : [
                     "biz|5",
                     "module|65",
                     "set|15"
                   ]
                 },
                 "record_id" : "787c0f67e3743291fb8f97dc99df6e28.1592628763"
               },
               "anomaly" : {
                 "2" : {
                   "anomaly_id" : "787c0f67e3743291fb8f97dc99df6e28.1592628763.341.340.2",
                   "anomaly_time" : 1592628775,
                   "anomaly_message" : "Ping不可达"
                 }
               }
             },
             "anomaly_id" : "787c0f67e3743291fb8f97dc99df6e28.1592628763.341.340.2",
             "source_time" : "2020-06-20T04:05:43",
             "create_time" : "2020-06-21T11:16:26.262976+00:00",
             "strategy_id" : 341,
             "count" : 1,
             "event_id" : "787c0f67e3743291fb8f97dc99df6e28.1592628643.341.340.2"
           }
         }
       ]
     }
   }
   ```

   

   从上面的例子可以看出，只需要加一个 `documents.py`，对所需的Model进行关联。我们甚至不用改原有的代码，即可将所需的数据存入ES。对于老版本迁移来说还是十分方便的。

   如果你不希望写入MySQL，可以直接使用 `AnomalyRecordDocument` 进行操作

   ```python
   from home.documents import AnomalyRecordDocument
   
   doc = AnomalyRecordDocument(
       meta={'id': '787c0f67e3743291fb8f97dc99df6e28.1592628763.341.340.2'},
       anomaly_id='787c0f67e3743291fb8f97dc99df6e28.1592628763.341.340.2',
       strategy_id=341,
       event_id='787c0f67e3743291fb8f97dc99df6e28.1592628643.341.340.2',
       source_time=datetime.datetime(2020, 6, 20, 4, 5, 43),
       origin_alarm=json.loads('{"data": {"time": 1592628763, "value": "", "values": {"time": 1592628763, "value": ""}, "dimensions": {"bk_target_ip": "20.1.1.76", "bk_target_cloud_id": 0, "bk_topo_node": ["biz|5", "module|65", "set|15"]}, "record_id": "787c0f67e3743291fb8f97dc99df6e28.1592628763"}, "anomaly": {"2": {"anomaly_id": "787c0f67e3743291fb8f97dc99df6e28.1592628763.341.340.2", "anomaly_time": 1592628775, "anomaly_message": "Ping\\u4e0d\\u53ef\\u8fbe"}}}')
   )
   
   doc.save()
   ```

   

   

5. **数据检索**

   分两种情况

   - 查询 MySQL 数据
     
   - 直接使用 Model，与之前的使用方法并无差异
     
   - 查询 ES 数据，使用 Document 类，例如

     ```python
     # 按ID查询
     record = AnomalyRecordDocument.get("787c0f67e3743291fb8f97dc99df6e28.1592628763.341.340.2")
     
     # 按其他条件查询
     hits = AnomalyRecordDocument.search().query("match", strategy_id=341)
     for h in hits:
         print(h.anomaly_id)
     ```



6. **索引拆分**

   若将数据都存储到同一个索引，随着时间的推移，单个索引的数据将会越来越大，会严重影响索引的查询和写入性能，后续也会难以维护。因此一般的实践是按照某种规则对索引进行拆分，避免单个索引数据量过大的情况。

   Elasticsearch-dsl-py 支持带有通配符的索引名称设置，通过重写 save 方法来决定数据写入的真实索引名称

   ```python
   # -*- coding: utf-8 -*-
   import json
   import datetime
   
   from django_elasticsearch_dsl import Document, fields, Index
   from django_elasticsearch_dsl.registries import registry
   from .models import AnomalyRecord
   
   
   @registry.register_document
   class AnomalyRecordDocument(Document):
   
       class Index:
           # ES 索引名称，此处使用了通配符，用于搜索
           name = 'anomaly_record-*'
           # ES 索引的配置
           settings = {'number_of_shards': 1,
                       'number_of_replicas': 0}
   
       class Django:
           # 关联的 Django Model 类，会根据 Model 的字段自动推导出 ES 的 Mapping 配置
           model = AnomalyRecord
   
           # 希望保存到 ES 的字段
           fields = [
               'anomaly_id',
               'source_time',
               'create_time',
               'strategy_id',
               # 'origin_alarm',
               'count',
               'event_id',
           ]
   
           # 是否忽略 Model 的保存和更新操作，默认为 False
           # ignore_signals = False
   
           # 每次保存和更新时，是否对索引进行 refresh 操作，默认是 True
           # auto_refresh = True
   
       # 可以根据需要，重写某些字段的类型，而不使用自动推导
       origin_alarm = fields.ObjectField()
   
       def prepare_origin_alarm(self, instance):
           """
           prepare_xxx
           在某个字段被保存到 ES 之前，可以做一些前置工作，比如将 json 字符串进行反序列化
           """
           return json.loads(instance.origin_alarm)
   
       def save(self, **kwargs):
           # assign now if no timestamp given
           if not self.create_time:
               self.create_time = datetime.datetime.now()
   
           # 将 * 替换为实际的日期，拼接索引名称
           kwargs['index'] = self.generate_index_name(self.create_time)
           return super().save(**kwargs)
   
       @classmethod
       def generate_index_name(cls, datetime_obj):
           """
           根据时间生成对应的真实索引名称
           """
           return cls.Index.name.replace("*", datetime_obj.strftime('%Y%m%d'))
   
       @classmethod
       def init_index_template(cls):
           """
           创建或更新 Index Template
           """
           index_template = cls._index.as_template("anomaly_record")
           return index_template.save()
   
       @classmethod
       def delete_expired_indices(cls, expire_days=30):
           """
           删除过期索引
           :param expire_days: 过期天数
           """
   
           # 生成需要保留的最老的一条索引的名称
           first_date = datetime.datetime.now() - datetime.timedelta(days=expire_days)
           oldest_index_name = cls.generate_index_name(first_date)
   
           # 获取索引集中的索引列表
           indices = list(cls._index.get().keys())
   
           # 字符串比对，若索引名称比指定的名称小，则说明时间更老，需要被删除
           indices_to_be_deleted = [index for index in indices if index < oldest_index_name]
   
           delete_results = dict.fromkeys(indices_to_be_deleted)
   
           for index in indices_to_be_deleted:
               # 遍历删除过期索引
               result = Index(name=index).delete(ignore_unavailable=True)
               delete_results[index] = result
   
           return delete_results
   
   ```

   

   同时，需要新建一个migrations文件，在DB迁移时初始化 IndexTemplate

   ```python
   from django.db import migrations
   
   from home.documents import AnomalyRecordDocument
   
   
   def init_index_template(apps, schema_editor):
       """
       创建或更新 Index Template
       """
       return AnomalyRecordDocument.init_index_template()
   
   
   class Migration(migrations.Migration):
       dependencies = [
           ('home', '0001_initial'),
       ]
   
       operations = [
           migrations.RunPython(init_index_template),
       ]
   
   ```

   

   

   执行 migrate 之后，可以看到对应的模板配置。今后创建的索引，只要匹配 `anomaly_record-*` ，都会应用该模板的配置。

   ```javascript
   // curl "localhost:9200/_template/anomaly_record?pretty"
   
   {
     "anomaly_record" : {
       "order" : 0,
       "index_patterns" : [
         "anomaly_record-*"
       ],
       "settings" : {
         "index" : {
           "number_of_shards" : "1",
           "number_of_replicas" : "0"
         }
       },
       "mappings" : {
         "properties" : {
           "anomaly_id" : {
             "type" : "text"
           },
           "event_id" : {
             "type" : "text"
           },
           "create_time" : {
             "type" : "date"
           },
           "source_time" : {
             "type" : "date"
           },
           "strategy_id" : {
             "type" : "integer"
           },
           "count" : {
             "type" : "integer"
           },
           "origin_alarm" : {
             "type" : "object"
           }
         }
       },
       "aliases" : { }
     }
   }
   ```

   

   插入新的数据之后再查询，发现数据已经落在对应日期的索引中了

   ```python
   In [25]: AnomalyRecordDocument.search().query("match", strategy_id=341).execute()
   Out[25]: <Response: [AnomalyRecordDocument(index='anomaly_record-20200718', id='787c0f67e3743291fb8f97dc99df6e28.1592628763.341.340.2')]>
   ```



7. **小结**
- 基于elasticsearch dsl py，很好地支持各种个性化场景
   - 通过Django的Signal机制保持与Elasticsearch的数据同步
   - 提供了创建、删除、重建和填充索引的管理命令
   - 根据Django模型字段自动推导索引的Mapping
   - 通过重写 save 方法，可将数据按照一定规则拆分为多个索引，便于后续的维护



## 3. Django Elasticsearch DSL DRF

[django-elasticsearch-dsl-drf](https://django-elasticsearch-dsl-drf.readthedocs.io/en/latest/#id1) 整合了 [Elasticsearch DSL](https://pypi.python.org/pypi/elasticsearch-dsl) 和 [Django REST framework](https://pypi.python.org/pypi/djangorestframework) ，用于快速开发基于 ES 索引查询的 REST API

### 3.1 生成Rest API

- models.py

```python
# models.py

class Event(models.Model):
    """
    事件

    经过检测范围匹配，收敛等判断后，生成事件
    """

    EVENT_LEVEL_FATAL = 1
    EVENT_LEVEL_WARNING = 2
    EVENT_LEVEL_REMIND = 3

    EVENT_LEVEL = (
        (EVENT_LEVEL_FATAL, "致命"),
        (EVENT_LEVEL_WARNING, "预警"),
        (EVENT_LEVEL_REMIND, "提醒"),
    )

    DEFAULT_END_TIME = datetime.datetime(1980, 1, 1, 8, tzinfo=pytz.UTC)

    create_time = models.DateTimeField(verbose_name="创建时间", auto_now_add=True)
    event_id = models.CharField(verbose_name="事件ID", db_index=True, unique=True, max_length=255)
    begin_time = models.DateTimeField(verbose_name="事件产生时间")
    end_time = models.DateTimeField(verbose_name="事件结束时间", default=DEFAULT_END_TIME)
    bk_biz_id = models.IntegerField(verbose_name="业务ID", default=0, blank=True)
    strategy_id = models.IntegerField(verbose_name="关联策略ID")
    origin_alarm = models.TextField(verbose_name="原始的异常内容", default=None)
    origin_config = models.TextField(verbose_name="告警策略原始配置", default=None)
    level = models.IntegerField(verbose_name="级别", choices=EVENT_LEVEL, default=0)
    status = models.IntegerField(verbose_name="状态", default=30)
    is_ack = models.BooleanField(verbose_name="是否确认", default=False)
    p_event_id = models.CharField(verbose_name="父事件ID", default="", blank=True, max_length=255)
    is_shielded = models.BooleanField(verbose_name="是否处于屏蔽状态", default=False)
    shield_type = models.CharField(verbose_name="屏蔽类型", default="", blank=True, max_length=16)

    target_key = models.CharField(verbose_name="目标标识符", default="", blank=True, max_length=128)
    notify_status = models.IntegerField(verbose_name="通知状态", default=0)

    class Meta:
        verbose_name = "事件"
        verbose_name_plural = "事件"
        db_table = "alarm_event"
        index_together = (
            ("level", "end_time", "bk_biz_id", "status", "strategy_id", "notify_status"),
            ("end_time", "bk_biz_id", "status", "strategy_id", "notify_status"),
            ("target_key", "status", "end_time", "bk_biz_id", "level", "strategy_id"),
            ("notify_status", "status", "end_time", "bk_biz_id", "level", "strategy_id"),
        )

```



- documents.py

```python
# documents.py

@registry.register_document
class EventDocument(Document):

    class Index:
        # ES 索引名称，此处使用了通配符，用于搜索
        name = 'alarm_event-*'
        # ES 索引的配置
        settings = {'number_of_shards': 1,
                    'number_of_replicas': 0}

    class Django:
        # 关联的 Django Model 类，会根据 Model 的字段自动推导出 ES 的 Mapping 配置
        model = Event

        # 希望保存到 ES 的字段
        fields = [
            "create_time",
            "event_id",
            "begin_time",
            "end_time",
            "bk_biz_id",
            "strategy_id",
            "level",
            "status",
            "is_ack",
            "p_event_id",
            "is_shielded",
            "shield_type",
            "target_key",
            "notify_status",
        ]

    # 可以根据需要，重写某些字段的类型，而不使用自动推导
    origin_alarm = fields.ObjectField()
    origin_config = fields.ObjectField()

    def prepare_origin_alarm(self, instance):
        return json.loads(instance.origin_alarm)

    def prepare_origin_config(self, instance):
        return json.loads(instance.origin_config)

    def save(self, **kwargs):
        # assign now if no timestamp given
        if not self.begin_time:
            self.begin_time = datetime.datetime.now()

        # 将 * 替换为实际的日期，拼接索引名称
        kwargs['index'] = self.generate_index_name(self.begin_time)
        return super(EventDocument, self).save(**kwargs)

    @classmethod
    def generate_index_name(cls, datetime_obj):
        """
        根据时间生成对应的真实索引名称
        """
        return cls.Index.name.replace("*", datetime_obj.strftime('%Y%m'))

    @classmethod
    def init_index_template(cls):
        """
        创建或更新 Index Template
        """
        index_template = cls._index.as_template("alarm_event")
        return index_template.save()
```



- serializers.py

```python
# -*- coding: utf-8 -*-

from django_elasticsearch_dsl_drf.serializers import DocumentSerializer

from .documents import EventDocument


class EventDocumentSerializer(DocumentSerializer):

    class Meta(object):
        document = EventDocument
```



- views.py

```python
from django_elasticsearch_dsl_drf.viewsets import DocumentViewSet

from .documents import EventDocument
from .serializers import EventDocumentSerializer


class EventDocumentViewSet(DocumentViewSet):

    document = EventDocument
    serializer_class = EventDocumentSerializer

```



- urls.py

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from home.views import EventDocumentViewSet

router = DefaultRouter()
router.register(r'event', EventDocumentViewSet, basename='event_document')


urlpatterns = [
    path('', include(router.urls)),
]

```



好了，配置完成。跟 Django Model 的配置非常类似，接下来尝试访问接口

```
curl "http://localhost:8000/event/"
```

发现最多只返回了10条数据，因为这是 ES 查询 API 的默认行为 



如果需要查看更多，则需要配置分页

- views.py

```python
# views.py

from django_elasticsearch_dsl_drf.viewsets import DocumentViewSet
from django_elasticsearch_dsl_drf.pagination import PageNumberPagination as BasePageNumberPagination

from .documents import EventDocument
from .serializers import EventDocumentSerializer


class PageNumberPagination(BasePageNumberPagination):

    # 指定分页参数
    page_size_query_param = 'page_size'


class EventDocumentViewSet(DocumentViewSet):

    document = EventDocument
    serializer_class = EventDocumentSerializer
    
    # 指定分页处理类
    pagination_class = PageNumberPagination

```

尝试访问。通过 `page` 和 `page_size` 参数指定分页

```shell
curl "http://localhost:8000/event/?page=10&page_size=20"
```



### 3.2 search - 搜索



- views.py

```python
class EventDocumentViewSet(DocumentViewSet):

    document = EventDocument
    serializer_class = EventDocumentSerializer
    pagination_class = PageNumberPagination

    filter_backends = [
        SearchFilterBackend,   # 增加检索的backend
    ]

    # 定义检索字段
    search_fields = (
        'origin_config.strategy_name',
        'origin_config.item_list.item_name',
    )

```



可通过 `search` 参数，对 `search_fields` 中指定的字段进行搜索

```shell
# 任一字段匹配 "PING" 关键字
curl "http://localhost:8000/event/?search=PING"
```

也可以针对特定字段进行搜索

```shell
# 仅字段 origin_config.item_list.item_name 匹配 "PING" 关键字
curl "http://localhost:8000/event/?search=origin_config.item_list.item_name:PING"

# 多个字段同时匹配
curl "http://localhost:8000/event/?search=origin_config.item_list.item_name:PING&search=origin_config.strategy_name:CPU"
```



支持权重搜索

```python
class EventDocumentViewSet(DocumentViewSet):

    document = EventDocument
    serializer_class = EventDocumentSerializer
    pagination_class = PageNumberPagination

    filter_backends = [
        SearchFilterBackend,
        DefaultOrderingFilterBackend,  # 增加默认排序的backend
    ]

    # 定义检索字段，并配置权重
    search_fields = {
        'origin_config.strategy_name': None,
        'origin_config.item_list.item_name': {'boost': 4},
    }

    # 将 _score 加入默认排序，按匹配得分排序
    ordering = ('_score',)

```



> 参考：https://django-elasticsearch-dsl-drf.readthedocs.io/en/latest/filtering_usage_examples.html#search



### 3.3 filter - 过滤

filter 适用于完全匹配及范围检索

- views.py

```python
class EventDocumentViewSet(DocumentViewSet):

    document = EventDocument
    serializer_class = EventDocumentSerializer
    pagination_class = PageNumberPagination

    filter_backends = [
        SearchFilterBackend,
        DefaultOrderingFilterBackend,
        FilteringFilterBackend,  # 增加过滤的backend
    ]

    search_fields = {
        'origin_config.strategy_name': None,
        'origin_config.item_list.item_name': {'boost': 4},
    }

    ordering = ('_score',)

    # 增加过滤字段配置，以下字段支持过滤
    filter_fields = {
        'strategy_id': None,
        'level': None,
        'begin_time': None,
        'end_time': None,
    }

```



可通过查询参数指定字段进行过滤

```shell
# 单个值过滤
curl "http://localhost:8000/event/?strategy_id=12"

# 多个值过滤
curl "http://localhost:8000/event/?strategy_id=12&strategy_id=13"

# 大小过滤
curl "http://localhost:8000/event/?level__gt=1"

# 范围过滤
curl "http://localhost:8000/event/?begin_time__range=2020-07-04__2020-07-20"
```



> 参考：https://django-elasticsearch-dsl-drf.readthedocs.io/en/latest/filtering_usage_examples.html#filtering



### 3.4 suggest - 搜索建议

在前端交互中，有时候需要根据用户的输入，给出对应的自动补全候选菜单。仅需要通过简单的配置，即可实现搜索建议功能

- documents.py

```python
@registry.register_document
class EventDocument(Document):

    class Index:
        name = 'alarm_event-*'
        settings = {'number_of_shards': 1,
                    'number_of_replicas': 0}

    class Django:
        model = Event

        fields = [
            "create_time",
            "event_id",
            "begin_time",
            "end_time",
            "bk_biz_id",
            "strategy_id",
            "level",
            "status",
            "is_ack",
            "p_event_id",
            "is_shielded",
            "shield_type",
            "target_key",
            "notify_status",
        ]

    origin_alarm = fields.ObjectField()
    origin_config = fields.ObjectField(properties={
        "strategy_name": fields.TextField(
            fields={
                'raw': fields.TextField(analyzer='keyword'),
                'suggest': fields.CompletionField(),  # 给需要生成搜索建议的字段定义一个CompletionField
            }
        )
    })
```

- views.py

```python
class EventDocumentViewSet(DocumentViewSet):

    document = EventDocument
    serializer_class = EventDocumentSerializer
    pagination_class = PageNumberPagination

    filter_backends = [
        SearchFilterBackend,
        DefaultOrderingFilterBackend,
        FilteringFilterBackend,
        SuggesterFilterBackend,  # 增加搜索建议的backend
    ]

    search_fields = {
        'origin_config.strategy_name': None,
        'origin_config.item_list.item_name': {'boost': 4},
    }

    ordering = ('_score',)

    filter_fields = {
        'strategy_id': None,
        'level': None,
        'begin_time': None,
        'end_time': None,
    }

    # 建议字段配置
    suggester_fields = {
        'strategy_name_suggest': {
            'field': 'origin_config.strategy_name.suggest',
            'suggesters': [
                SUGGESTER_COMPLETION,
            ],
            'default_suggester': SUGGESTER_COMPLETION,
            'options': {
                'size': 20,                # 建议条数
                'skip_duplicates': True,   # 是否去重
            },
        },
    }

```



通过 `suggest` 路由进行查询

```shell
# 查询以 "cpu" 开头的策略名称
curl "http://localhost:8000/event/suggest/?strategy_name_suggest=cpu"
```

返回格式如下

```json
{
  "strategy_name_suggest": [
    {
      "text": "cpu",
      "offset": 0,
      "length": 3,
      "options": [
        {
          "text": "CPU告警测试",
          "_index": "alarm_event-202002",
          "_type": "_doc",
          "_id": "01HCanMBPJqO3Kl8XXuP",
          "_score": 1.0,
          "_source": {...}
        },
        {
          "text": "CPU范围屏蔽测试",
          "_index": "alarm_event-202002",
          "_type": "_doc",
          "_id": "rlHCanMBPJqO3Kl8gIBH",
          "_score": 1.0,
          "_source": {...}
      ]
    }
  ]
}
```





### 3.5 faceted search - 分面搜索

分面搜索适用于根据某些字段进行聚合的场景

- views.py

```python
class EventDocumentViewSet(DocumentViewSet):

    document = EventDocument
    serializer_class = EventDocumentSerializer
    pagination_class = PageNumberPagination

    filter_backends = [
        SearchFilterBackend,
        DefaultOrderingFilterBackend,
        FilteringFilterBackend,
        SuggesterFilterBackend,
        FacetedSearchFilterBackend,   # 增加分面搜索backend
    ]

    search_fields = {
        'origin_config.strategy_name': None,
        'origin_config.item_list.item_name': {'boost': 4},
    }

    ordering = ('_score',)

    filter_fields = {
        'strategy_id': None,
        'level': None,
        'begin_time': None,
        'end_time': None,
    }

    suggester_fields = {
        'strategy_name_suggest': {
            'field': 'origin_config.strategy_name.suggest',
            'suggesters': [
                SUGGESTER_COMPLETION,
            ],
            'default_suggester': SUGGESTER_COMPLETION,
            'options': {
                'size': 20,               
                'skip_duplicates': True, 
            },
        },
    }

    # 配置支持分面搜索的字段
    faceted_search_fields = {
        'strategy_id': 'strategy_id',
        'begin_time': {
            'field': 'begin_time',
            'facet': DateHistogramFacet,
            'options': {
                'interval': 'day',   # 按天聚合
            }
        },
    }

```



通过 `facet` 参数指定聚合字段

```shell
# 按照 begin_time 聚合
curl "http://localhost:8000/event/?page=1&page_size=1&facet=begin_time"
```

返回结果

```json
{
  "facets": {
    "_filter_begin_time": {
      "doc_count": 201066,
      "begin_time": {
        "buckets": [
          {
            "key_as_string": "2020-02-18T00:00:00.000Z",
            "key": 1581984000000,
            "doc_count": 123
          },
          {
            "key_as_string": "2020-02-19T00:00:00.000Z",
            "key": 1582070400000,
            "doc_count": 47
          },
          {
            "key_as_string": "2020-02-20T00:00:00.000Z",
            "key": 1582156800000,
            "doc_count": 3
          },
          {
            "key_as_string": "2020-02-21T00:00:00.000Z",
            "key": 1582243200000,
            "doc_count": 203
          },
          {
            "key_as_string": "2020-02-22T00:00:00.000Z",
            "key": 1582329600000,
            "doc_count": 41
          }
        ]
      }
    }
  },
  "results": [...]
}
```



> 注意：框架仅支持单个字段的聚合，若想要同时对多个字段进行聚合，需要重写 FacetedSearchFilterBackend



## 4. 总结

### 4.1 使用到的框架与模块

|                    | **Elasticsearch**                           | **MySQL**           |
| ------------------ | ------------------------------------------- | ------------------- |
| **Client**         | elasticsearch-py                            | mysqlclient         |
| **Model 层** (ORM) | elasticsearch-dsl, django-elasticsearch-dsl | django model        |
| **View 层**        | django-elasticsearch-dsl-drf                | djangorestframework |

对比表格中可以看到，从 Model 层到 View 层，ES都有对应的框架与模块支持。开发成本上与MySQL其实相差无几。

### 4.2 与MySQL对比

- 优点
  - 查询功能强大。通过简单配置，即可支持大多数查询场景
  - 查询效率高
  - 与MySQL的分库分表方案相比，ES索引拆分的方式更便于开发和维护
- 缺点
  - 数据写入是非实时的
  - 数据量比较大时(1w+)，不支持分页操作

