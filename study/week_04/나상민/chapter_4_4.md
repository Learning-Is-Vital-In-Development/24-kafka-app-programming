# 4.4 스프링 카프카
스프링 카프카는 카프카를 스프링 프레임워크에서 효과적으로 사용할 수 있도록 만들어진 라이브러리이다.
## 4.4.1 스프링 카프카 프로듀서
스프링 카프카 프로듀서는 `카프카 템플릿`이라고 불리는 클래스를 사용하여 데이터를 전송할 수 있다.
카프카 템플릿은 프로듀서 팩토리 클래스를 통해 생성할 수 있다.
카프카 템플릿을 사용하는 방법은 크게 두 가지가 있다. 첫 번째는 스프링 카프카에서 제공하는 기본 카프카 템플릿을 사용하는 방법이고 
두 번째는 직접 사용자가 카프카 템플릿을 프로듀서 팩토리로 생성하는 방법이다.

### 기본 카프카 템플릿
기본 카프카 템플릿은 기본 프로듀서 팩토리를 통해 생성된 카프카 템플릿을 사용한다. 기본 카프카 템플릿을 사용할 때는 application.yaml에 프로듀서 옵션을 넣고 사용할 수 있다.

카프카 템플릿은 카프카 클라이언트 라이브러리의 필수 값인 `bootstrap-servers`, `key-serializer`, `value-serializer`값을 넣지 않아도 기본 값이 넣어져 있다.

- application.yaml
```yaml
spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      acks: all
```

- SpringProducerApplication.java
```java
@SpringBootApplication
public class SpringProducerApplication implements CommandLineRunner {
    
    private final String TOPIC_NAME = "test";
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate; // application.yaml에 설정한 값이 자동 주입된다.

    public static void main(String[] args) {
        SpringApplication application = SpringApplication.run(SpringProducerApplication.class, args);
        application.run(args);
    }

    @Override
    public void run(String... args) throws Exception {
        for (int i = 0; i < 10; i++) {
            kafkaTemplate.send(TOPIC_NAME, "test" + i);
        }
        System.exit(0);
    }
}
```

### 커스텀 카프카 템플릿
커스텀 카프카 템플릿은 프로듀서 팩토리를 통해 만든 카프카 템플릿 객체를 빈으로 등록하여 사용하는 것이다.
프로듀서에 필요한 각종 옵션을 선언하여 사용할 수 있으며 한 스프링 카프카 애플리케이션 내부에 다양한 종류의 카프카 프로듀서 인스턴스를 생성하고 싶다면 이 방식을 사용하면 된다.

- KafkaTemplateConfiguration.java
```java

@Configuration
public class KafkaTemplateConfiguration {
    
    @Bean
    public KafkaTemplate<String, String> customKafkaTemplate() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configs.put(ProducerConfig.ACKS_CONFIG, "all");
        configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        
        ProducerFactory<String,String> pf = new DefaultKafkaProducerFactory<>(configs);
        
        return new KafkaTemplate<>(pf);
    }
    
    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

- SpringProducerApplication.java

```java
import java.util.List;

@SpringBootApplication
public class SpringProducerApplication implements CommandLineRunner {

    private final String TOPIC_NAME = "test";

    @Autowired
    private KafkaTemplate<String, String> customKafkaTemplate;

    public static void main(String[] args) {
        SpringApplication application = SpringApplication.run(SpringProducerApplication.class, args);
        application.run(args);
    }

