# 默认接入点收发消息

本文介绍如何在 VPC 环境下使用 Python SDK 接入消息队列 CKafka 版的默认接入点并收发消息。


#### 前提条件
- [安装 python](https://www.python.org/downloads/)
- [安装 pip](https://pip-cn.readthedocs.io/en/latest/installing.html)

#### 添加 Python 依赖库
```bash
pip install kafka-python
```
#### 修改配置
##### 生产消息
```python
#coding:utf8
from kafka import KafkaProducer

producer = KafkaProducer(
   bootstrap_servers = ['$ip:$port'],
   api_version = (0,10,0)
)

producer.send('$topic_name',value = "Hello World! Hello Ckafka!")
producer.close()
```
| 参数 | 描述 |
| --- | --- |
| bootstrap_servers | 接入点。在【消息队列 CKafka】控制台 -【xx实例详情页面】-【基本信息】-【接入方式】或【内网IP与端口】获取 |
| topic_name | Topic名称。在【消息队列 CKafka】控制台 -【xx实例详情页面】-【topic管理】创建和获取 |

##### 消费消息
```python
#coding:utf8
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    '$topic_name',
    group_id = "$group_id",
    bootstrap_servers = ['$ip:$port'],
    api_version = (0,10,0)
)

for message in consumer:
    print ("Topic:[%s] Partition:[%d] Offset:[%d] Value:[%s]" % (message.topic, message.partition, message.offset, message.value))
```
| 参数 | 描述 |
| --- | --- |
| bootstrap_servers | 接入点。在【消息队列 CKafka】控制台 -【xx实例详情页面】-【基本信息】-【接入方式】或 【内网IP与端口】获取 |
| group_id | 消费者的组 Id。根据业务需求自定义 |
| topic_name | Topic名称。在【消息队列 CKafka】控制台 -【xx实例详情页面】-【topic管理】创建和获取 |

