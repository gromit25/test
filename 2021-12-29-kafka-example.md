---
layout: post
title: "Kafka 서버 Docker 설치 및 Java Publisher/Consumer 구현"
date: 2021-12-29
author: 손정모

---

이번 포스팅에서는 Docker를 이용한 Kafka 서버 구동 및 Java 언어로 간단한 Publisher와 Consumer 프로그램 구현방법에 대해 알아 보겠습니다.

## What is the Kafka?

Kafka는 Publisher-Consumer 구조의 메시지 큐 서비스입니다.    
Publisher에서 발행된 메시지는 Kafka 서버의 메시지 큐에 저장됩니다.   
Consumer는 구독 중인 Kafka서버의 메시지 큐에 있는 메시지를 수신 받아 처리할 수 있습니다.   
Kafka 서버의 메시지 큐는 파일 형태로 저장됩니다.   
Kafka는 가용성 향상 등을 위해 Local 파일이 아닌 분산 파일 시스템(ZooKeeper)를 사용합니다.   
이는 메모리를 사용할 경우 발생하는 메모리 사용량의 한계와 분산환경에서의 가용성 및 확장성을 확보할 수 있게 합니다.

 ![image](1)

### Publisher
Publisher는 특정 Topic에 메시지를 발행합니다.   
Topic은 여러개의 Partition을 가지고 있습니다.   
Publisher가 Topic의 Partition을 지정하여 입력할 수도 있고, Topic만 지정하여 메시지를 발행할 수도 있습니다.    

 ![image](2)

### Consumer
Consumer는 ConsumerGroup에 Consumer들을 가지는 구조입니다.    
즉, Topic은 ConsumerGroup이 소비하고, Topic의 Partition은 ConsumerGroup의 Consumer가 소비하는 구조입니다.    
Topic의 하나의 Partition은 하나의 Consumer를 가질 수 있고, 하나의 Consumer는 여러 Partition을 구독할 수 있습니다.    
Consumer는 poll 함수를 이용하여 메시지를 수신 받아 처리할 수 있습니다.    
단, Consumer의 메시지 처리시간이 느려 다음 poll까지 상당히 시간이 걸린 경우, rebalancing이 일어나게 됩니다.    
rebalancing 완료때까지 Consumer에서는 메시지를 처리할 수 없습니다.     
이런 rebalancing을 최소화하기 위해서는 max.poll.interval.ms 설정값이 적절한지 확인이 필요합니다.     
또한, heart beat의 타임아웃 시간으로 session.timeout.ms값을 사용하는데, 이를 초과할 시 rebalancing이 발생하기 때문에 주의 깊게 설정하여야 합니다.    
 
 ![image](3)

 
## Kafka 서버 Docker 실행

### 1. Kafka Image Pull
먼저 docker hub에서 ZooKeeper와 Kafka의 image를 서버로 가져옵니다.

```
docker pull wurstmeister/zookeeper:3.4.6
docker pull wurstmeister/kafka:2.13-2.7.0
```

### 2. Kafka docker-compose.yaml 설정 파일 작성
docker 이미지를 따로 실행 시킬 수 있으나, 매번 설정을 해야하는 귀찮음을 줄이기 위해 docker-compose를 사용하여 실행하겠습니다.    
    
kafka의 설정 중    
    
KAFKA_ADVERTISED_LISTENERS, KAFKA_ADVERTISED_HOST_NAME는 Java Publisher와 Consumer가 접근할 수 있는 URL로 설정하여야 합니다.    
만일 'localhost'로 설정하는 경우에는 다른 컴퓨터에서 접근할 수가 없어 제대로 동작하지 않습니다.    
비동기 통신이라 그런지 오류도 않나지만 메시지를 발행할 수도 수신할 수도 없습니다.    

KAFKA_CREATE_TOPICS는 kafka 서버가 구동될 때 기본적으로 생성되는 topic의 정보 입니다.     
형식은 'Topic명:Partition개수:Replica개수'입니다.    
     
KAFKA_JMX_OPTS, JMX_PORT는 Java Management eXtension 기능을 이용하여 외부에서 모니터링시에 필요합니다.    
필요 없으면 삭제해도 상관 없습니다.    
      