    @Override
    public void run(String... args) throws Exception {
        ListenableFuture<SendResult<String, String>> future = customKafkaTemplate.send(TOPIC_NAME, "test");
        future.addCallback(new ListenableFutureCallback<SendResult<String, String>>() {
            @Override
            public void onFailure(Throwable ex) {
                System.out.println("Failed to send message : " + ex.getMessage());
            }

            @Override
            public void onSuccess(SendResult<String, String> result) {
                System.out.println("Sent message : " + result.getRecordMetadata());
            }
        });
    }
}
```

## 4.4.2 스프링 카프카 컨슈머
스프링 카프카의 컨슈머는 기존 컨슈멀를 2개의 타입으로 나누고 커밋을 7가지로 나누어 세분화했다.
우선, 타입은 레코드 리스너와 배치 리스너가 있다.

레코드 리스너는 단 1개의 레코드를 처리한다. 반면, 배치 리스너는 기존 카프카 클라이언트 라이브러리의 poll() 메서드로 리턴받은 ConsumerRecords처럼 한 번에 여러개 레코드들을 처리할 수 있다.

### 메시지 리스너 종류와 구현에 필요한 파라미터(Record)
| 리스너 이름         | 생성 메서드 파라미터                                                                                                                         | 상세 설명                                                     |
|----------------|-------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| MessageListener| onMessage(ConsumerRecord<K, V> data)<br/>onMessage(V data)                                                                               | Record 인스턴스 단위로 프로세싱, 오토 커밋 또는 컨슈머 컨테이너의 AckMode를 사용하는 경우 |
|AcknowledgingMessageListener| onMessage(ConsumerRecord<K, V> data, Acknowledgment acknowledgment)<br/>onMessage(V data, Acknowledgment acknowledgment)| Record 인스턴스 단위로 프로세싱, 매뉴얼 커밋을 사용하는 경우                     |
|ConsumerAwareMessageListener|	onMessage(ConsumerRecord<K, V> data, Consumer<K, V> consumer)<br/>onMessage(V data, Consumer<K, V> consumer)| Record 인스턴스 단위로 프로세싱, 컨슈머 객체를 활용하고 싶은 경우                  |
|AcknowledgingConsumerAwareMessageListener|	onMessage(ConsumerRecord<K, V> data, Acknowledgment acknowledgment, Consumer<?, ?> consumer)<br/>onMessage(V data, Acknowledgment acknowledgment, Consumer<K, V> consumer)| Record 인스턴스 단위로 프로세싱, 매뉴얼 커밋을 사용하고 컨슈머 객체를 활용하고 싶은 경우     |

### 메시지 리스너 종류와 구현에 필요한 파라미터(Batch)
| 리스너 이름                                        | 생성 메서드 파라미터                                                                                                                                                                        | 상세 설명                                                     |
|-----------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| BatchMessageListener                          | onMessage(ConsumerRecords<K, V> data)<br/>onMessage(List<V> data)                                                                                                                  | Record 인스턴스 단위로 프로세싱, 오토 커밋 또는 컨슈머 컨테이너의 AckMode를 사용하는 경우 |
| BatchAcknowledgingMessageListener             | onMessage(ConsumerRecords<K, V> data, Acknowledgment acknowledgment)<br/>onMessage(List<V> data, Acknowledgment acknowledgment)                                                    | Record 인스턴스 단위로 프로세싱, 매뉴얼 커밋을 사용하는 경우                     |
| BatchConsumerAwareMessageListener             | 	onMessage(ConsumerRecords<K, V> data, Consumer<K, V> consumer)<br/>onMessage(List<V> data, Consumer<K, V> consumer)                                                               | Record 인스턴스 단위로 프로세싱, 컨슈머 객체를 활용하고 싶은 경우                  |
| BatchAcknowledgingConsumerAwareMessageListener | 	onMessage(ConsumerRecords<K, V> data, Acknowledgment acknowledgment, Consumer<?, ?> consumer)<br/>onMessage(List<V> data, Acknowledgment acknowledgment, Consumer<K, V> consumer) | Record 인스턴스 단위로 프로세싱, 매뉴얼 커밋을 사용하고 컨슈머 객체를 활용하고 싶은 경우     |

카프카 컨슈머에서 커밋을 직접 구현할 때는 오토 커밋, 동기 커밋, 비동기 커밋 크게 세가지로 나뉘지만 실제 운영환경에서는 다양한 종류의 커밋을 구현해서 사용한다.
그러나 스프링 카프카에서는 사용자가 사용할 만한 커밋의 종류를 7가지(RECORD,VATCH,TIME,COUNT,COUNT_TIME,MANUAL,MANUAL_IMMEDIATE)로 나누어 제공한다.
스프링 카프카에서는 커밋이라고 부르지 않고 'AckMode'라고 부른다.

### AckMode 종류
| AckMode 이름 | 설명                                                                                                                                                                                                        |
|-------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| RECORD      | 레코드 단위로 프로세싱 이후 커밋                                                                                                                                                                                        |
| BATCH       | poll() 메서드로 호출된 레코드가 모두 처리된 이후 커밋<br/>스프링 카프카 컨슈머의 AckMode 기본값                                                                                                                                            |
| TIME        | 특정시간 이후에 커밋<br/>이 옵션을 사용할 경우에는 시간 간격을 선언하는 AckTime 옵션을 설정해야 한다.                                                                                                                                           |
| COUNT       | 특정 개수만큼 레코드가 처리된 이후에 커밋<br/>이 옵션을 사용할 경우에는 레코드 개수를 선언하는 AckCount 옵션을 설정해야 한다                                                                                                                              |
| COUNT_TIME  | TIME, COUNT 옵션 중 맞는 조건이 하나라도 나올 경우 커밋                                                                                                                                                                     |
| MANUAL      | Acknowledgement.acknowledge() 메서드가 호출되면 다음번 poll()때 커밋을 한다. 매번 acknowledge() 메서드를 호출하면 BATCH 옵션과 동일하게 동작한다. 이 옵션을 사용할 경우에는 AcknowledgingMessageListener 또는 BatchAcknowledgingMessageListener를 리스너로 사용해야한다 |
| MANUAL_IMMEDIATE | Acknowledgement.acknowledge()메서드를 호출한 즉시 커밋한다. 이 옵션을 사용할 경우에는 AcknowledgingMessageListener 또는 BatchAcknowledgingMessageListener를 리스너로 사용해야 한다.                                                            |

리스너를 생성하고 사용하는 방식은 크게 두 가지가 있다. 첫 번째는 기본 리스너 컨테이너를 사용하는 것이고 두 번째는 컨테이너 팩토리를 사용하여 직접 리스너를 만드는 것이다.

### 기본 리스너 컨테이너

### record 리스너 예제
- application.yaml
```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost
    listener:
      ack-mode: MANUAL_IMMEDIATE
      type: RECORD
