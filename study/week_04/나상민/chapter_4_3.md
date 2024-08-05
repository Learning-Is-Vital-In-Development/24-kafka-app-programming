# 4.3 카프카 컨슈머
컨슈머는 카프카 클러스터로 부터 데이터를 가져와서 처리한다.
## 4.3.1 멀티 스레드 컨슈머
카프카는 처리량을 늘리기 위해 파티션과 컨슈머 개수를 늘려서 운영할 수 있다.
파티션을 여러 개로 운영하는 경우 데이터를 병렬처리하기 위해서 파티션 개수와 컨슈머 개수를 동일하게 맞추는 것이 가장 좋은 방법이다.
토픽의 파티션은 1개 이상으로 이루어져 있으며 1개의 파티션은 1개 컨슈머가 할당되어 데이터를 처리 할 수 있다.

카프카에서 공식적으로 지원하는 라이브러리인 자바는 멀티 스레드를 지원하므로 자바 어플리케이션에서는 멀티 스레드로 동작하는 멀티 컨슈머 스레드를 개발, 적용할 수 있다.

멀티 스레드로 컨슈머를 안전하게 운영하기 위해서는 고려해야 할 부분이 많다.
우선, 멀티 스레드 애플리케이션으로 개발할 경우 하나의 프로세스 내부에 스레드가 여러 개 생성되어 실행되기 때문에 
하나의 컨슈머 스레드에서 예외적 상황이 발생할 경우 프로세스 자체가 종료될 수 있고 이는 다른 컨슈머 스레드에까지 영향을 미칠 수 있다.
컨슈머 스레드들이 비정상적으로 종료될 경우 데이터 처리에서 중복 또는 유실이 발생 할 수 있기 때문이다.
또한, 각 컨슈머 스레드 간에 영향이 미치지 않도록 스레드 세이프 로직, 변수를 적용해야 한다.

컨슈머를 멀티 스레드로 활용하는 방식은 크게 두 가지로 나뉜다.

첫 번째는 컨슈머 스레드는 1개만 실행하고 데이터 처리를 담당하는 워커 스레드를 여러 개 실행하는 방법인 `멀티 워커 스레드 전략`이다.
두 번째는 컨슈머 인스턴스에서 poll() 메서드를 호출하는 스레드르 여러 개 띄워서 사용하는 `컨슈머 멀티 스레드 전략`이다.

### 카프카 컨슈머 멀티 워커 스레드 전략
브로커로부터 전달받은 레코드들을 병렬로 처리한다면 1개의 컨슈머 스레드로 받은 데이터들을 더욱 향상된 속도로 처리할 수 있다.

멀티 스레드를 생성하는 ExecutorService 자바 라이브러리를 사용하면 레코드를 병렬처리하는 스레드를 효율적으로 생성하고 관리할 수 있는데, 데이터 처리 환경에 맞는 스레드 풀을 사용하면 된다.
작업 이후 스레드가 종료되어야 한다면 CachedThreadPool을 사용하여 스레드를 실행한다.
이제 CachedThreadPool을 사용하여 멀티 스레드를 생성하고 레코드를 병렬로 처리하는 코드를 살펴보자.

### ConsumerWorker 구현 클래스
```java
public class ConsumerWorker Implements Runnable {
    private final static Logger logger = LoggerFactory.getLogger(ConsumerWorker.class);
    private String recordValue;
    
    public ConsumerWorker(String recordValue) {
        this.recordValue = recordValue;
    }
    
    @Override
    public void run() {
        logger.info("thread: {}\trecord:{}", Thread.currentThread().getName(), recordValue);
    }
}
```

### ConsumerWorker 호출 부분
```java
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(configs); // 카프카 컨슈머 인스턴스 생성
consumer.subscribe(Arrays.asList(TOPIC_NAME)); // 토픽 구독
ExecutorService executorService = Executors.newCachedThreadPool(); // 스레드 풀 생성

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(10)); // 레코드 가져오기
    for (ConsumerRecord<String, String> record : records) {
        ConsumerWorker worker = new ConsumerWorker(record.value()); // 워커 스레드 생성
        executorService.execute(worker); // 워커 스레드 실행
    }
}
```
```shell
# 실행 결과
[pool-1-thread-1] INFO ConsumerWorker - thread: pool-1-thread-1 record: hello  
[pool-1-thread-2] INFO ConsumerWorker - thread: pool-1-thread-2 record: world
```

