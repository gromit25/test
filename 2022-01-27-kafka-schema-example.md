---
layout: post
title: "Kafka Schema Registry 서버 Docker 설치 및 Java Producer/Consumer 구현"
date: 2022-01-27
author: 손정모

---
    
이번 포스팅에서는 지난 "Kafka 서버 Docker 설치 및 Java Producer/Consumer 구현"에 이어
Kafka Schema 서비스에 대해 알아 보겠습니다.    
    
## What is the Kafka Schema?    
    
Kafka Schema Registry 서비스는 메시지의 구조를 정의하고 관리하는 기능을 제공합니다.    
    
아래와 같은 그림을 보면    
Producer가 Id, Event, Name을 보내는 중이었으나,    
변경 요청으로 인해 Name을 firstName, lastName으로 변경하여 Schema가 변경되었습니다.    
그런데 Consumer 가 수정되지 않은 것이죠.. 심각한 장애가 발생할 수도 있는 상황입니다.    
    
 ![image](1)
    
 ![image](2)
    
Kafka Schema Registry 서버는 기존 Kafka 서버와는 독립적으로 구성됩니다. 
Producer/Consumer는 메시지 전송 및 수신시에 Kafka Schema Registry 서버와의 통신을 통해 메시지의 Schema 정보를 획득할 수 있습니다.    
    
 ![image](3)
    
메시지의 Schema는 Apache AVRO를 통해 정의할 수 있습니다.    
AVRO는 JSON 방식 메시지 구조 정의 방식 입니다.    
예를 들어, 영화정보(제목, 상영년도)에 대해 아래와 같이 정의할 수 있습니다.    
```
{
  "namespace": "com.epozen.kafka.schema",
  "type": "record",
  "name": "Movie",
  "fields": [
    { "name": "title", "type": "string" },
    { "name": "year", "type": "int" }
  ]
}
```
    
상기 영화정보를 이용하여 Producer에서 메시지를 생성하고 Consumer에서 처리하는 방법을 알아 보겠습니다.    
아래의 과정으로 진행하겠습니다.    
    
* Docker를 이용한 Kafka 및 Kafka Schema Registry 서버 실행    
* AVRO 정의 및 정의된 AVRO 파일로 부터 Java 파일 생성(early binding) - Provider/Consumer 공통 작업    
* Consumer 구현    
* Producer 구현    
* 실행 결과 확인    
    
     
## Docker를 이용한 Kafka 및 Kafka Schema Registry 서버 실행    
    
### 1. Kafka 및 Kafka Schema Registry Image Pull    
먼저 docker hub에서 ZooKeeper와 Kafka, Kafka Schema Registry의 image를 서버로 가져옵니다.    
    
```
docker pull wurstmeister/zookeeper:3.4.6
docker pull wurstmeister/kafka:2.13-2.7.0
docker pull confluentinc/cp-schema-registry:5.3.6
```
    
### 2. Kafka 및 Kafka Schema Registry docker-compose.yaml 설정 파일 작성
docker 이미지를 따로 실행 시킬 수 있으나, 매번 설정을 해야하는 귀찮음을 줄이기 위해 docker-compose를 사용하여 실행하겠습니다.    
    
kafka의 설정 값 관련해서는 이전 포스팅("Kafka 서버 Docker 설치 및 Java Producer/Consumer 구현") 참고하여 주세요.    
    
* SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS는 kafka 서버를 지정하면 됩니다.    

      
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

        schema-registry:
                image: confluentinc/cp-schema-registry:5.3.6
                environment:
                        SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'PLAINTEXT://kafka:9092'
                        SCHEMA_REGISTRY_HOST_NAME: 'schema-registry'
                        SCHEMA_REGISTRY_LISTENERS: 'http://0.0.0.0:8085'
                        SCHEMA_REGISTRY_LOG4J_ROOT_LOGLEVEL: 'INFO'
                ports:
                        - 8085:8085
                depends_on:
                        - kafka
                networks:
                        - test
