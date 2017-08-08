---
ID: 188
post_title: >
  Producer-Consumer with
  java.util.concurrent
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2014/02/producer-consumer-with-java-util-concurrent/
published: true
post_date: 2014-02-17 00:51:26
---
Since JSR 236 introduced Concurrency Utilities for Java EE 7, I reckon it's about time to dedust basic knowledge of Java concurrency components. 
Since the release of Java 5 there isn't much arguments for using cumbersome locking scheme using wait/notify. Robust <code>java.util.concurrent</code> package made developer's life easier. 
A good place to start would be providing simple solution for <a href="http://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem" target="_blank">Producer-Consumer problem</a>. 
Basic assumptions are outlined below:
Producer sends a Queueable message to a Queue
Consumer reads a message from a Queue in a loop
After fetching a message, Consumer may perform additional processing

[sourcecode lang="java"]
package pl.ciruk.blog.producerconsumer.concurrent;

/**
 * Marker interface for messages.
 * @author piotr.ciruk
 *
 */
public interface Queueable {
	boolean isLastMessage();
}
[/sourcecode]

[sourcecode lang="java"]
package pl.ciruk.blog.producerconsumer.concurrent;

import java.util.concurrent.BlockingQueue;

public class Producer&lt;T extends Queueable&gt; {
	private BlockingQueue&lt;T&gt; queue;
	
	public Producer(BlockingQueue&lt;T&gt; queue) {
		this.queue = queue;
	}
	
	public void send(T message) {
		try {
			queue.put(message);
		} catch (InterruptedException e) {
			throw new RuntimeException(e);
		}
	}
}
[/sourcecode]

[sourcecode lang="java"]
package pl.ciruk.blog.producerconsumer.concurrent;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

public class Consumer&lt;T extends Queueable&gt; implements Runnable {
	private BlockingQueue&lt;T&gt; queue;
	
	private Processable&lt;T&gt; processor;
	
	public Consumer(BlockingQueue&lt;T&gt; queue) {
		this(queue, Processable.EMPTY);
	}
	
	public Consumer(BlockingQueue&lt;T&gt; queue, Processable&lt;T&gt; processor) {
		this.queue = queue;
		this.processor = processor;
	}

	@Override
	public void run() {
		
		try {
			consumeMessages();
		} catch (InterruptedException e) {
			throw new RuntimeException(e);
		}
	}
	
	private void consumeMessages() throws InterruptedException {
		while (true) {
			T message = queue.poll(1, TimeUnit.SECONDS);
			if (Queueables.isLastMessage(message)) {
				return;
			}
			
			processor.process(message);
		}
	}
}
[/sourcecode]

[sourcecode lang="java"]
package pl.ciruk.blog.producerconsumer.concurrent;

public interface Processable&lt;T&gt; {
	Processable EMPTY = new Processable&lt;Object&gt;() {
		@Override
		public void process(Object element) {
			// Nothing to do, just visiting
		}
		
	};
	
	void process(T element);
}
[/sourcecode]

[sourcecode lang="java"]
// Create a Queue which acts as a communication channel
BlockingQueue&lt;Page&gt; queue = new LinkedBlockingQueue&lt;Page&gt;();

// Create producer and wire it with a queue
Producer&lt;Page&gt; producer = new Producer&lt;&gt;(queue);

// Create a consumer and wire it with the same queue as producer
Consumer&lt;Page&gt; consumer = new Consumer&lt;&gt;(queue, new Processable&lt;Page&gt;() {
	@Override
	public void process(Page element) {
		// Additional work to do
		element.getContent();
	}
});

// Run consumers
ExecutorService executor = Executors.newFixedThreadPool(2);
executor.execute(consumer);

// Send a message
producer.send(new Page(&quot;http://google.com&quot;));
[/sourcecode]

Complete implementation example covering fetching content of a number of websites is available on <a href="https://github.com/cpiotr/blog" target="_blank">my Github</a>.