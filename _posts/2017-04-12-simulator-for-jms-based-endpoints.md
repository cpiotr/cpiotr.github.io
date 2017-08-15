---
ID: 368
post_title: Simulator for JMS based endpoints
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2017-04-12 22:59:19
---
Testing components which communicate over JMS can quickly become a complex task. It is seldom sufficient to simply verify that a given message is sent to an endpoint. Prevalent case is scenario that involves producing messages on multiple sides of interaction. There can be a number of message types required to be sent to fulfill specific application-level communication protocol. 
Obviously, most of the test cases can be verified by dividing the complex suite into more granular parts that cover single, cohesive unit of work. Yet, more often than not I find myself facing the same problem all over again - simulating JMS based endpoint in a stateful communication.   
For HTTP-based APIs there is a robust mocking framework - <a href="http://wiremock.org/">Wiremock</a>. It allows you to mimic very sophisticated use-cases and eventually make your application more resilient. It's definitely something worth spending some time on. Unfortunately, no such utility exists for JMS yet. As I will present further on, it's relatively easy to prototype something similar that covers bare minimum of functionality. To take advantage of existing components and available framework support, Spring will be used as configuration provider. 
The following code snippet presents a proof-of-concept of a mocking endpoint for JMS communication. To get a full grasp of configuration you might want to refer to <a href="http://ciruk.pl/2017/01/testing-jms-bridge-to-ibm-mq-with-spring-boot/">my previous post on integration testing JMS components</a>.
```
@Test
public void shouldPrintTwoReceivedMessagesAndRespondToOne() throws Exception {
    mockDestination
            .whenReceived(message -> message.equals(FIRST_MESSAGE))
            .thenSend(RESPONSE_QUEUE, message -> message + " acknowledged");
    mockDestination
            .whenReceived(message -> message.startsWith("A"))
            .thenConsume(message -> System.out.println("Received: " + message));

    send(FIRST_MESSAGE);
    send(SECOND_MESSAGE);

    System.out.println(jmsTemplate.receiveAndConvert(RESPONSE_QUEUE));
}

@Bean
Gateway<String> mockDestination(JmsTemplate jmsTemplate) {
    return new MockDestination<>(jmsTemplate, Function.identity());
}
```

The test is based on <a href="https://github.com/cpiotr/blog/blob/master/blog-code/src/main/java/pl/ciruk/blog/jms/MockDestination.java">MockDestination</a> class which uses Spring's `JmsTemplate` underneath. First a set of rules is recorded: when the endpoint receives `FIRST_MESSAGE` on the `REQUEST_QUEUE`, it should respond appropriately to the `RESPONSE_QUEUE`. Furthermore, if a received message starts with the letter <em>A</em>, it should be printed to the standard output.
Mock destination implementation consists of set of predicates with potentially multiple ordered actions. When a given predicate is met, actions are triggered. Additionally, destination exposes methods to mimic sending and receiving messages to make it allow manual flow manipulation.    
```
@Slf4j
class MockDestination<T> implements Gateway<T> {
    private final List<WhenReceived<T>> listeners = new ArrayList<>();

    private final JmsTemplate jmsTemplate;

    private final Function<String, T> fromTextToMessageConverter;

    MockDestination(JmsTemplate jmsTemplate, Function<String, T> fromTextToMessageConverter) {
        this.jmsTemplate = jmsTemplate;
        this.fromTextToMessageConverter = fromTextToMessageConverter;
    }

    @Override
    public void receive(T message) {
        listeners.forEach(consumer -> consumer.onMessage(message));
    }

    @Override
    public void send(String destination, String message) {
        jmsTemplate.convertAndSend(destination, message);
    }

    @JmsListener(destination = "RequestQueue", containerFactory = "defaultJmsListenerContainerFactory")
    private void onMessage(TextMessage message) throws JMSException {
        log.info("onMessage");
        log.debug("onMessage - Message: {}", message);

        try {
            String messageText = message.getText();

            receive(fromTextToMessageConverter.apply(messageText));
        } catch (JMSException e) {
            log.error("Cannot consume message: {}", message, e);
        }
    }

    public WhenReceived<T> whenReceived(Predicate<T> messageMatcher) {
        WhenReceived<T> whenReceived = new WhenReceived<>(this, messageMatcher);
        listeners.add(whenReceived);
        return whenReceived;
    }
}

```

This simple component might not be very generic at the moment and certainly requires a great dose of improvements to be reusable, but it a good starting point for future work. 

The complete solution is available on <a href="https://github.com/cpiotr/blog/blob/master/blog-code/src/test/java/pl/ciruk/blog/jms/MockDestinationIntegrationTest.java">Github</a>.