```
    
### 3. Docker Compose 실행 및 중단    
docker compose는 현재 디렉토리의 docker-compose.yaml을 참조하여 실행합니다.     
     
Docker Compose 실행     
```
$ docker-compose up -d
Creating network "kafka_test" with the default driver
Creating zookeeper ...
Creating zookeeper ... done
Creating kafka ...
Creating kafka ... done
Creating kafka_schema-registry_1 ...
Creating kafka_schema-registry_1 ... done
$
```

Docker Compose 중단
```
$ docker-compose down
Stopping kafka_schema-registry_1 ... done
Stopping kafka                   ... done
Stopping zookeeper               ... done
Removing kafka_schema-registry_1 ... done
Removing kafka                   ... done
Removing zookeeper               ... done
Removing network kafka_test
$
```
    
## AVRO 정의 및 정의된 AVRO 파일로 부터 Java 파일 생성(early binding) - Provider/Consumer 공통 작업
    
Java Producer와 Consumer에서 사용할 수 있도록 AVRO 파일에 정의된 Schema를 참조하여 Java 파일을 생성합니다.    
eclipse 기준으로 설명하겠습니다.    
AVRO파일에서 Java 파일 생성하기 위한 라이브러리를 pom.xml에 추가합니다.
repository와 dependency를 추가합니다.    
    
```xml
<repositories>
    ...  
    <!-- confluent사의 kafka-avro-serializer를 가져오기 위한 maven repository -->
    <repository>
        <id>confluent</id>
        <url>http://packages.confluent.io/maven/</url>
    </repository>
    ...
</repositories>
...

<dependencies>
    ...
    <!-- Kafka 라이브러리 -->
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>2.8.0</version>
    </dependency>

    <!-- AVRO Apache 라이브러리 -->
    <dependency>
        <groupId>org.apache.avro</groupId>
        <artifactId>avro</artifactId>
        <version>1.10.2</version>
    </dependency>
    
    <!-- Kafka용 AVRO 객체 생성(early binding)을 위한 라이브러리 -->
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-avro-serializer</artifactId>
        <version>6.2.1</version>
    </dependency>
    ...    
</dependencies>
```
    
그리고 plugin을 추가합니다.    
    
```xml
<build>
    <plugins>
      ...
      <!-- Java 컴파일러 버전 1.8 기준 -->
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.7.0</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>

      <!--
      AVRO early binding
      .avro -> .java 파일 생성
      -->
      <plugin>
        <groupId>org.apache.avro</groupId>
        <artifactId>avro-maven-plugin</artifactId>
        <version>1.10.2</version>
        <executions>
          <execution>
            <phase>generate-sources</phase>
            <goals>
              <goal>schema</goal>
            </goals>
            <configuration>
              <sourceDirectory>${project.basedir}/src/main/avro/</sourceDirectory>
              <outputDirectory>${project.basedir}/src/main/java/</outputDirectory>
            </configuration>
          </execution>
        </executions>
      </plugin>
      ...
    </plugins>
