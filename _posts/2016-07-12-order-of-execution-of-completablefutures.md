---
ID: 305
post_title: Order of execution of CompletableFutures
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2016/07/order-of-execution-of-completablefutures/
published: true
post_date: 2016-07-12 00:48:18
---
<a href="http://ciruk.pl/2016/04/fan-out-completablefuture-and-collect-results/" target="_blank">The previous post</a> briefly mentioned <code>CompletableFuture</code> class and suggested a convenience method for <a href="http://ciruk.pl/2016/04/fan-out-completablefuture-and-collect-results/" target="_blank">workflow fan-out</a>. I would like to take a slightly deeper dive on the subject of <code>CompletableFutures</code> and focus on the order of execution. Tasks represented by <code>CompletableFuture</code> operation are executed by the underlying thread pools. While this fact sounds obvious, it is really important to keep that in mind.

<h2>Methodology</h2>
I have prepared three sample tests to show how a chain of tasks is executed. Each chain (flow) is represented by three long-running tasks (operations with intrinsic latency). Within a chain, tasks are connected by composition, meaning that the second task is started after the first one finishes.

[sourcecode lang="java"]
public class CompletableFutureFlow {
    static CompletableFuture&lt;Integer&gt; flowWithId(int id, ExecutorService pool) {
        return firstOperation(id, pool)
                .thenCompose(__ -&gt; secondOperation(id, pool))
                .thenCompose(__ -&gt; thirdOperation(id, pool));
    }

    private static CompletableFuture&lt;Integer&gt; firstOperation(int id, ExecutorService pool) {
        return slowOperationAsync(id, 1, pool);
    }

    private static CompletableFuture&lt;Integer&gt; slowOperationAsync(int flow, int step, ExecutorService pool) {
        return CompletableFuture.supplyAsync(
                () -&gt; slowOperation(flow, step),
                pool
        );
    }

    private static int slowOperation(int flow, int step) {
        System.out.println(String.format(&quot;[%d.%d] - %s&quot;, flow, step, Thread.currentThread().getName()));
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().isInterrupted();
            e.printStackTrace();
        }

        return flow;
    }
}
[/sourcecode]

Each test creates a number of such flows up front and stores references to them in a list. Afterwards the test method blocks, waiting for flows to stop running.

[sourcecode lang="java"]
@Test
public void testFlowWithCommonForkJoinPool() throws Exception {
    pool = ForkJoinPool.commonPool();

    List&lt;CompletableFuture&lt;Integer&gt;&gt; futures = new ArrayList&lt;&gt;();
    for (int i = 0; i &lt; 20; i++) {
        futures.add(
                CompletableFutureFlow.flowWithId(i, pool));
    }
    futures.forEach(CompletableFutureFlow::getFuture);
}
[/sourcecode]

<h2>Observations</h2>
CompletableFuture executed in a default pool (when task is supplied using method signature without explicit thread pool, i.e. <code>CompletableFuture.supplyAsync(() -&gt; 1)</code>) were intermingled as expected.

[sourcecode]
[Flow:0][Step:1] - ForkJoinPool.commonPool-worker-1
[Flow:3][Step:1] - ForkJoinPool.commonPool-worker-4
[Flow:1][Step:1] - ForkJoinPool.commonPool-worker-2
[Flow:2][Step:1] - ForkJoinPool.commonPool-worker-3
[Flow:4][Step:1] - ForkJoinPool.commonPool-worker-7
[Flow:6][Step:1] - ForkJoinPool.commonPool-worker-6
[Flow:5][Step:1] - ForkJoinPool.commonPool-worker-5
[Flow:0][Step:2] - ForkJoinPool.commonPool-worker-1
[Flow:7][Step:1] - ForkJoinPool.commonPool-worker-4
[Flow:8][Step:1] - ForkJoinPool.commonPool-worker-2
[Flow:3][Step:2] - ForkJoinPool.commonPool-worker-3
[Flow:2][Step:2] - ForkJoinPool.commonPool-worker-7
[Flow:4][Step:2] - ForkJoinPool.commonPool-worker-6
[Flow:5][Step:2] - ForkJoinPool.commonPool-worker-5
[Flow:6][Step:2] - ForkJoinPool.commonPool-worker-4
[Flow:0][Step:3] - ForkJoinPool.commonPool-worker-2
[/sourcecode]
If default pool works fine, why would anyone like to use custom one? It depends on the use case. If your tasks are IO bound, then you might want to extend your core pool size.

