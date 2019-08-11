---
title: 学习Spring Boot：（二十六）使用 RabbitMQ 消息队列
date: 2018-03-19 22:08:08
tags: [Spring Boot,RabbitMQ]
categories: 学习笔记
---

### 前言
前面学习了 RabbitMQ 基础，现在主要记录下学习 Spring Boot 整合 RabbitMQ ，调用它的 API ，以及中间使用的相关功能的记录。
<!--more-->
### 正文
我这里测试都是使用的是 `topic` 交换器，Spring Boot 2.0.0， jdk 1.8
#### 配置
Spring Boot 版本 2.0.0
在 `pom.xml` 文件中引入 AMQP 的依赖
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```
在系统配置文件中加入连接属性
```yaml
spring:
  application:
    name: RabbitMQ-Demo
  rabbitmq:
    host: k.wuwii.com
    port: 5672
    username: kronchan
    password: 123456
    #virtual-host: test
    publisher-confirms: true # 开启确认消息是否到达交换器，需要设置 true
    publisher-returns: true # 开启确认消息是否到达队列，需要设置 true
```
#### 基本的使用
##### 消费者
新增一个消费者类：
```java
@Log
public class MessageReceiver implements ChannelAwareMessageListener {

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        try {
            byte[] body = message.getBody();
            log.info(">>>>>>> receive： " + new String(body));
        } finally {
            // 确认成功消费，否则消息会转发给其他的消费者，或者进行重试
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        }
    }
}
```

##### 配置类
新增 RabbitMQ 的配置类，主要是对消费者的队列，交换器，路由键的一些设置：
```java
@Configuration
public class RabbitMQConfig {
    public final static String QUEUE_NAME = "springboot.demo.test1";
    public final static String ROUTING_KEY = "route-key";
    public final static String EXCHANGES_NAME = "demo-exchanges";

    @Bean
    public Queue queue() {
        // 是否持久化
        boolean durable = true;
        // 仅创建者可以使用的私有队列，断开后自动删除
        boolean exclusive = false;
        // 当所有消费客户端连接断开后，是否自动删除队列
        boolean autoDelete = false;
        return new Queue(QUEUE_NAME, durable, exclusive, autoDelete);
    }

    /**
     * 设置交换器，这里我使用的是 topic exchange
     */
    @Bean
    public TopicExchange exchange() {
        // 是否持久化
        boolean durable = true;
        // 当所有消费客户端连接断开后，是否自动删除队列
        boolean autoDelete = false;
        return new TopicExchange(EXCHANGES_NAME, durable, autoDelete);
    }

    /**
     * 绑定路由
     */
    @Bean
    public Binding binding(Queue queue, TopicExchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with(ROUTING_KEY);
    }


    @Bean
    public SimpleMessageListenerContainer container(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.setQueueNames(QUEUE_NAME);
        container.setMessageListener(receiver());
        //container.setMaxConcurrentConsumers(1);
        //container.setConcurrentConsumers(1); 默认为1
        //container.setExposeListenerChannel(true);
         container.setAcknowledgeMode(AcknowledgeMode.MANUAL); // 设置为手动，默认为 AUTO，如果设置了手动应答 basicack，就要设置manual
        return container;
    }

    @Bean
    public MessageReceiver receiver() {
        return new MessageReceiver();
    }

}
```

##### 生产者
```java
@Component
public class MessageSender {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    /**
     * logger
     */
    private static final Logger log = LoggerFactory.getLogger(MessageSender.class);

    public void send() {
        // public void convertAndSend(String exchange, String routingKey, final Object object, CorrelationData correlationData)
        // exchange:    交换机名称
        // routingKey:  路由关键字
        // object:      发送的消息内容
        // correlationData:消息ID
        CorrelationData correlationId = new CorrelationData(UUID.randomUUID().toString());
        // ConfirmListener是当消息无法发送到Exchange被触发，此时Ack为False，这时cause包含发送失败的原因，例如exchange不存在时
        // 需要在系统配置文件中设置 publisher-confirms: true
        if (!rabbitTemplate.isConfirmListener()) {
            rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
                if (ack) {
                    log.info(">>>>>>> 消息id:{} 发送成功", correlationData.getId());
                } else {
                    log.info(">>>>>>> 消息id:{} 发送失败", correlationData.getId());
                }
            });
        }
        // ReturnCallback 是在交换器无法将路由键路由到任何一个队列中，会触发这个方法。
        // 需要在系统配置文件中设置 publisher-returns: true
        rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> {
            log.info("消息id：{} 发送失败", message.getMessageProperties().getCorrelationId());
        });
        rabbitTemplate.convertAndSend(RabbitMQConfig.EXCHANGES_NAME, RabbitMQConfig.ROUTING_KEY, ">>>>> Hello World", correlationId);
        log.info("Already sent message.");
    }

}
```
##### 测试发送消息
先启动系统启动类，消费者开始订阅，启动测试类发送消息。
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootRabbitmqApplicationTests {

	@Autowired
	private MessageSender sender;

	@Test
	public void testReceiver() {
		sender.send();
	}
}
```

