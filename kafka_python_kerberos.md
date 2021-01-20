# Python 样例代码（SASL）
## 环境需求
Python版本为2.7、3.4、3.5、3.6或3.7。  
1. 执行如下命令，安装环境：
```
yum install librdkafka-devel python-devel
```
2. 拷贝krb5.conf文件

将krb5文件拷贝到/etc/ 目录下
## 安装Python依赖库
执行以下命令安装Python依赖库。  
```
pip install --no-binary :all: confluent-kafka
```

## 重新编译librdkafka
如果不重新编译librdkafka，则python会报错：  
recompile librdkafka with libsasl2 or openssl support. current build options: plain sasl_scram oauthbearer  
1. 下载源码
```
git clone https://github.com/edenhill/librdkafka.git
```
2. 检查依赖
```
[root@localhost ~]# rpm -qa| grep openssl
openssl-1.0.2k-21.el7_9.x86_64
openssl-devel-1.0.2k-21.el7_9.x86_64
openssl-libs-1.0.2k-21.el7_9.x86_64
```
3. 编译安装
```
./configure
make && make install
```
如果configure的时候出现如下信息则正确：  
```
checking for libssl (by pkg-config)... ok
checking for libssl (by compile)... ok (cached)
```

4. 替换库文件

先找到python使用的库文件：  
```
[root@localhost lib]# find / -name "librdkafka*"
/root/venv/lib/python2.7/site-packages/confluent_kafka/.libs/librdkafka.so.1
/root/venv/lib/python2.7/site-packages/confluent_kafka/.libs/librdkafka-dfeb6276.so.1
```  
替换库文件：  
```
[root@localhost lib]# cd /root/venv/lib/python2.7/site-packages/confluent_kafka/.libs/
[root@localhost .libs]# ln -s /usr/local/lib/librdkafka.so.1 
[root@localhost .libs]# mv librdkafka-dfeb6276.so.1 librdkafka-dfeb6276.so.1.bak
[root@localhost .libs]# mv librdkafka.so.1 librdkafka-dfeb6276.so.1
```
librdkafka-dfeb6276.so.1 后缀名需要根据具体的环境进行替换。
## 准备配置
创建Kafka配置文件setting.py。
```
kafka_setting = {
    'principal': 'xxx',
    'keytab_file': '/xxx/xxx.keytab',
    'bootstrap_servers': 'xxx:9092,xxx:9092,xxx:9092',
    'topic_name': 'xxx',
    'consumer_id': 'xxx'
}
```

##发送消息
1. 创建发送消息程序producer.py。
```
#!/usr/bin/env python
#
# Copyright 2020 Confluent Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

# =============================================================================
#
# Produce messages to Confluent Cloud
# Using Confluent Python Client for Apache Kafka
#
# =============================================================================

from confluent_kafka import Producer, KafkaError
import json
import settings

conf = settings.kafka_setting

if __name__ == '__main__':

    # Read arguments and configurations and initialize

    # Create Producer instance
    producer = Producer({
        'bootstrap.servers': conf['bootstrap_servers'],
        'sasl.mechanisms': 'GSSAPI',
        'security.protocol': 'SASL_PLAINTEXT',
        'sasl.kerberos.service.name': 'kafka',
        'sasl.kerberos.keytab': conf['keytab_file'],
        'sasl.kerberos.principal': conf['principal'],
    })

    topic = conf['topic_name']
    delivered_records = 0


    # Optional per-message on_delivery handler (triggered by poll() or flush())
    # when a message has been successfully delivered or
    # permanently failed delivery (after retries).
    def acked(err, msg):
        global delivered_records
        """Delivery report handler called on
        successful or failed delivery of message
        """
        if err is not None:
            print("Failed to deliver message: {}".format(err))
        else:
            delivered_records += 1
            print("Produced record to topic {} partition [{}] @ offset {}"
                  .format(msg.topic(), msg.partition(), msg.offset()))


    for n in range(5):
        record_key = "Bob"
        record_value = json.dumps({'count': n})
        print("Producing record: {}\t{}".format(record_key, record_value))
        producer.produce(topic, key=record_key, value=record_value, on_delivery=acked)
        # p.poll() serves delivery reports (on_delivery)
        # from previous produce() calls.
        producer.poll(0)

    producer.flush()

    print("{} messages were produced to topic {}!".format(delivered_records, topic))

```

2. 执行以下命令发送消息。
```
python producer.py
```
## 订阅消息
1. 创建订阅消息程序kafka_consumer.py
```
#!/usr/bin/env python
# encoding: utf-8
#
# Copyright 2020 Confluent Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

# =============================================================================
#
# Produce messages to Confluent Cloud
# Using Confluent Python Client for Apache Kafka
#
# =============================================================================

from confluent_kafka import Consumer, KafkaError
import settings

conf = settings.kafka_setting

if __name__ == '__main__':

    # Read arguments and configurations and initialize

    # Create Consumer instance
    consumer = Consumer({
        'bootstrap.servers': conf['bootstrap_servers'],
        'group.id': conf['consumer_id'],
        'sasl.mechanisms': 'GSSAPI',
        'security.protocol': 'SASL_PLAINTEXT',
        'sasl.kerberos.service.name': 'kafka',
        'sasl.kerberos.keytab': conf['keytab_file'],
        'sasl.kerberos.principal': conf['principal'],
    })

    topic = conf['topic_name']
    print topic
    print 'consumer start to consuming...'
    consumer.subscribe([topic])
    while True:
        message = consumer.poll(1)
        if message:
            print message.topic(), message.offset(), message.key(), message.partition(), message.value()

```

2. 执行以下命令消费消息
```
python kafka_consumer.py
```