```
- SpringConsumerApplication.java
```java
@SpringBootApplication
public class SpringConsumerApplication {
	public static Logger logger = LoggerFactory.getLogger(SpringConsumerApplication.class);


	public static void main(String[] args) {
		SpringApplication application = new SpringApplication(SpringConsumerApplication.class);
		application.run(args);
	}

	// 기본적인 리스너 선언
	@KafkaListener(topics = "test",
			groupId = "test-group-00")
	public void recordListener(ConsumerRecord<String,String> record) {
		logger.info(record.toString());
	}

	// 메시지 값을 파라미터로 받는 리스너
	@KafkaListener(topics = "test",
			groupId = "test-group-01")
	public void singleTopicListener(String messageValue) {
		logger.info(messageValue);
	}

    // 개별 리스너에 카프카 컨슈머 옵션값을 부여하고 싶다면 properties를 사용한다.
	@KafkaListener(topics = "test",
			groupId = "test-group-02", properties = {
			"max.poll.interval.ms:60000",
			"auto.offset.reset:earliest"
	})
	public void singleTopicWithPropertiesListener(String messageValue) {
		logger.info(messageValue);
	}

	// 2개 이상의 카프카 컨슈머 스레드를 실행하고 싶다면 concurrency 옵션을 사용하면 된다.
	@KafkaListener(topics = "test",
			groupId = "test-group-03",
			concurrency = "3")
	public void concurrentTopicListener(String messageValue) {
		logger.info(messageValue);
	}

    // 특정 파티션을 리스닝하고 싶다면 topicPartitions를 사용한다.
	@KafkaListener(topicPartitions =
			{
					@TopicPartition(topic = "test01", partitions = {"0", "1"}),
					@TopicPartition(topic = "test02", partitionOffsets = @PartitionOffset(partition = "0", initialOffset = "3"))
			},
			groupId = "test-group-04")
	public void listenSpecificPartition(ConsumerRecord<String, String> record) {
		logger.info(record.toString());
	}
}
```

### batch 리스너 예제
배치 리스너는 레코드 리스너와 다르게 메서드의 파라미터를 List또는 ConsumerRecords로 받는다.

- application.yaml
```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost
    listener:
      ack-mode: MANUAL_IMMEDIATE
      type: BATCH
```

- SpringConsumerApplication.java
```java
@SpringBootApplication
public class SpringConsumerApplication {
    public static Logger logger = LoggerFactory.getLogger(SpringConsumerApplication.class);

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(SpringConsumerApplication.class);
        application.run(args);
    }

    @KafkaListener(topics = "test",
            groupId = "test-group-01")
    public void batchListener(ConsumerRecords<String, String> records) {
        records.forEach(record -> logger.info(record.toString()));
    }

    @KafkaListener(topics = "test",
            groupId = "test-group-02")
    public void batchListener(List<String> list) {
        list.forEach(recordValue -> logger.info(recordValue));
    }

    @KafkaListener(topics = "test",
            groupId = "test-group-03",
            concurrency = "3")
    public void concurrentBatchListener(ConsumerRecords<String, String> records) {
        records.forEach(record -> logger.info(record.toString()));
    }

}
```