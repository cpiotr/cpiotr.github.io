---
ID: 246
post_title: >
  Connecting to IBM MQ with Spring Boot
  and JMS
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2015/04/connecting-to-ibm-mq-with-spring-boot-and-jms/
published: true
post_date: 2015-04-08 21:23:17
---
Spring Boot integrates nearly seamlessly with IBM MQ, thanks to its JMS support. Plumbing must be done manually, as usual.
You can use <a href="https://start.spring.io/" target="_blank">Spring Initializr</a> to generate project's stub for simple application with JMS.
It's mandatory to place all IBM MQ jars on classpath. The best solution is to add them to local Maven repository and refer to them from build descriptor (e.g. POM file or <code>build.gradle</code>).

JMS factory beans need to be set up to establish connection. <code>javax.jms.ConnectionFactory</code> is generic factory used by both message producers and consumers. <code>org.springframework.jms.config.DefaultJmsListenerContainerFactory</code> is responsible for message consumer's settings, while <code>org.springframework.jms.core.JmsTemplate</code> can be used for dead simple message producers.
Session Acknowledge Mode is set to <code>Session.CLIENT_ACKNOWLEDGE</code>, meaning message delivery is acknowledged as a part of transaction. If transaction is rolled back, JMS messsage will be redelivered.
It is worth noting, that <code>Connection Factory</code> has <code>Transport Type</code> set to <code>WMQConstants.WMQ_CM_CLIENT</code>.

Connection properties are enclosed in a convenience class called <code>MQProperties</code>.
[sourcecode lang="java"]
@ConfigurationProperties(prefix = &quot;pl.ciruk.blog.mq&quot;)
@Data
public class MQProperties {
    String queueManager;
    String host;
    int port;
    String channel;
    String incomingQueue;
    String outgoingQueue;
}
[/sourcecode]

They are used to populate connection factory. It it the only bean which is tightly coupled with IBM MQ implementation. Below a <code>MQXAConnectionFactory</code> is used, which is suitable for distributed transactions (XA). If you don't rely on distributed transactions, feel free to use <code>MQConnectionFactory</code> instead.
[sourcecode lang="java"]
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
[/sourcecode]

Configuration of beans specific for current application is enclosed within separate class to make it easy to replace connection factory for tests. Nearly all beans from the class below depend on previously defined connection factory. 
[sourcecode lang="java"]
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
        factory.setConcurrency(&quot;5-10&quot;);
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
    public Consumer&lt;String&gt; messageConsumer() {
        return System.out::println;
    }
}
[/sourcecode]

Having defined all beans, it's now high time to use them in concrete service, which serves as both message consumer and producer.
[sourcecode lang="java"]
@Named
@Transactional
@Slf4j
public class MQGateway {
    private JmsTemplate jmsTemplate;

    private MQProperties properties;

    private Consumer&lt;String&gt; messageConsumer;

    @Inject
    public MQGateway(JmsTemplate jmsTemplate, MQProperties properties, @Named(&quot;messageConsumer&quot;) Consumer&lt;String&gt; messageConsumer) {
        this.jmsTemplate = jmsTemplate;
        this.properties = properties;
        this.messageConsumer = messageConsumer;
    }

    @JmsListener(destination = &quot;${pl.ciruk.blog.mq.incoming-queue}&quot;, containerFactory = &quot;defaultJmsListenerContainerFactory&quot;)
    public void onMessage(TextMessage message) throws JMSException {
        log.info(&quot;onMessage&quot;);
        log.debug(&quot;onMessage - Message: {}&quot;, message);

        try {
            messageConsumer.accept(message.getText());
        } catch (JMSException e) {
            log.error(&quot;Cannot consume message: {}&quot;, message, e);
        }

    }

    public void send(String message) {
        log.info(&quot;send&quot;);
        log.debug(&quot;send - Message: {}&quot;, message);

        jmsTemplate.convertAndSend(properties.getOutgoingQueue(), message);
    }
}
[/sourcecode]

Complete source code can be found on <a href="https://github.com/cpiotr/blog/tree/master/blog-code/src/main/java/pl/ciruk/blog/mq" target="_blank">Github</a>.