```
version: '2'
networks:
        test:
services:
        zookeeper:
                image: wurstmeister/zookeeper:3.4.6
                container_name: zookeeper
                ports:
                        - "2181:2181"
                networks:
                        - test
        kafka:
                image: wurstmeister/kafka:2.13-2.7.0
                container_name: kafka
                environment:
                        KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://(kafka 서버 주소):9092
                        KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
                        KAFKA_ADVERTISED_HOST_NAME: (kafka 서버 주소)
                        KAFKA_ADVERTISED_PORT: 9092
                        KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
                        KAFKA_CREATE_TOPICS: "my-topic:1:1"
                        KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=127.0.0.1 -Dcom.sun.management.jmxremote.rmi.port=1099"
                        JMX_PORT: 1099
                volumes:
                        - /var/run/docker.sock:/var/run/docker.sock
                ports:
                        - "9092:9092"
                        - "1099:1099"
                depends_on:
                        - zookeeper
                networks:
                        - test
```

### 3. Docker Compose 실행 및 중단
docker compose는 현재 디렉토리의 docker-compose.yaml을 참조하여 실행합니다.     
     
Docker Compose 실행     
```
$ docker-compose up -d
Creating network "kafka_test" with the default driver
Creating zookeeper ... done
Creating kafka     ... done
$
```

Docker Compose 중단
```
$ docker-compose down
Stopping kafka     ... done
Stopping zookeeper ... done
Removing kafka     ... done
Removing zookeeper ... done
Removing network kafka_test
$
```
     
## Java Consumer 구현
Java Consumer를 구현하기 위해 Kafka Client 라이브러리를 Maven을 이용하여 가져옵니다.     

```xml
<dependencies>
  <dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.8.0</version>
  </dependency>
</dependencies>
```

Java Consumer는 연결정보 설정 및 Consumer 객체 생성 부와 Kafka 메시지 수신 및 처리부로 나눌 수 있습니다.
GROUP_ID_CONFIG는 적절한 이름을 정하면 됩니다(아무거나).    
단, 동일한 Consumer Group으로 묶기 위해서는 GROUP_ID_CONFIG의 이름이 같아야 합니다.     


```java
// --------- 1. Kafka 연결정보 설정 및 Consumer 객체 생성 --------------
Properties properties = new Properties();
properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "(kafka 서버 주소):9092");
properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
properties.put(ConsumerConfig.GROUP_ID_CONFIG, "my-topic-consumer-group");

KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(properties);
consumer.subscribe(Collections.singletonList(topic));

// --------- 2. Kafka 메시지 수신 및 처리부 -------------- 
try {
        	
    // Kafka로 부터 메시지를 받아 출력
    while(true) {
	
        // Kafka로 부터 메시지를 받아 출력
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(10 * 1000));
        String message = null;

        for (ConsumerRecord<String, String> record : records) {
            message = record.value();
            System.out.println("Received Message:" + message + ":");
        }
    }

} catch(Exception ex) {
    ex.printStackTrace();
} finally {
    consumer.close();
}
```

## Java Publisher의 구현

Java Publisher는 Consumer와 거의 유사하게 Kafka 연결정보 설정 및 Consumer 객체 생성부와 Kafka 메시지 송신부로 나누어 보았습니다.     

```java
// --------- 1. Kafka 연결정보 설정 및 Consumer 객체 생성 --------------
Properties props = new Properties();
props.put("bootstrap.servers", "(kafka 서버 주소):9092");
props.put("key.serializer", StringSerializer.class.getName());
props.put("value.serializer", StringSerializer.class.getName());

Producer<String, String> producer = new KafkaProducer<String, String>(props);

// --------- 2.Kafka 메시지 송신 --------------
try {
    Random random = new Random();

    while(true) {

        ProducerRecord<String, String> record = null;

        // partition id를 지정하지 않을 경우,
        record = new ProducerRecord<String, String>(topic, key, message);
        // partition id를 지정하는 경우
	// record = new ProducerRecord<String, String>(topic, 1, key, message);

        // Kafka에 메시지 전송
        producer.send(record);

        // 최대 5초 이내의 랜덤 시간 동안 대기 후 다시 전송
        long sleep = random.nextInt(5);
        System.out.println("wait " + sleep + "seconds.");
        Thread.sleep(sleep * 1000);
    }

} catch (Exception ex) {
    ex.printStackTrace();
} finally {
    producer.close();
}
```

## Java Publisher와 Consumer의 실행
이클립스 화면에서 실행한 화면 입니다.    

Java Publisher 화면     
(4)     

Java Consumer 화면     
(5)     
     
이번 포스트에서는 Kafka 서버 구동 및 Java Publisher/Consumer 구현 방법에 대해 알아 보았습니다.     
다음번에는 Kafka Schema Registry를 이용한 데이터 송수신 방법에 대해 알아 보겠습니다.     
     
감사합니다.     
