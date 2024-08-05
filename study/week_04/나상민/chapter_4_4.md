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