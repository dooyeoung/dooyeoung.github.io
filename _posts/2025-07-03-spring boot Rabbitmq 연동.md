---
layout: post
title: "Spring Boot - RabbitMQ 연동하기"
subtitle: "RabbitMQ 개념부터 실제 연동, 관리자 페이지 확인까지"
categories: ["Spring Boot"]
tags: ["spring-boot", "rabbitmq", "message-queue", "amqp"]
sidebar: ['article-menu']
---

Spring Boot로 웹 애플리케이션을 만들다 보면, "이 작업은 시간이 좀 걸리는데, 사용자를 계속 기다리게 해야 하나?" 싶은 순간이 있습니다. 예를 들어, 사용자가 주문을 완료했을 때, 주문 접수 처리, 재고 감소, 배송 시스템에 정보 전달 등 여러 작업이 필요하죠. 이 모든 과정이 끝날 때까지 사용자가 로딩 화면만 보고 있다면 좋은 경험이 아닐 겁니다.

이럴 때 등장하는 해결사가 바로 **RabbitMQ**와 같은 메시지 큐입니다.

### 1. RabbitMQ, 왜 사용하는 건가요?

RabbitMQ를 한마디로 표현하면 **'똑똑한 우체국'**입니다. 보내는 사람(Producer)이 받는 사람(Consumer)의 주소를 직접 몰라도, 우체국에 메시지를 맡기기만 하면 알아서 정확하고 안전하게 전달해주죠.

![RabbitMQ Flow](https://i.imgur.com/sJ8xS3j.png) <!-- 이미지 예시 -->

이 '우체국'을 사용하면 다음과 같은 엄청난 장점이 생깁니다.

- **비동기 처리**: 시간이 걸리는 작업을 RabbitMQ에 "처리해줘!" 하고 맡겨두고, 사용자는 즉시 다른 작업을 계속할 수 있습니다. 서비스 응답 속도가 빨라져 사용자 경험이 향상됩니다.
- **애플리케이션 간의 결합도 감소**: '주문 서비스'와 '배송 서비스'가 직접 대화하는 대신, RabbitMQ라는 중간 창구를 통해 대화합니다. 덕분에 한쪽 서비스가 잠시 아파도(장애가 발생해도) 다른 서비스에 영향이 가지 않아, 전체 시스템이 훨씬 안정적으로 운영됩니다.

### 2. 이것만은 알고 가세요! RabbitMQ 핵심 개념

코드를 보기 전에, 이 네 가지 용어만 기억하면 훨씬 이해하기 쉬워집니다.

- **Producer (생산자)**: 메시지를 만들어 우체국에 보내는 **'발신인'**.
- **Exchange (교환기)**: 우체국의 **'분류 담당자'**. Producer에게 받은 메시지를 어떤 우편함으로 보낼지 규칙(Routing Key)에 따라 결정합니다.
- **Queue (큐)**: **'우편함'**. 메시지가 Consumer에게 전달되기 전까지 안전하게 보관되는 장소.
- **Consumer (소비자)**: 우편함에 도착한 메시지를 가져와서 처리하는 **'수신인'**.

### 3. 사전 준비: Docker로 RabbitMQ 실행하기

가장 먼저 우리 컴퓨터에 RabbitMQ 우체국을 설치해야 합니다. Docker를 사용하면 아주 간단하게 설치할 수 있습니다. 프로젝트 폴더에 `docker-compose.yml` 파일을 만들고 아래 내용을 작성해주세요.

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

### 4. Spring Boot 프로젝트 설정

이제 Spring Boot 프로젝트가 RabbitMQ와 대화할 수 있도록 준비시킵니다.

**1) `build.gradle` 파일 수정**

Spring에서 RabbitMQ를 쉽게 사용하도록 도와주는 라이브러리를 추가합니다.
```groovy
implementation 'org.springframework.boot:spring-boot-starter-amqp'
```

**2) `application.yaml` 파일 수정**

우리 프로젝트에게 방금 띄운 RabbitMQ 우체국의 주소와 로그인 정보를 알려줍니다.
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: admin123!@#
```

### 5. 코드로 우체국 만들기 (RabbitMQ 설정)

이제 코드를 통해 Exchange(분류기)와 Queue(우편함)를 만들고, 둘을 연결해주는 작업을 진행합니다.

```java
@Configuration
public class RabbitMqConfig {

    // "delivery.exchange"라는 이름의 Exchange(우체국 분류기)를 만듭니다.
    @Bean
    public DirectExchange directExchange() {
        return new DirectExchange("delivery.exchange");
    }

    // "delivery.queue"라는 이름의 Queue(우편함)를 만듭니다.
    @Bean
    public Queue queue(){
        return new Queue("delivery.queue");
    }

    // 위에서 만든 Exchange와 Queue를 "delivery.key"라는 주소(라우팅 키)로 연결합니다.
    // 이제 "delivery.key"로 온 편지는 "delivery.queue" 우편함에 쌓이게 됩니다.
    @Bean
    public Binding binding(DirectExchange exchange, Queue queue){
        return BindingBuilder.bind(queue).to(exchange).with("delivery.key");
    }

    // 메시지를 RabbitMQ로 보낼 때 사용할 도구(RabbitTemplate)를 설정합니다.
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
*참고: `ObjectMapper` 설정은 기존 초안의 코드를 재사용합니다.*

### 6. 메시지 보내고 받기 (Producer & Consumer)

이제 발신인(Producer)과 수신인(Consumer)을 만들어 메시지를 주고받아 보겠습니다.

**1) 메시지 보내기 (Producer)**

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

**2) 메시지 받기 (Consumer)**

`@RabbitListener` 어노테이션은 특정 큐를 항상 지켜보다가, 새 메시지가 오면 자동으로 가져와서 처리하는 똑똑한 집배원 역할을 합니다.

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

### 7. 눈으로 확인하기: RabbitMQ 관리자 페이지

이제 정말 메시지가 잘 오고 가는지 눈으로 확인해봅시다. 웹 브라우저에서 `http://localhost:15672` 로 접속하고, `docker-compose.yml`에 설정한 아이디(`admin`)와 비밀번호(`admin123!@#`)로 로그인하세요.

- **[Exchanges]**, **[Queues]** 탭을 눌러보세요. 우리가 코드로 만든 `delivery.exchange`와 `delivery.queue`가 실제로 생성된 것을 볼 수 있습니다.
- 메시지를 보내는 API를 호출한 뒤, **[Queues]** 탭에서 `delivery.queue`를 클릭해보세요. `Ready` 숫자가 올라가고, 잠시 후 Consumer가 메시지를 처리하면 `0`으로 바뀌는 것을 관찰할 수 있습니다.

이 관리자 페이지를 통해 메시지의 흐름을 직관적으로 파악할 수 있어 디버깅 시 매우 유용합니다.

### 결론

이번 글에서는 RabbitMQ의 기본 개념부터 Spring Boot와의 연동, 그리고 관리자 페이지를 통한 확인까지의 전 과정을 알아보았습니다. 비동기 메시지 큐를 사용하면 애플리케이션의 성능을 향상시키고, 더 안정적인 시스템을 구축할 수 있습니다. 처음에는 개념이 조금 낯설 수 있지만, 우체국 비유를 생각하며 차근차근 코드를 작성해보시면 금방 익숙해지실 겁니다.