可以在消费者接收到信息，并且发送端将打出日志 成功发送消息的记录，也可以测试下 `Publisher Confirms and Returns机制` 主要是测试 `ConfirmCallback` 和 `ReturnCallback` 这两个方法。
* `ConfirmCallback` ，确认消息是否到达交换器，例如我们发送一个消息到一个你没有创建过的 交换器上面去，看看情况，
* `ReturnCallback`，确认消息是否到达队列，我们可以这样测试，定义一个路由键，不会被任何队列订阅到，最后查看结果就可以了。

[学习源码](https://github.com/kaimz/rabbitmq-prac/tree/master/springboot-rabbitmq)

#### 使用注解的方式
##### 引入依赖和连接参数 
跟文章第一步的配置一样的。

##### 消费者
```java
@Component
@Log
public class MessageReceiver {

    /**
     * 无返回消息的
     *
     * @param message
     */
    @RabbitListener(bindings = @QueueBinding(value = @Queue(value = Constant.QUEUE_NAME, durable = "true", exclusive = "false", autoDelete = "false"),
            exchange = @Exchange(value = Constant.EXCHANGES_NAME, ignoreDeclarationExceptions = "true", type = ExchangeTypes.TOPIC, autoDelete = "false"),
            key = Constant.ROUTING_KEY))
    public void receive(byte[] message) {
        log.info(">>>>>>>>>>> receive：" + new String(message));
    }

    /**
     * 设置有返回消息的
     * 需要注意的是，
     * 1. 在消息的在生产者（发送消息端）一定要使用 SendAndReceive(……) 这种带有 receive 的方法，否则会抛异常，不捕获会死循环。
     * 2. 该方法调用时会锁定当前线程，并且有可能会造成MQ的性能下降或者服务端/客户端出现死循环现象，请谨慎使用。
     *
     * @param message
     * @return
     */
    @RabbitListener(bindings = @QueueBinding(value = @Queue(value = Constant.QUEUE_NAME, durable = "true", exclusive = "false", autoDelete = "false"),
            exchange = @Exchange(value = Constant.EXCHANGES_NAME, ignoreDeclarationExceptions = "true", type = ExchangeTypes.TOPIC, autoDelete = "false"),
            key = Constant.ROUTING_REPLY_KEY))
    public String receiveAndReply(byte[] message) {
        log.info(">>>>>>>>>>> receive：" + new String(message));
        return ">>>>>>>> I got the message";
    }

}
```
主要是使用到 `@RabbitListener`，虽然看起来参数很多，仔细的你会发现这个和写配置类里面的基本属性是一摸一样的，没有任何区别。

需要注意的是我在这里多做了个有返回值的消息，这个使用异常的话，会不断重试消息，从而阻塞了线程。而且使用它的时候只能使用带有 `receive` 的方法给它发送消息。

##### 生产者
生产者没什么变化。
```java
@Component
public class MessageSender implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnCallback {

    /**
     * logger
     */
    private static final Logger log = LoggerFactory.getLogger(MessageSender.class);
    private RabbitTemplate rabbitTemplate;

    /**
     * 注入 RabbitTemplate
     */
    @Autowired
    public MessageSender(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
        this.rabbitTemplate.setConfirmCallback(this);
        this.rabbitTemplate.setReturnCallback(this);
    }

    /**
     * 测试无返回消息的
     */
    public void send() {
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
        rabbitTemplate.convertAndSend(Constant.EXCHANGES_NAME, Constant.ROUTING_KEY, ">>>>>> Hello World".getBytes(), correlationData);
        log.info(">>>>>>>>>> Already sent message");
    }

    /**
     * 测试有返回消息的，需要注意一些问题
     */
    public void sendAndReceive() {
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
        Object o = rabbitTemplate.convertSendAndReceive(Constant.EXCHANGES_NAME, Constant.ROUTING_REPLY_KEY, ">>>>>>>> Hello World Second".getBytes(), correlationData);
        log.info(">>>>>>>>>>> {}", Objects.toString(o));
    }

    /**
     * Confirmation callback.
     *
     * @param correlationData correlation data for the callback.
     * @param ack             true for ack, false for nack
     * @param cause           An optional cause, for nack, when available, otherwise null.
     */
    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        if (ack) {
            log.info(">>>>>>> 消息id:{} 发送成功", correlationData.getId());
        } else {
            log.info(">>>>>>> 消息id:{} 发送失败", correlationData.getId());
        }
    }

    /**
     * Returned message callback.
     *
     * @param message    the returned message.
     * @param replyCode  the reply code.
     * @param replyText  the reply text.
     * @param exchange   the exchange.
     * @param routingKey the routing key.
     */
    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.info("消息id：{} 发送失败", message.getMessageProperties().getCorrelationId());
    }
}
```

##### 测试
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootAnnotationApplicationTests {

    @Autowired
    private MessageSender sender;

    @Test
    public void send() {
        sender.send();
    }

    @Test
    public void sendAndReceive() {
        sender.sendAndReceive();
    }
}
```

[学习源码](https://github.com/kaimz/rabbitmq-prac/tree/master/springboot-annotation)
