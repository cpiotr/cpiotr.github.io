---
ID: 188
post_title: Producer-Consumer with java.util.concurrent
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2014-02-17 00:51:26
---
Since JSR 236 introduced Concurrency Utilities for Java EE 7, I reckon it's about time to dedust basic knowledge of Java concurrency components. 
Since the release of Java 5 there isn't much arguments for using cumbersome locking scheme using wait/notify. Robust `java.util.concurrent` package made developer's life easier.
A good place to start would be providing simple solution for <a href="http://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem" target="_blank">Producer-Consumer problem</a>. 
Basic assumptions are outlined below:
Producer sends a Queueable message to a Queue
Consumer reads a message from a Queue in a loop
After fetching a message, Consumer may perform additional processing

```
package pl.ciruk.blog.producerconsumer.concurrent;

/**
 * Marker interface for messages.
 * @author piotr.ciruk
 *
 */
public interface Queueable {
	boolean isLastMessage();
}
```

```
package pl.ciruk.blog.producerconsumer.concurrent;

import java.util.concurrent.BlockingQueue;

public class Producer<T extends Queueable> {
	private BlockingQueue<T> queue;
	
	public Producer(BlockingQueue<T> queue) {
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
```

```
package pl.ciruk.blog.producerconsumer.concurrent;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

public class Consumer<T extends Queueable> implements Runnable {
	private BlockingQueue<T> queue;
	
	private Processable<T> processor;
	
	public Consumer(BlockingQueue<T> queue) {
		this(queue, Processable.EMPTY);
	}
	
	public Consumer(BlockingQueue<T> queue, Processable<T> processor) {
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
```

```
package pl.ciruk.blog.producerconsumer.concurrent;

public interface Processable<T> {
	Processable EMPTY = new Processable<Object>() {
		@Override
		public void process(Object element) {
			// Nothing to do, just visiting
		}
		
	};
	
	void process(T element);
}
```

```
// Create a Queue which acts as a communication channel
BlockingQueue<Page> queue = new LinkedBlockingQueue<Page>();

// Create producer and wire it with a queue
Producer<Page> producer = new Producer<>(queue);

// Create a consumer and wire it with the same queue as producer
Consumer<Page> consumer = new Consumer<>(queue, new Processable<Page>() {
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
producer.send(new Page("http://google.com"));
```

Complete implementation example covering fetching content of a number of websites is available on <a href="https://github.com/cpiotr/blog" target="_blank">my Github</a>.