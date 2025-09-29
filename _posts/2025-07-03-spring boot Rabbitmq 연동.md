---
layout: post
title: "Spring Boot - RabbitMQ 연동하기"
subtitle: "RabbitMQ 개념부터 실제 연동, 관리자 페이지 확인까지"
categories: ["Spring Boot"]
tags: ["spring-boot", "rabbitmq", "message-queue", "amqp"]
sidebar: ['article-menu']
---

Spring Boot로 웹 애플리케이션을 만들다 비동기 처리가 필요합니다. 이때 주로 사용되는 RabbitMQ 사용 방법을 정리합니다.


### 1.핵심 개념

- Producer (생산자): 메시지를 만들어 우체국에 보내는 발신인.
- Consumer (소비자): 우편함에 도착한 메시지를 가져와서 처리하는 수신인.
- Exchange (교환기): 우체국의 분류 담당자. Producer에게 받은 메시지를 어떤 우편함으로 보낼지 규칙(Routing Key)에 따라 결정합니다.
- Queue (큐): 우편함. 메시지가 Consumer에게 전달되기 전까지 안전하게 보관되는 장소.

### 2. Docker로 RabbitMQ 실행하기

```yaml
version: "3"
services:
  rabbitmq:
    image: rabbitmq:latest # 최신 RabbitMQ 이미지를 사용합니다.
    ports:
      - "5672:5672"     # 애플리케이션이 접속할 포트
      - "15672:15672"   # 관리자 웹 페이지 접속 포트
    environment:
      - RABBITMQ_DEFAULT_USER=admin      # 관리자 아이디
      - RABBITMQ_DEFAULT_PASS=admin123!@# # 관리자 비밀번호
```
터미널에서 `docker-compose up -d` 명령어를 실행하면 RabbitMQ가 배경에서 실행됩니다.

### 3. Spring Boot 프로젝트 설정

`build.gradle` 파일을 수정합니다. 라이브러리를 추가합니다.
```groovy
implementation 'org.springframework.boot:spring-boot-starter-amqp'
```

`application.yaml` 파일을 수정합니다. RabbitMQ 정보를 입력합니다.
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: admin123!@#
```

### 4. RabbitMQ 설정

이제 코드를 통해 Exchange와 Queue를 만들고 둘을 연결해주는 작업을 진행합니다.

```java
@Configuration
public class RabbitMqConfig {

    // "delivery.exchange"라는 이름의 Exchange를 만듭니다.
    @Bean
    public DirectExchange directExchange() {
        return new DirectExchange("delivery.exchange");
    }

    // "delivery.queue"라는 이름의 Queue를 만듭니다.
    @Bean
    public Queue queue(){
        return new Queue("delivery.queue");
    }

    // 위에서 만든 Exchange와 Queue를 "delivery.key"라는 주소(라우팅 키)로 연결합니다.
    // 이제 "delivery.key"로 온 메시지는 "delivery.queue"에 쌓이게 됩니다.
    @Bean
    public Binding binding(DirectExchange exchange, Queue queue){
        return BindingBuilder.bind(queue).to(exchange).with("delivery.key");
    }

    // 메시지를 RabbitMQ로 보낼 때 RabbitTemplate를 설정합니다.
    @Bean
    public RabbitTemplate rabbitTemplate(
        ConnectionFactory connectionFactory,
        MessageConverter messageConverter
    ){
        var rabbitTemplate = new RabbitTemplate(connectionFactory);
        // 메시지를 JSON 형태로 보내기 위해 MessageConverter를 설정합니다.
        rabbitTemplate.setMessageConverter(messageConverter);
        return rabbitTemplate;
    }

    // Spring이 메시지를 JSON으로 변환할 때 사용할 MessageConverter입니다.
    @Bean
    public MessageConverter messageConverter(ObjectMapper objectMapper){
        return new Jackson2JsonMessageConverter(objectMapper);
    }
}
```

### 5. 메시지 보내고 받기 (Producer & Consumer)

이제 Producer과 Consumer을 만들어 메시지를 주고받아 보겠습니다.
`RabbitTemplate` 도구를 사용하여 메시지를 보내는 코드입니다.
```java
@RequiredArgsConstructor
@Component
public class Producer {

  private final RabbitTemplate rabbitTemplate;

  // exchange, routeKey, 그리고 보낼 메시지(object)를 받아 메시지를 보냅니다.
  public void producer(String exchange, String routeKey, Object object){
    rabbitTemplate.convertAndSend(exchange, routeKey, object);
  }
}

// 실제 서비스 로직에서 Producer를 호출하는 예시
@RequiredArgsConstructor
@Service
public class UserOrderProducer {

    private final Producer producer;

    public void sendOrder(Long userOrderId){
        var message = UserOrderMessage.builder()
            .userOrderId(userOrderId)
            .build();
        // "delivery.exchange"를 목적지로, "delivery.key"라는 주소를 붙여 메시지를 보냅니다.
        producer.producer("delivery.exchange", "delivery.key", message);
    }
}
```

메시지 수신 처리입니다. `@RabbitListener` 어노테이션은 특정 큐를 항상 지켜보다가, 새 메시지가 오면 자동으로 가져와서 처리합니다.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class UserOrderConsumer {

    @RabbitListener(queues = "delivery.queue") // "delivery.queue" 우편함을 주시합니다.
    public void userOrderConsumer(UserOrderMessage userOrderMessage) {
        log.info("새로운 주문 메시지를 받았습니다: {}", userOrderMessage);
        // TODO: 전달받은 메시지를 가지고 실제 비즈니스 로직 처리
    }
}
```

### 6. RabbitMQ 관리자 페이지

웹 브라우저에서 `http://localhost:15672` 로 접속하고, `docker-compose.yml`에 설정한 아이디(`admin`)와 비밀번호(`admin123!@#`)로 로그인합니다.

- Exchanges, Queues 탭을 확인하면 `delivery.exchange`와 `delivery.queue`가 실제로 생성된 것을 볼 수 있습니다.
- 메시지를 보내는 API를 호출한 뒤 Queues 탭에서 `delivery.queue`를 클릭하면 `Ready` 숫자가 올라가고, 잠시 후 Consumer가 메시지를 처리하면 `0`으로 바뀌는 것을 확인할 수 있습니다.

