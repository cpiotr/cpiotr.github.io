---
ID: 279
post_title: Intersection types in Java
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2016/02/intersection-types-java/
published: true
post_date: 2016-02-28 00:00:07
---
In <a href="http://blog.ciruk.pl/2016/01/getting-rid-of-switch/" target="_blank">my previous post</a> I presented how Java 8 Streams can be used to increase code readability by eliminating conditional flows. To achieve that goal, a special interface which extends <code>Predicate&lt;T&gt;</code> and <code>Consumer&lt;T&gt;</code> was created.
The solution was clear and type-safe. But let's complicate things a bit. Assume that there are many different interfaces that combine <code>Predicate&lt;T&gt;</code> and <code>Consumer&lt;T&gt;</code>. However, they are not known at compile time. The aim is to provide a container for classes implementing those interfaces and to be able to treat them as both predicates and consumers.

There is no explicit way to create a collection of classes that implement two interfaces. But the collection of Objects can be encapsulated and accessed only through the type-safe API. Fortunately, Java Language Specification defines a concept of <a href="https://docs.oracle.com/javase/specs/jls/se7/html/jls-4.html#jls-4.9" target="_blank">intersection types</a>, which allows developers to mimic interface combination at runtime.
[sourcecode lang="Java"]
public class ConsumerWithPredicates&lt;U&gt; {
        List&lt;Object&gt; consumers = Lists.newArrayList();

        public &lt;T extends Consumer&lt;U&gt; &amp; Predicate&lt;U&gt;&gt; void add(T consumer) {
                consumers.add(consumer);
        }

        public void consume(U element) {
                consumers.stream()
                                .map(c -&gt; (Consumer&lt;U&gt; &amp; Predicate&lt;U&gt;) c)
                                .filter(c -&gt; c.test(element))
                                .forEach(c -&gt; c.accept(element));
        }
}
[/sourcecode]

Based on the implementation above, it's now possible to add different classes implementing both consumer and predicate interfaces. 
[sourcecode lang="Java"]
@Test
public void shouldAcceptDifferentInterfaces() throws Exception {
	// Given
	ConsumerWithPredicates&lt;String&gt; types = new ConsumerWithPredicates();

	// When
	types.add(newTestAndConsume(e -&gt; true, e -&gt; {}));
	types.add(newActionWithPredicate(e -&gt; true, e -&gt; {}));

	// Then
	assertThat(types.consumers, hasSize(2));
}

private &lt;T&gt; TestAndConsume&lt;T&gt; newTestAndConsume(Predicate&lt;T&gt; predicate, Consumer&lt;T&gt; consumer) {
	return new TestAndConsume&lt;T&gt;(){
		@Override
		public boolean test(T element) {
			return predicate.test(element);
		}

		@Override
		public void accept(T element) {
			consumer.accept(element);
		}
	};
}

private &lt;T&gt; ActionWithPredicate&lt;T&gt; newActionWithPredicate(Predicate&lt;T&gt; predicate, Consumer&lt;T&gt; consumer) {
	return new ActionWithPredicate&lt;T&gt;(){
		@Override
		public boolean test(T element) {
			return predicate.test(element);
		}

		@Override
		public void accept(T element) {
			consumer.accept(element);
		}
	};
}
[/sourcecode]

Appropriate <a href="https://github.com/cpiotr/blog/blob/master/blog-code/src/test/java/pl/ciruk/blog/noswitch/ConsumerWithPredicatesTest.java" target="_blank">unit tests</a> are available on Github.

The use of intersection types is <a href="https://www.reddit.com/r/programming/comments/1fr8fv/little_known_java_feature_joint_union_in_type/" target="_blank">rather limited</a>. For further reading: <a href="http://www.javaspecialists.eu/archive/Issue233.html" target="_blank">Heinz Kabuntz on intersection types</a>.