스레드를 사용하면 한번 poll()을 통해 받은 데이터를 병렬처리함으로써 속도의 이점을 확실히 얻을 수 있다. 그러나 몇가지 주의해야 하는 사항이 있다.

첫번째는 스레드를 사용함으로써 데이터 처리가 끝나지 않았음에도 불구하고 커밋을 하기 때문에 리밸런싱, 컨슈머 자애 시에 `데이터 유실`이 발생할 수 있다는 것이다.
두번째는 `레코드 처리의 역전현상`이다. 레코드별로 스레드의 생성은 순서대로 진행되지만 스레드의 처리시간이 달라서 나중에 생성된 스레드가 먼저 처리 될 수도 있다.

멀티 워커 스레드 전략은 레코드 처리에 있어 중복이 발생하거나 데이터의 역전현상이 발생해도 되며 매우 빠른 처리 속도가 필요한 데이터 처리에 적합하다. 
서버 리소스 모니터링 파이프라인, IOT 서비스의 센서 데이터 수집 파이프라인이 그 예이다.

### 카프카 컨슈머 멀티 스레드 전략
컨슈머 스레드를 늘려서 여러 파티션을 동시에 처리하는 전략이다.
우선 컨슈머 스레드 클래스를 Runnable 인터페이스를 구현하여 생성한다.

```java
public class ConsumerWorker Implements Runnable {
    private final static Logger logger = LoggerFactory.getLogger(ConsumerWorker.class);
    private Properties prop;
    private String topic;
    private String threadName;
    private KafkaConsumer<String, String> consumer;
    
    public ConsumerWorker(Properties prop, String topic, int number) {
        this.prop = prop;
        this.topic = topic;
        this.threadName = "consumer-thread-" + number;
    }
    
    @Override
    public void run() {
        consumer = new KafkaConsumer<>(prop);
        consumer.subscribe(Arrays.asList(topic));
        
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1));
            for (ConsumerRecord<String, String> record : records) {
                logger.info({}, record);
            }
        }
    }
}
```

멀티 컨슈머 스레드를 가진 애플리케이션을 실행한다.
```java
public class MultiConsumerThread {
    
    private final static String TOPIC_NAME = "test";
    private final static String GROUP_ID = "test-group";
    private final static String BOOTSTRAP_SERVERS = "localhost:9092";
    private final static int CONSUMER_THREAD_COUNT = 3;
    
    public static void main(String[] args) {
        Properties configs = new Properties();
        configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        configs.put(ConsumerConfig.GROUP_ID_CONFIG, GROUP_ID);
        configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        
        ExecutorService executorService = Executors.newCachedThreadPool();
        
        for (int i = 0; i < CONSUMER_THREAD_COUNT; i++) {
            ConsumerWorker worker = new ConsumerWorker(configs, TOPIC_NAME, i);
            executorService.execute(worker);
        }
    }
}
```

## 4.3.2 컨슈머 랙
컨슈머 랙은 토픽의 최신 오프셋과 컨슈머 오프셋간의 차이다. 컨슈머 랙은 컨슈머가 정상 동작하는지 여부를 확인할 수 있기 때문에 애플리케이션을 운영한다면 필수적으로 모니터링해야 하는 지표이다.
컨슈머 렉을 모니터링함으로써 컨슈머의 장애를 확인할 수 있고 파티션 개수를 정하는 데에 참고할 수 있다.

컨슈머 랙은 컨슈머 그룹과 토픽, 파티션별로 생성된다. 
1개의 토픽에 3개의 파티션이 있고 1개의 컨슈머 그룹이 토픽을 구독하여 데이터를 가져가면 컨슈머 랙은 총 3개가 된다.

컨슈머 랙을 확인하는 방법은 총 세가지가 있다.
1. 카프카 명령어를 사용하는 방법
2. 컨슈머 애플리케이션에서 metrics() 메서드를 사용하는 방법
3. 외부 모니터링 툴을 사용하는 방법

