---
ID: 264
post_title: >
  Integration testing with HornetQ and
  Spring Boot
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2015/06/integration-test-hornetq-spring-boot/
published: true
post_date: 2015-06-19 09:58:27
---
<h2>Update</h2>
The current approach to integration tests with Spring Boot and an in-memory queue is described in <a href="http://ciruk.pl/2017/01/testing-jms-bridge-to-ibm-mq-with-spring-boot/">Testing JMS bridge to IBM MQ with Spring Boot</a>.
<h2>Scenario</h2>
Consider the following scenario. You've implemented message driven component that triggers business logic flow. Each part of the flow is unit tested. However, end to end processing path is not yet covered.
Composition of modules for common interaction testing is called <a href="https://en.wikipedia.org/wiki/Integration_testing" target="_blank">integration testing</a>. Ideally, integration test set-up reflects production parties: in case of message driven approach, there should be an actual message queue from which system under test reads.
There is additional complexity related to such configuration: there must be a queue for every run of given integration test. Either a shared one or dedicated. Things become more intricate, when commercial queueing mechanism is involved.
Previously I've described <a href="http://blog.ciruk.pl/2015/04/connecting-to-ibm-mq-with-spring-boot-and-jms/">how to wire up Spring Boot with IBM MQ</a>. Below, I present a way to mock IBM MQ using embedded HornetQ.
<h2>Dependencies</h2>
HornetQ server will only be needed during tests.

[sourcecode lang="xml"]
&lt;dependency&gt;
	&lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
	&lt;artifactId&gt;spring-boot-starter-hornetq&lt;/artifactId&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
	&lt;groupId&gt;org.hornetq&lt;/groupId&gt;
	&lt;artifactId&gt;hornetq-jms-server&lt;/artifactId&gt;
	&lt;scope&gt;test&lt;/scope&gt;
&lt;/dependency&gt;
[/sourcecode]

<h2>Configuration</h2>
You may recall basic configuration for setting up connection to IBM MQ from Spring components:

[sourcecode lang="java"]
@Configuration
@EnableConfigurationProperties(MQConfiguration.MQProperties.class)
@EnableJms
public class MQConfiguration {
        @Inject
        MQConfiguration.MQProperties properties;
 
        @Bean(name = &quot;DefaultJmsListenerContainerFactory&quot;)
        public DefaultJmsListenerContainerFactory provideJmsListenerContainerFactory(PlatformTransactionManager transactionManager) {
            DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
            factory.setConnectionFactory(connectionFactory());
            // omitted
            return factory;
        }
 
        @Bean(name = &quot;JmsTemplate&quot;)
        public JmsTemplate provideJmsTemplate() {
            JmsTemplate jmsTemplate = new JmsTemplate(connectionFactory());
            return jmsTemplate;
        }
 
        private ConnectionFactory connectionFactory() {
            MQXAConnectionFactory factory = new MQXAConnectionFactory();
            factory.setHostName(properties.getHost());
            // ommited
 
            return factory;
        }
 
        // ommited
    }
}
[/sourcecode]

Configuration shown above must be overridden in order to establish connection to an embedded HornetQ. <a href="http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-messaging.html" target="_blank">Spring Boot integrates seamlessly with HornetQ</a>, which makes this step fairly easy. It is sufficient to simply rely on default mechanism to shadow the custom one.

[sourcecode lang="java"]
@Configuration
public class SDCQueueConfiguration {

	@Inject
	DefaultJmsListenerContainerFactory jmsListenerContainerFactory;

	@Bean(name = &quot;DefaultJmsListenerContainerFactory&quot;)
	public DefaultJmsListenerContainerFactory provideJmsListenerContainerFactory(PlatformTransactionManager transactionManager) {
		return jmsListenerContainerFactory;
	}

	@Bean(name = &quot;JmsTemplate&quot;)
	public JmsTemplate provideJmsTemplate(ConnectionFactory connectionFactory) {
		return new JmsTemplate(connectionFactory);
	}
}
[/sourcecode]

<h2>Integration test</h2>
The test is hardly complicated. First a random message is put to a queue. Then it is expected that MyMessageListener picks it up, process it and stores it in a database. Its presence in the DB is then verified with custom assertion.

[sourcecode lang="java"]
@RunWith(SpringJUnit4ClassRunner.class)
@EnableJms
@EnableAutoConfiguration
@SpringApplicationConfiguration(classes = {MockQueueConfiguration.class, MockDBConfiguration.class})
public class MyMessageListenerIT {

	@Inject
	JmsTemplate sender;

	@Value(&quot;${pl.ciruk.mq.incoming-queue}&quot;)
	String requestQueue;

	
	@Inject
	EntityManager entityManager;
	
	@Inject
	MyMessageListener listenerUnderTest;

	CountDownLatch processingFinished;
	
	@Before
	public void setUp() throws Exception {
		processingFinished = new CountDownLatch(1);
		listenerUnderTest.onProcessingComplete(() -&gt; processingFinished.countDown());
	}

	@Test
	public void shouldOnMessage() throws Exception {
		String sampleMessage = randomMessage();
		
		sender.convertAndSend(
				requestQueue,
				sampleMessage
		);
		processingFinished.await(5, TimeUnit.SECONDS);

		assertThat(entityManager, containsQueueMessage(sampleMessage));
	}
}
[/sourcecode]