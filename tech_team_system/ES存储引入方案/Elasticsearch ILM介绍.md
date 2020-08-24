# ILM：索引生命周期管理



## 1. 概念

随着索引的数据量增长，单个索引的写入和查询性能都会受到影响，占用的资源也会越来越多（如磁盘、CPU）。为了保证Elasticsearch的性能，索引从产生到删除的整个生命周期，需要有一个机制去进行管理。这个管理机制就叫做 ILM（索引生命周期管理）



### 索引的生命周期

ES 索引生命周期管理分为4个阶段：`hot`, `warm`, `cold`, `delete`

- **Hot**: 索引被频繁查询和更新
- **Warm**: 索引不再有更新请求，但仍然存在查询请求
- **Cold**: 索引不再有更新请求，且查询请求也比较少了。但要求数据还是能被搜索到，哪怕查询速度慢一点
- **Delete**: 索引数据不再使用了，可以安全地进行删除

注意，不是所有的索引都必须会经历每个声明周期。比如，索引只有 `Hot` 和 `Delete` 2个阶段，也是允许的



## 2. 功能说明

从 7.0 版本开始，Elasticsearch 正式提供了索引生命周期管理策略 (ILM Policy) 配置功能，用于设置索引的每个阶段的流转条件以及每个阶段需要执行的操作

### 2.1 阶段流转

ILM根据策略配置在整个生命周期中进行流转。需要为每个阶段设置最小触发期限。要使索引流转到下一阶段，当前阶段中的所有操作都必须先完成，且下一个阶段触发期限必须大于上一阶段的期限。每隔一段时间，ES会检查索引是否达到了阶段流转的条件，轮询时间可通过 `indices.lifecycle.poll_interval` 参数配置（默认10分钟）

### 2.2 阶段执行

每个阶段可以定义多个动作，并且是有序的。ILM 负责控制每个阶段中操作的执行。



#### 支持的动作列表

- **Rollover**：在现有索引达到某个时间、文档数或大小后，开始写入到新的索引。
- **Shrink**：减少索引中主分片的数量
- **Force merge**：手动触发合并，以减少索引的每个分片中的 segment 数，并释放被删除文档使用的空间。
- **Freeze**：冻结索引，最小化其内存占用。
- **Delete**：永久删除索引，包括其所有数据和元数据。
- **Set Priority**：设置索引恢复的优先级
- **Allocate**：调整索引配置，如副本数量，重新分配节点



#### 每个阶段支持的动作

- Hot
  - [Set Priority](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-set-priority.html) 
  - [Unfollow](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-unfollow.html)
  - [Rollover](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-rollover.html)
- Warm
  - [Set Priority](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-set-priority.html)
  - [Unfollow](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-unfollow.html)
  - [Read-Only](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-readonly.html)
  - [Allocate](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-allocate.html)
  - [Shrink](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-shrink.html)
  - [Force Merge](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-forcemerge.html)