### 카프카 명령어를 사용하여 컨슈머 랙 조회
kafka-consumer-groups.sh 스크립트를 사용하여 컨슈머 랙을 조회할 수 있다.
```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group test-group --describe
```
```shell
# 실행 결과
TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
test-topic            0          2             5             3               consumer-01    /127.0.0.1      consumer-1
test-topic            1          3             5             2               consumer-01    /127.0.0.1      consumer-2
test-topic            2          4             5             1               consumer-01    /127.0.0.1      consumer-3
```

### 컨슈머 metrics() 메서드를 사용하여 컨슈머 랙 조회
컨슈머 애플리케이션에서 KafkaConsumer 인스턴스의 metrics() 메서드를 사용하여 컨슈머 랙을 조회할 수 있다.
컨슈머 인스턴스가 제공하는 컨슈머 랙 관련 모니터링 지표는 3가지로 records-rag-max, records-lag, records-lag-avg이다.

```java
for (Map.Entry<MetricName, ? extends Metric> entry : consumer.metrics()
.entrySet()) {
    if(entry.getKey().name().equals("records-lag")||
            entry.getKey().name().equals("records-lag-avg")||
            entry.getKey().name().equals("records-lag-max")) {
        Metric metric = entry.getValue();
        logger.info("metricName: {}, metricValue: {}", entry.getKey().name(), metric.metricValue());
    }
}
```
metrics() 메서드로 컨슈머 랙을 확인하는 방법은 3가지 문제점이 있다.
1. 컨슈머가 정상 동작할 경우에만 확인할 수 있다.
2. 모든 컨슈머 애플리케이션에 컨슈머 랙 모니터링 코드를 중복해서 작성해야 한다.
3. 컨슈머 랙을 모니터링하는 코드를 추가할 수 없는 카프카 서드 파티 애플리케이션의 컨슈머 랙 모니터링이 불가능하다.

### 외부 모니터링 툴을 사용하여 컨슈머 랙 조회
컨슈머 랙을 모니터링하는 가장 최선의 방법은 외부 모니터링 툴을 사용하는 것이다.
데이터 독, 컨플루언트 컨트롤 센터와 같은 카프카 클러스터 종합 모니터링 툴을 사용하면 카프카 운영에 필요한 다양한 지표를 모니터링 할 수 있다.

## 4.3.2.1 카프카 버로우
카프카 버로우는 링크드인에서 공개한 오픈소스 컨슈머 랙 체크 툴이다. 버로우를 카프카 클러스터와 연동하면 REST API를 통해 컨슈머 그룹별 컨슈머 랙을 조회할 수 있다.

|요청 메서드|호출 경로|설명|
|---|---|---|
|GET|/burrow/admin                                     |버로우 헬스 체크|
|GET|/v3/kafka	                                       |버로우와 연동 중인 카프카 클러스터 리스트|
|GET|/v3/kafka/{클러스터 이름}                            |클러스터 정보 조회|
|GET|/v3/kafka/{클러스터 이름}/consumer                   |클러스터에 존재하는 컨슈머 그룹 리스트|
|GET|/v3/kafka/{클러스터 이름}/topic                      |클러스터에 존재하는 토픽 리스트|
|GET|/v3/kafka/{클러스터 이름}/consumer/{컨슈머 그룹 이름}    |컨슈머 그룹의 컨슈머 랙, 오프셋 정보 조회|
|GET|/v3/kafka/{클러스터 이름}/consumer/{컨슈머 그룹 이름}/lag|컨슈머 그룹의 파티션 정보, 상태, 컨슈머 랙 조회|
|GET|/v3/kafka/{클러스터 이름}/topic/{토픽 이름}            |토픽 상세 조회|

버로우는 다수의 카프카 클러스터를 동시에 연결하여 컨슈머 랙을 확인한다.
그래서 한 번의 설정으로 다수의 카프카 클러스터 컨슈머 랙을 확인할 수 있다는 장점이 있다.
그러나 모니터링을 위해 컨슈머 랙 지표를 수집, 적재, 알람 설정을 하고 싶다면 별도의 저장소와 대시보드를 구축해야 한다.

