```
pip install kafka-python
```

```python

from kafka import KafkaConsumer
from json import loads

consumer = KafkaConsumer('test-topic',
                         bootstrap_servers=['192.168.10.9:9092'],
                         auto_offset_reset='latest',
                         enable_auto_commit=True,
                         group_id='my-topic-consumers',
                         value_deserializer=lambda x: loads(x.decode('utf-8')),
                         consumer_timeout_ms=1000
                         )

print('start receiving message from kafka')

while True:
    message = consumer.poll(timeout_ms=1000)
    if len(message) != 0:
        print(message)
        
```

```python
from kafka import KafkaProducer
from json import dumps

producer = KafkaProducer(bootstrap_servers=['192.168.10.9:9092'],
                         value_serializer=lambda x: dumps(x).encode('utf-8')
                         )

producer.send('test-topic', value='hello world-1!')
producer.send('test-topic', value='hello world-2!')
producer.flush()
print('complete sending message')
```
