# Python 样例代码（SASL）
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
    'sasl_plain_username': 'XXX',
    'sasl_plain_password': 'XXX',
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

from kafka import KafkaProducer
from kafka.errors import KafkaError
import kafka_setting

conf = kafka_setting.kafka_setting

print conf


producer = KafkaProducer(bootstrap_servers=conf['bootstrap_servers'],
                         sasl_mechanism="PLAIN",
                         security_protocol='SASL_PLAINTEXT',
                         api_version=(0, 10),
                         retries=5,
                         sasl_plain_username=conf['sasl_plain_username'],
                         sasl_plain_password=conf['sasl_plain_password'])

partitions = producer.partitions_for(conf['topic_name'])
print 'Topic下分区: %s' % partitions

try:
    future = producer.send(conf['topic_name'], 'hello kafka!!')
    future.get()
    print 'send message succeed.'
except KafkaError, e:
    print 'send message failed.'
    print e

```

2. 执行以下命令发送消息。
```
python kafka_producer.py
```

## 订阅消息
1. 创建订阅消息程序kafka_consumer.py
```
#!/usr/bin/env python
# encoding: utf-8

from kafka import KafkaConsumer
import kafka_setting

conf = kafka_setting.kafka_setting


consumer = KafkaConsumer(bootstrap_servers=conf['bootstrap_servers'],
                         group_id=conf['consumer_id'],
                         sasl_mechanism="PLAIN",
                         security_protocol='SASL_PLAINTEXT',
                         api_version=(0, 10),
                         sasl_plain_username=conf['sasl_plain_username'],
                         sasl_plain_password=conf['sasl_plain_password'])

print 'consumer start to consuming...'
consumer.subscribe((conf['topic_name'],))
for message in consumer:
    print message.topic, message.offset, message.key, message.partition, message.value
```

2. 执行以下命令消费消息
```
python kafka_consumer.py
```