---
layout: post
title: "Spring Boot에서 RabbitMQ 연동"
subtitle: "Spring Boot에서 RabbitMQ 연동"
categories: ["Spring Boot"]
tags: ["Spring Boot"]
sidebar: ['article-menu']
---

## RabbitMQ 연동?

### Rabbitmq 실행을 위한 docker compose 작성
```yaml
version: "3"
services:
  rabbitmq:
    image: rabbitmq:latest
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123!@#
```

### build.gradle 파일 수정

```
implementation 'org.springframework.boot:spring-boot-starter-amqp'
```

### resource/application.yaml 수정
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: admin123!@#
```

### Producer를 위한 설정 코드 작성
json 처리에 사용될 ObjectMapper와 Producer 설정을 추가합니다

```java
// json 처리에 사용될 ObjectMapper 설정
@Configuration
public class ObjectMapperConfig {

  @Bean
  public ObjectMapper objectMapper(){
    var objectMapper = new ObjectMapper();
    objectMapper.registerModule(new Jdk8Module());
    objectMapper.registerModule(new JavaTimeModule());
    objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    objectMapper.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
    objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    objectMapper.setPropertyNamingStrategy(new PropertyNamingStrategies.SnakeCaseStrategy());
    return objectMapper;
  }
}

...
// rabbitmq 설정
@Configuration
public class RabbitMqConfig {

    @Bean
    public DirectExchange directExchange() {
        return new DirectExchange("delivery.exchange");
    }

    @Bean
    public Queue queue(){
        return new Queue("delivery.queue");
    }

    @Bean
    public Binding binding(DirectExchange exchange, Queue queue){
        return BindingBuilder.bind(queue).to(exchange).with("delivery.key");
    }

    @Bean
    public RabbitTemplate rabbitTemplate(
        ConnectionFactory connectionFactory, //application.yaml 정보로 구성됨
        MessageConverter messageConverter
    ){
        var rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMessageConverter(messageConverter);
        return rabbitTemplate;
    }

    @Bean
    public MessageConverter messageConverter(
        ObjectMapper objectMapper
    ){
        return new Jackson2JsonMessageConverter(objectMapper);
    }
}

...
// 메시지 발행 코드 작성
@RequiredArgsConstructor
@Component
public class Producer {

  private final RabbitTemplate rabbitTemplate;

  public void producer(String exchange, String routeKey, Object object){
    rabbitTemplate.convertAndSend(exchange, routeKey, object);
  }
}
```

### Producer 클래스 추가
위에서 작성한 Producer를 호출하여 메시지를 추가합니다
```java
@RequiredArgsConstructor
@Service
public class UserOrderProducer {

    private final Producer producer;

    public void sendOrder(Long userOrderId){
        var message = UserOrderMessage.builder()
            .userOrderId(userOrderId)
            .build();
        producer.producer(
            "delivery.exchange", "delivery.key", message
        );
    }
}
```

### Consumer를 위한 설정 코드 작성
```java
@Configuration
public class RabbitMqConfig {
    @Bean
    public MessageConverter messageConverter(ObjectMapper objectMapper){
        return new Jackson2JsonMessageConverter(objectMapper);
    }
}
```

### Consumer 클래스 추가
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class UserOrderConsumer {

    private final UserOrderBusiness userOrderBusiness;

    @RabbitListener(queues = "delivery.queue")
    public void userOrderConsumer(
        UserOrderMessage userOrderMessage
    ){
        log.info(userOrderMessage);
        // do something
    }
}
```