- Cold
  - [Set Priority](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-actions.html#ilm-set-priority-action)
  - [Unfollow](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-actions.html#ilm-unfollow-action)
  - [Allocate](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-allocate.html)
  - [Freeze](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-freeze.html)
- Delete
  - [Wait For Snapshot](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-actions.html#ilm-wait-for-snapshot-action)
  - [Delete](https://www.elastic.co/guide/en/elasticsearch/reference/7.8/ilm-delete.html)



## 3. 使用方法

### 需求

- 当单个索引大小达到50GB或索引创建超过7天，就进行轮转
- 将7天前的索引设置为只读，分片数量减少为1
- 冻结30天前的索引，且将索引分配至cold节点
- 删除60天前的索引



### 3.1 创建ILM Policy

```js
PUT _ilm/policy/alarm_event_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover" : {
            "max_size": "50GB",
            "max_age": "7d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink" : {
            "number_of_shards": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze" : {},
          "allocate" : {
            "include" : {
                "box_type": "cold"
            }
          }
        }
      },
      "delete": {
        "min_age": "60d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```



### 3.2 创建索引模板并设置策略

```js
PUT _template/alarm_event
{
  "index_patterns": ["alarm_event-*"],                 
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.lifecycle.name": "alarm_event_policy",      
    "index.lifecycle.rollover_alias": "alarm_event"    
  }
}
```



### 3.3 创建初始索引

需要先创建一个初始索引，这样ILM才会被启动

```js
PUT alarm_event-000001

{
  "aliases": {
    "alarm_event": {
      "is_write_index": true
    }
  }
}
```

每次满足轮转条件后，会自动创建一个名为 `alarm_event-000002`的新索引，并添加别名 `alarm_event` 。新索引名称同样匹配 `alarm_event-*` 索引模板，因此模板中的ILM策略配置将应用于新索引。然后新索引会被指定为写入索引 (`is_write_index` 会被设置为 `true`，旧索引则被设置为 `false`)

- 查询：使用 `alarm_event` 作为索引名称，会对所有符合 `alarm_event-*` 的索引进行查询 
- 写入：使用 `alarm_event` 作为索引名称，会将数据写入到当前指向的最新索引



### 3.4 检查索引生命周期过程

索引配置了ILM后，可以通过API查询到当前索引生命周期详情

- 索引处于什么阶段以及何时进入该阶段

- 当前正在执行的动作以及所处的步骤
- 执行过程中是否有发生异常

```js
GET alarm_event-*/_ilm/explain

{
    "indices": {
        "alarm_event-000001": {
            "index": "alarm_event-000001",
            "managed": true,
            "policy": "alarm_event_policy",
            "lifecycle_date_millis": 1596376069251,
            "age": "12.12h",
            "phase": "hot",
            "phase_time_millis": 1596419593849,
            "action": "rollover",
            "action_time_millis": 1596376395563,
            "step": "check-rollover-ready",
            "step_time_millis": 1596419593849,
            "is_auto_retryable_error": true,
            "failed_step_retry_count": 2,
            "phase_execution": {
                "policy": "alarm_event_policy",
                "phase_definition": {
                    "min_age": "0ms",
                    "actions": {
                        "rollover": {
                            "max_size": "50gb",
                            "max_age": "7d"
                        }
                    }
                },
                "version": 1,
                "modified_date_in_millis": 1596375703432
            }
        }
    }
}
```



> 参考：https://www.elastic.co/guide/en/elasticsearch/reference/7.8/getting-started-index-lifecycle-management.html#ilm-gs-create-policy

## 4. 在 Django 中的使用

与 `django-elasticsearch-dsl` 结合

```python
@registry.register_document
class EventDocument(Document):

    class Index:
        name = 'alarm_event'
        # ES 索引的配置
        settings = {
            "number_of_shards": 2,
            "number_of_replicas": 1,
            "index.lifecycle.name": "alarm_event_policy",      # 使用的轮转策略
            "index.lifecycle.rollover_alias": "alarm_event",   # 用于轮转的别名
        }

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

    @classmethod
    def init_index(cls):
        """
        初始化索引
        """
        
        # 获取 ES 客户端
        es_client = cls._index._get_connection()

        # 1. 创建/更新 ILM 策略
        es_client.ilm.put_lifecycle(
            policy=cls.Index.settings["index.lifecycle.name"],
            body={
                "policy": {
                    "phases": {
                        "hot": {
                            "actions": {
                                "rollover": {
                                    "max_size": "50gb",
                                    "max_age": "7d"
                                }
                            }
                        },
                        "warm": {
                            "min_age": "7d",
                            "actions": {
                                "shrink": {
                                    "number_of_shards": 1
                                }
                            }
                        },
                        "cold": {
                            "min_age": "30d",
                            "actions": {
                                "freeze": {},
                                "allocate": {
                                    "include": {
                                        "box_type": "cold"
                                    }
                                }
                            }
                        },
                        "delete": {
                            "min_age": "60d",
                            "actions": {
                                "delete": {}
                            }
                        }
                    }
                }
            }
        )

        # 2. 创建/更新索引模板
        index_template = cls._index.as_template(template_name=cls.Index.name, pattern=f"{cls.Index.name}-*")
        index_template.save()

        # 3. 创建引导索引
        bootstrap_index_name = f"{cls.Index.name}-000001"
        
        if es_client.indices.exists(index=bootstrap_index_name):
            # 如果存在，则不进行创建
            return

        es_client.indices.create(
            index=bootstrap_index_name,
            body={
                "aliases": {
                    cls.Index.name: {
                        "is_write_index": True
                    }
                }
            }
        )
```

添加一个 migrations，执行 `EventDocument.init_index()` 即可。允许重复执行。