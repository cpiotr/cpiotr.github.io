---
ID: 331
post_title: Testing JMS bridge to IBM MQ with Spring Boot
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2017-01-03 21:38:38
---
<em>Not invented here</em> syndrome gained recognition in software development, because it's relatively common phenomenon. It has its roots in psychology, where it was identified as knowledge communication problems of decision makers (<em>Martin, E. (2007). Knowledge Communication Problems between Experts and Decision Makers: an Overview and Classification. Electronic Journal of Knowledge Management, 5(3), pp.291-300.</em>). Specific technical choices of other people comprise an easy target for complaints, especially when artifacts that were picked are not state-of-the-art anymore. 
The fact that architectural decisions were made elsewhere is a lousy excuse for writing low-quality code. Bare in mind that low-quality comes in different shapes and sizes. One of them is related to tests. Writing poor tests or no tests at all is a repugnant practice. But it's tempting to blame proprietary middleware messaging system for missing integration tests - setting up additional broker is costly, mundane, cumbersome, etc. A simple solution can be to rely on standardized API for communication and then replace target system with the one you have control over.
Nelson Elhage recently described <a href="https://blog.nelhage.com/2016/12/how-i-test/">how he writes tests</a>. In his post he encourages to write lots of fakes, meaning to mock complex things that are not crucial to current testing subject. As an example he provides an idea of in-memory mock for Amazon S3 storage. The same concept can be applied to messaging middleware.
<a href="https://kafka.apache.org/">Apache Kafka</a> has an <a href="https://github.com/apache/kafka/blob/41e676d29587042994a72baa5000a8861a075c8c/streams/src/test/java/org/apache/kafka/streams/integration/utils/KafkaEmbedded.java">embedded version of broker</a> included in test module out of the box. You can set up your tests to communicate with full-blown Kafka broker which happens to reside in memory. No such facility exists for IBM Websphere MQ, however equipped with JMS interfaces one can take advantage of a system that is a bit more flexible.
In one of my previous posts I shortly described, how to get started with <a href="http://ciruk.pl/2015/04/connecting-to-ibm-mq-with-spring-boot-and-jms/">IBM MQ with Spring Boot and JMS</a>. Having all classes designed for testability greatly reduces the effort needed to wire up a working example. Some hints on testing IBM Websphere MQ with Spring Boot were put together <a href="http://ciruk.pl/2015/06/integration-test-hornetq-spring-boot/">previously</a>, however using HornetQ with Spring Boot became deprecated. The current post can be treated as an update.
Configuration is divided into two classes. One is tightly coupled with IBM Websphere MQ and defines a connection factory. The other configuration class defines a number of messaging components that depend on aforementioned factory. 
There's no need for running IBM MQ instance in tests. An in-memory queuing solution can be used, for example <a href="https://activemq.apache.org/artemis/">Apache ActiveMQ Artemis</a> in embedded mode. Spring Boot provides a starting point for Artemis as starter dependency, which allows automatic detection of dependencies in classpath, important components instantiation and handful of autowiring capabilities. Excellent for testing purposes with low plumbing overhead.
To get started with Artemis the following dependencies have to declared.
```
dependencies {
	// ...
	testCompile 'org.springframework.boot:spring-boot-starter-artemis'
	testCompile 'org.apache.activemq:artemis-jms-server:1.3.0'
	testCompile 'org.springframework.boot:spring-boot-starter-test'
	testCompile 'org.awaitility:awaitility:2.0.0'
}
```

As a next step the code must somehow indicate that Artemis is supposed to run in embedded mode. To achieve the goal, an entry in `application.properties` must be added.
```
spring.artemis.mode=embedded
# ...
```

When it comes to configuration classes, only non-IBM-related one can be picked. It will rely on Spring Boot to provide default connection factory to an in-memory queue. We can even extend the configuration class and replace specific beans with the implementation that fits tests better. 
```
@Configuration
static class TestConfiguration extends MQConfiguration {
    List<String> receivedMessages = new CopyOnWriteArrayList<>();

    @Override
    @Bean
    public Consumer<String> messageConsumer() {
        return receivedMessages::add;
    }

    public boolean hasReceivedMessages() {
        return !receivedMessages.isEmpty();
    }

}
```

`SpringBootTest` annotation allows limiting set of classes which contains beans definition, so that a single test class populates Spring context only with significant components. It makes tests faster and less error-prone.
```
@RunWith(SpringRunner.class)
@SpringBootTest(classes = { MQGatewayIntegrationTest.TestConfiguration.class, MQGateway.class, MQProperties.class })
@EnableAutoConfiguration
@EnableJms
public class MQGatewayIntegrationTest {
    @Inject
    JmsTemplate jmsTemplate;

    @Inject
    TestConfiguration configuration;

    @Value("${pl.ciruk.blog.mq.incoming-queue}")
    String queue;

    @Test
	public void shouldReceiveMessageInListener() throws Exception {
        String message = "This is a test message";

        jmsTemplate.convertAndSend(queue, message);
        await().atMost(5, TimeUnit.SECONDS)
                .until(configuration::hasReceivedMessages);

        assertThat(configuration.receivedMessages, contains(message));
    }

    // ...
}
```

Complete code can be found on <a href="https://github.com/cpiotr/blog/blob/master/blog-code/src/test/java/pl/ciruk/blog/mq/MQGatewayIntegrationTest.java">Github</a>.