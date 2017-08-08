---
ID: 291
post_title: >
  Fan-out CompletableFuture and collect
  results
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2016/04/fan-out-completablefuture-and-collect-results/
published: true
post_date: 2016-04-19 22:26:30
---
When asked about features introduced in Java 8 people often mention <a href="http://openjdk.java.net/jeps/126" target="_blank">lambda expressions</a> and streams. Changes within <code>java.util.concurrent</code> package don't seem to be that popular, but they have definitely been relatively important. 
One of the most prominent members of <a href="http://openjdk.java.net/jeps/155" target="_blank">JEP 155</a> were <a href="https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html" target="_blank">CompletionStage</a> interface and its concrete implementation class <a href="https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html" target="_blank">CompletableFuture</a> which allows creation of asynchronous flows. It's tempting to note, that <a href="https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html" target="_blank">CompletionStage</a> seems to be a subtle wink towards <a href="https://en.wikipedia.org/wiki/Reactive_programming" target="_blank">reactive programming paradigm</a>. Indeed, working with <code>CompletableFutures</code> resembles younger sibling of <a href="https://github.com/ReactiveX/RxJava" target="_blank">RxJava</a> - to a quite limited degree.
<code>CompletableFuture</code> has already been described quite thoroughly and I encourage everyone to take a look at some excellent blog posts on <a href="https://dzone.com/articles/java-8-definitive-guide" target="_blank">DZone</a> or <a href="http://www.infoq.com/articles/Functional-Style-Callbacks-Using-CompletableFuture" target="_blank">QCon</a>. My intention here is not to repeat after others, but to point the use case that seems to be missing and provide a sample implementation as a solution.
There is no concise way to <a href="https://en.wikipedia.org/wiki/Fan-out_(software)" target="_blank">fan-out</a> the flow and collect the results. Consider the following use case. User data is processed encapsulated in a <code>CompletableFuture</code>. At some point we'd like to send requests to a number of remote resources and then collect all responses in a <a href="https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html" target="_blank">Stream</a>. Checking a score of a given film on different popular web pages is what first comes to my mind. In terms of basic building blocks, I'd like to map film title to a score using different functions and then collect results.
Let me define a <code>fanOut</code> method that can be leveraged in described situation.

[sourcecode lang="java"]
static &lt;F, T&gt; Function&lt;F, CompletableFuture&lt;Stream&lt;T&gt;&gt;&gt; fanOut(Collection&lt;Function&lt;F, T&gt;&gt; mappings) {
	return fanOut(mappings, ForkJoinPool.commonPool());
}

static &lt;F, T&gt; Function&lt;F, CompletableFuture&lt;Stream&lt;T&gt;&gt;&gt; fanOut(Collection&lt;Function&lt;F, T&gt;&gt; mappings, ExecutorService executorService) {
	Function&lt;F, CompletableFuture&lt;Stream&lt;T&gt;&gt;&gt; reduce = input -&gt; mappings.stream()
			.map(mapper -&gt; supplyAsync(
					() -&gt; mapper.apply(input))) // from F to CompletableFuture&lt;T&gt;
			.map(cf -&gt; cf.thenApply(Stream::of))  // from CompletableFuture&lt;T&gt; to CompletableFuture&lt;Stream&lt;T&gt;&gt;
			.reduce(completedFuture(Stream.empty()), combineUsing(Stream::concat, executorService));
	return reduce;
}
[/sourcecode]
First thing to note is that the method comes in two flavors: it's overloaded to either accept <code>ExecutorService</code> as second parameter or not. In the latter case a common pool from <code>ForkJoin</code> framework is used. This approach is in line with implementation of <code>CompletableFuture</code> methods.
The algorithm behind fan-out is straightforward. Every mapping is transformed into a <code>CompletableFuture</code>. Then each Future is instructed to map the result into stream. As a last step <code>CompletableFutures</code> are combined using stream concatenation as combining function.
To combine <code>CompletableFutures</code> means to somehow join the results of independent <code>CompletableFutures</code>. Checking score of given film on one web page does not depend on the score of another one.

[sourcecode lang="java"]
public static &lt;T&gt; BinaryOperator&lt;CompletableFuture&lt;T&gt;&gt; combineUsing(BiFunction&lt;T, T, T&gt; combiningFunction, ExecutorService executorService) {
	return (cf1, cf2) -&gt; cf1.thenCombineAsync(cf2, combiningFunction, executorService);
}
[/sourcecode]
Combining function accepts two arguments of type T and returns also T.

Complete source code and tests are available on <a href="https://github.com/cpiotr/blog/blob/master/blog-code/src/main/java/pl/ciruk/blog/fanout/CompletableFutures.java" target="_blank">GitHub</a>.

From the JDK 9 perspective, another set of concurrency updates will extend <code>CompletableFuture</code> capabilities. There's also <a href="http://download.java.net/java/jdk9/docs/api/java/util/concurrent/Flow.html" target="_blank">Flow API</a> arriving under the <a href="http://openjdk.java.net/jeps/266" target="_blank">JEP 266</a>, which is a direct implementation of <a href="http://www.reactive-streams.org/" target="_blank">Reactive Streams</a>.