Let's try to do that, except I'll restrict the number of active threads in the pool. Here's the output for <code>pool = Executors.newFixedThreadPool(4)</code>:
[sourcecode]
[Flow:0][Step:1] - pool-1-thread-1
[Flow:1][Step:1] - pool-1-thread-2
[Flow:2][Step:1] - pool-1-thread-3
[Flow:3][Step:1] - pool-1-thread-4
[Flow:4][Step:1] - pool-1-thread-1
[Flow:5][Step:1] - pool-1-thread-2
[Flow:6][Step:1] - pool-1-thread-3
[Flow:7][Step:1] - pool-1-thread-4
[Flow:8][Step:1] - pool-1-thread-1
[Flow:9][Step:1] - pool-1-thread-2
[Flow:10][Step:1] - pool-1-thread-3
[Flow:11][Step:1] - pool-1-thread-4
[Flow:12][Step:1] - pool-1-thread-1
[Flow:13][Step:1] - pool-1-thread-2
[Flow:14][Step:1] - pool-1-thread-3
[Flow:15][Step:1] - pool-1-thread-4
[Flow:16][Step:1] - pool-1-thread-1
[Flow:18][Step:1] - pool-1-thread-3
[Flow:19][Step:1] - pool-1-thread-4
[Flow:17][Step:1] - pool-1-thread-2
[Flow:0][Step:2] - pool-1-thread-1
[Flow:1][Step:2] - pool-1-thread-3
[Flow:3][Step:2] - pool-1-thread-4
[/sourcecode]
It is clear that first step of each flow took precedence before other ones. That is rather significant change, regarding that the only thing that changed was a thread pool. What will happen if custom fork join pool is used, i.e. <code>pool = new ForkJoinPool(4)</code>?
[sourcecode]
Flow:0][Step:1] - ForkJoinPool-1-worker-1
[Flow:1][Step:1] - ForkJoinPool-1-worker-2
[Flow:2][Step:1] - ForkJoinPool-1-worker-3
[Flow:3][Step:1] - ForkJoinPool-1-worker-0
[Flow:0][Step:2] - ForkJoinPool-1-worker-2
[Flow:1][Step:2] - ForkJoinPool-1-worker-3
[Flow:4][Step:1] - ForkJoinPool-1-worker-1
[Flow:2][Step:2] - ForkJoinPool-1-worker-0
[Flow:7][Step:1] - ForkJoinPool-1-worker-1
[Flow:0][Step:3] - ForkJoinPool-1-worker-0
[Flow:6][Step:1] - ForkJoinPool-1-worker-3
[Flow:5][Step:1] - ForkJoinPool-1-worker-2
[Flow:4][Step:2] - ForkJoinPool-1-worker-1
[Flow:8][Step:1] - ForkJoinPool-1-worker-0
[Flow:3][Step:2] - ForkJoinPool-1-worker-3
[/sourcecode]
Tasks are intermingled once again. And why is that?

<h2>Explanation</h2>
Most of the common executors available in <code>java.util.concurrent.Executors</code> are based on blocking FIFO (first-in-first-out) queues. In the scenario described above, all flows are constructed eagerly, which means that execution of steps number one will take precedence before step two and three.
<a href="http://gee.cs.oswego.edu/dl/papers/fj.pdf" target="_blank">ForkJoinPool</a> takes a different approach. It employs <a href="https://en.wikipedia.org/wiki/Work_stealing" target="_blank">work stealing framework</a> to fit in scenario of dynamic tasks creation. Each worker thread in the pool maintain its own double-ended queue. If a given worker runs out of tasks to process, it can <em>steal</em> tasks from other workers. Processing of worker's own tasks comes in LIFO (last-in-first-out) order, while <em>stealing</em> is ordered FIFO. This combination (along with many more implementation subtleties) allows to achieve desired order of execution.

Since Java 1.8 there are factory methods for creating custom <code>ForkJoinPools</code>: <code>Executors.newWorkStealingPool()</code> and <code>Executors.newWorkStealingPool(int parallelism)</code>.