</build>
```
    
추가 후에 오류가 하나 발생합니다.    
"<sourceDirectory>${project.basedir}/src/main/avro/</sourceDirectory>" 이것 때문이죠.    
avro 디렉토리가 없기 때문 입니다.    
    
![image](4)    
    
이제 avro 디렉토리를 생성해 보겠습니다.    
생성 후에 "프로젝트명 오른쪽 클릭 > 메뉴에서 Maven > Update Project" 아시죠. ^^    
    
![image](5)
    
오류가 없어 졌네요.    
이제 준비는 끝났습니다. AVRO 파일을 만들어 보겠습니다.    
    
방금 만든 avro 디렉토리에 Moview.avsc 파일에 아래의 내용으로 만들겠습니다.    
※ 확장자가 avsc 입니다. 주의하여 주십시요.

```
{
  "namespace": "com.epozen.kafka.schema",
  "type": "record",
  "name": "Movie",
  "fields": [
    { "name": "title", "type": "string" },
    { "name": "year", "type": "int" }
  ]
}
```

"프로젝트 명 오른쪽 클릭 > Run As > Maven generate-sources"를 선택합니다.    
    
![image](6)
    
아래와 같이 "src/main/java" 에 Movie.java 파일이 생성된 것을 알 수 있습니다.    
    
![image](7)
    
이제 Producer/Consumer에서 참조할 수 있습니다.    
만일 저와 같이 Procducer/Consumer의 프로젝트를 따로 만들면, 각각 프로젝트에 위의 작업을 따로 해주어야 합니다.    
    
## Java Consumer 구현
Java Consumer는 기존과 연결정보 설정 및 Consumer 객체 생성 부와 Kafka 메시지 수신 및 처리부로 나눌 수 있습니다.    
※ 현재 예에서 소스 중 "kafka 서버 주소"와 "Kafka schema registry 서버 주소"는 동일한 주소 입니다. 다른 서버에 설치하신 경우에는 각각의 서버 주소를 넣으시면 됩니다.   
    
```java
// --------- 1. Kafka 연결정보 설정 및 Consumer 객체 생성 --------------
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "(kafka 서버 주소):9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "Movie Consumer");
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class.getName());
props.put(KafkaAvroDeserializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "(kafka schema registry 서버 주소):8085");
props.put(KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG, true);

KafkaConsumer<String, Movie> consumer = new KafkaConsumer<String, Movie>(props);
consumer.subscribe(Collections.singletonList("my-topic"));

// --------- 2. Kafka 메시지 수신 및 처리부 -------------- 
try {

    System.out.println("START CONSUMER");

    // Kafka로 부터 메시지를 받아 출력
    while (true) {

        consumer.poll(Duration.ofMillis(1000)).forEach(record -> {

            System.out.println("received a message:" + record);

            // Movie 객체 수신
            Movie receivedMovie = record.value();

            System.out.println("\tTitle:" + receivedMovie.getTitle());
            System.out.println("\tYear:" + receivedMovie.getYear());
        });

        consumer.commitAsync();
    }
} finally {
    consumer.close();
}
```
    
## Java Producer의 구현
      
Java Producer는 Consumer와 거의 유사하게 Kafka 연결정보 설정 및 Producer 객체 생성부와 Kafka 메시지 송신부로 나누어 구현하였습니다.      
      
```java
// --------- 1. Kafka 연결정보 설정 및 Producer 객체 생성 --------------
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.10.9:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class.getName());
props.put(KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://192.168.10.9:8085");

KafkaProducer<String, GenericRecord> producer = new KafkaProducer<String, GenericRecord>(props);

// --------- 2.Kafka 메시지 송신 --------------
try {

    // 발송할 Movie 객체 생성
    Movie newMovie = Movie.newBuilder()
        .setTitle("Titanic2")
        .setYear(2021)
        .build();
        
    System.out.println("Start Send Message");
    producer.send(new ProducerRecord<String, GenericRecord>("my-topic-2", "TEST-1", newMovie));
    System.out.println("End Send Message");

    producer.flush();

} catch (Exception ex) {
    ex.printStackTrace();
} finally {
    producer.close();
}
```
     
## Java Producer와 Consumer의 실행     
이클립스 화면에서 실행한 화면 입니다.    
     
Java Producer 실행 화면     
![image](8)     
     
Java Consumer 실행 화면     
![image](9)     
     
이번 포스트에서는 Kafka Schema Registry 서버 구동 및 Java Producer/Consumer 구현 방법에 대해 알아 보았습니다.     
다음번에는 Kafka 모니터링에 대해 알아 보겠습니다.     
     
감사합니다.     
     
--------------------------
## References
https://medium.com/@gaemi/kafka-%EC%99%80-confluent-schema-registry-%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%9C-%EC%8A%A4%ED%82%A4%EB%A7%88-%EA%B4%80%EB%A6%AC-1-cdf8c99d2c5c    
https://scorpio-mercury.tistory.com/30    
https://seokdev.site/287    