컨슈머 애플리케이션을 운영할 때 컨슈머 랙이 임계치에 도달할 때마다 알람을 받는 것은 무의미한 일이다.
그래서 버로우에서는 임계치가 아닌 슬라이딩 윈도우 계산을 통해 문제가 생긴 파티션과 컨슈머의 상태를 표현한다.
이렇게 버로우에서 컨슈머 랙의 상태를 표현하는 것을 `컨슈머 랙 평가`라고 부른다.

### 상황별 컨슈머와 파티션의 상태
1. 파티션 OK, 컨슈머 OK 상태
   ![image](https://github.com/user-attachments/assets/b9431c13-fc9f-42f2-8895-f54d3bbe1a7c)
2. 파티션 OK, 컨슈머 WARNING 상태
   ![image](https://github.com/user-attachments/assets/ea829dd1-ed7e-40dc-b837-df4d4ad8fa40)
3. 파티션 STALLED, 컨슈머 ERROR 상태
   ![image](https://github.com/user-attachments/assets/fd2d904f-32d4-4653-9cf2-07a110c32907)

### 컨슈머 랙 모니터링 아키텍쳐 준비물
- 버로우
  - REST API를 통해 컨슈머 랙을 조회할 수 있다.
- 텔레그래프
  - 데이터 수집 및 전달에 특화된 툴. 버로우를 조회하여 데이터를 엘라스틱 서치에 전달한다.
- 엘라스틱서치
  - 컨슈머 랙 정보를 담는 저장소
- 그라파나
  - 엘라스틱 서치의 정보를 시각화하고 특정 조건에 따라 슬랙 알람을 보낼 수 있는 웹 대시보드 툴

## 4.3.3 컨슈머 배포 프로세스
카프카 컨슈머 애플리케이션 운영 시 로직 변경으로 인한 배포를 하는 방법은 크게 두 가지가 있다.
첫 번째는 중단 배포이고 두 번째는 무중단 배포이다.

### 중단 배포
중단 배포는 컨슈머 애플리케이션을 완전히 종료한 이후에 개선된 코드를 가진 애플리케이션을 배포하는 방식이다.
이 방법은 한정된 서버 자원을 운영하는 기업에 적합하다.

또한 중단 배포를 사용할 경우 새로운 로직이 적용된 신규 애플리케이션의 실행 전후를 명확하게 특정 오프셋 지점으로 나눌 수 있다는 점이 장점이다.
컨슈머 랙과 파티션별 오프셋을 기록하는 버로우를 사용하거나 배포 시점의 오프셋을 로깅한다면 신규 로직 적용 전후의 데이터를 명확히 구분할 수 있다.

### 무중단 배포
컨슈머의 중단이 불가능한 애플리케이션의 신규 로직 배포가 필요할 경우에는 무중단 배포가 필요하다.
무중단 배포는 3가지 방법으로 블루/그린, 롤링 그리고 카나리 배포가 있다.
1. 블루/그린 배포
   - 블루/그린 배포는 두 개의 환경을 동시에 띄어놓고 배포하는 방식이다.
   - 블루 환경에서는 기존 애플리케이션이 동작하고 그린 환경에서는 신규 애플리케이션이 동작한다.
   - 블루 환경에서는 데이터를 처리하지 않고 그린 환경에서 데이터를 처리한다.
   - 그린 환경에서 데이터 처리가 완료되면 블루 환경을 그린 환경으로 전환한다.
2. 롤링 배포
   - 롤링 배포는 파티션 별로 순차적으로 중단시키면서 배포하는 방식이다.
   - 파티션 개수가 많을수록 리밸런스 시간도 길어지므로 파티션 개수가 많지 않은 경우에 효과적인 방법이다.
3. 카나리 배포
    - 카나리 배포는 일부 파티션에 컨슈머를 따로 배정하여 일부 데이터에 대해 신규 버전의 애플리케이션이 우선적으로 처리하는 방식으로 테스트
    - 문제가 없으면 나머지 파티션에 할당된 컨슈머는 롤링 또는 블루/그린 배포를 수행하여 무중단 배포가 가능하다.
