# Python 样例代码
## 环境需求
Python版本为2.7、3.4、3.5、3.6或3.7。  

## 安装Python依赖库
执行以下命令安装Python依赖库。  
```
pip install kafka-python
```
## 准备配置
创建Kafka配置文件setting.py。
```
kafka_setting = {
    'bootstrap_servers': ["XXX", "XXX", "XXX"],
    'topic_name': 'XXX',
    'consumer_id': 'XXX'
}
```

##发送消息
1. 创建发送消息程序kafka_producer.py。
```
#!/usr/bin/env python
# encoding: utf-8

import socket
from kafka import KafkaProducer
from kafka.errors import KafkaError
import setting

conf = setting.kafka_setting

print conf


producer = KafkaProducer(bootstrap_servers=conf['bootstrap_servers'],
                        api_version = (0,10),
                        retries=5)

partitions = producer.partitions_for(conf['topic_name'])
print 'Topic下分区: %s' % partitions

try:
    future = producer.send(conf['topic_name'], 'hello kafka!')
    future.get()
    print 'send message succeed.'
except KafkaError, e:
    print 'send message failed.'
    print e
```
2. 执行以下命令发送消息。
```
python aliyun_kafka_producer.py
```

## 订阅消息
1. 创建订阅消息程序kafka_consumer.py
```
#!/usr/bin/env python
# encoding: utf-8

import socket
from kafka import KafkaConsumer
from kafka.errors import KafkaError
import setting

conf = setting.kafka_setting

consumer = KafkaConsumer(bootstrap_servers=conf['bootstrap_servers'],
                        group_id=conf['consumer_id'],
                        api_version = (0,10,2), 
                        session_timeout_ms=25000,
                        max_poll_records=100,
                        fetch_max_bytes=1 * 1024 * 1024)

print 'consumer start to consuming...'
consumer.subscribe((conf['topic_name'], ))
for message in consumer:
    print message.topic, message.offset, message.key, message.value, message.partition
```

2. 执行以下命令消费消息
```
python aliyun_kafka_consumer.py
```