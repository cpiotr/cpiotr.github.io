---
ID: 246
post_title: >
  Connecting to IBM MQ with Spring Boot
  and JMS
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2015-04-08 21:23:17
---
Spring Boot integrates nearly seamlessly with IBM MQ, thanks to its JMS support. Plumbing must be done manually, as usual.
You can use <a href="https://start.spring.io/" target="_blank">Spring Initializr</a> to generate project's stub for simple application with JMS.
It's mandatory to place all IBM MQ jars on classpath. The best solution is to add them to local Maven repository and refer to them from build descriptor (e.g. POM file or `build.gradle`).

JMS factory beans need to be set up to establish connection. `javax.jms.ConnectionFactory` is generic factory used by both message producers and consumers. `org.springframework.jms.config.DefaultJmsListenerContainerFactory` is responsible for message consumer's settings, while `org.springframework.jms.core.JmsTemplate` can be used for dead simple message producers.
Session Acknowledge Mode is set to `Session.CLIENT_ACKNOWLEDGE`, meaning message delivery is acknowledged as a part of transaction. If transaction is rolled back, JMS messsage will be redelivered.
It is worth noting, that `Connection Factory` has `Transport Type` set to `WMQConstants.WMQ_CM_CLIENT`.

Connection properties are enclosed in a convenience class called `MQProperties`.
```
@ConfigurationProperties(prefix = "pl.ciruk.blog.mq")
@Data
public class MQProperties {
    String queueManager;
    String host;
    int port;
    String channel;
    String incomingQueue;
    String outgoingQueue;
}
```

They are used to populate connection factory. It it the only bean which is tightly coupled with IBM MQ implementation. Below a `MQXAConnectionFactory` is used, which is suitable for distributed transactions (XA). If you don't rely on distributed transactions, feel free to use `MQConnectionFactory` instead.
```
@Configuration
@EnableConfigurationProperties(MQProperties.class)
public class ConnectionConfiguration {
    @Inject
    MQProperties properties;

    @Bean
    public ConnectionFactory connectionFactory() {
        MQXAConnectionFactory factory = null;
        try {
            factory = new MQXAConnectionFactory();
            factory.setHostName(properties.getHost());
            factory.setPort(properties.getPort());
            factory.setQueueManager(properties.getQueueManager());
            factory.setChannel(properties.getChannel());
            factory.setTransportType(WMQConstants.WMQ_CM_CLIENT);
        } catch (JMSException e) {
            throw new RuntimeException(e);
        }

        return factory;
    }
}
```

Configuration of beans specific for current application is enclosed within separate class to make it easy to replace connection factory for tests. Nearly all beans from the class below depend on previously defined connection factory. 
```
@Configuration
@EnableJms
public class MQConfiguration {
    @Bean
    public PlatformTransactionManager platformTransactionManager(ConnectionFactory connectionFactory) {
        return new JmsTransactionManager(connectionFactory);
    }

    @Bean
    public DefaultJmsListenerContainerFactory defaultJmsListenerContainerFactory(PlatformTransactionManager transactionManager, ConnectionFactory connectionFactory) {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setTransactionManager(transactionManager);
        factory.setConcurrency("5-10");
        factory.setSessionAcknowledgeMode(Session.CLIENT_ACKNOWLEDGE);
        factory.setSessionTransacted(true);
        return factory;
    }

    @Bean
    public JmsTemplate jmsTemplate(ConnectionFactory connectionFactory) {
        JmsTemplate jmsTemplate = new JmsTemplate(connectionFactory);
        return jmsTemplate;
    }

    @Bean
    public Consumer<String> messageConsumer() {
        return System.out::println;
    }
}
```

Having defined all beans, it's now high time to use them in concrete service, which serves as both message consumer and producer.
```
@Named
@Transactional
@Slf4j
public class MQGateway {
    private JmsTemplate jmsTemplate;

    private MQProperties properties;

    private Consumer<String> messageConsumer;

    @Inject
    public MQGateway(JmsTemplate jmsTemplate, MQProperties properties, @Named("messageConsumer") Consumer<String> messageConsumer) {
        this.jmsTemplate = jmsTemplate;
        this.properties = properties;
        this.messageConsumer = messageConsumer;
    }

    @JmsListener(destination = "${pl.ciruk.blog.mq.incoming-queue}", containerFactory = "defaultJmsListenerContainerFactory")
    public void onMessage(TextMessage message) throws JMSException {
        log.info("onMessage");
        log.debug("onMessage - Message: {}", message);

        try {
            messageConsumer.accept(message.getText());
        } catch (JMSException e) {
            log.error("Cannot consume message: {}", message, e);
        }

    }

    public void send(String message) {
        log.info("send");
        log.debug("send - Message: {}", message);

        jmsTemplate.convertAndSend(properties.getOutgoingQueue(), message);
    }
}
```

Complete source code can be found on <a href="https://github.com/cpiotr/blog/tree/master/blog-code/src/main/java/pl/ciruk/blog/mq" target="_blank">Github</a>.