---
ID: 273
post_title: Getting rid of switch
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2016/01/getting-rid-of-switch/
published: true
post_date: 2016-01-03 16:49:06
---
In Java switch statement is a common approach to multiple conditional execution paths. It serves its purpose only for limited domain, however. Switch uses equality as comparison operator to find a possible match. More often than not a complex operator is required. More flexible alternative can be a humongous if-else monster. Readability and testability are seriously harmed in both cases. Such impasse occurs often when dealing with low-level protocols or API based on primitive values.

[sourcecode lang="java"]
void findFor(String pattern) {
	switch (pattern) {
		case &quot;FIND_ALL&quot;:
			findAll();
			break;
		case &quot;FIRST&quot;:
			findFirst();
			break;
		default:
			findSomethingElse();
			break;
	}
}

private void findSomethingElse(String pattern) {
	if (pattern.length() &lt; 3) {
		findShort();
	} else if (pattern.contains(&quot;:ID:&quot;)) {
		findForId();
	}
}
[/sourcecode]

Functional programming languages have a robust feature called <a href="https://dzone.com/articles/scala-pattern-matching-case" target="_blank">pattern matching</a> which can be a remedy for described situation. It is very powerful and definitely worth looking at. There is an interesting Java library available on GitHub: <a href="http://javaslang.com/" target="_blank">javaslang</a>, which introduces a number of clever features, <a href="http://javaslang.github.io/javaslang-docs/2.0.0-RC2/#_match" target="_blank">pattern matching</a> among the others.

Object oriented languages are equipped with polymorphism where a method is overridden with concrete implementation to cover given execution path. Target implementation still has to be picked somehow. 

[sourcecode lang="java"]
void findFor(String pattern) {
	Action action = null;

	switch (pattern) {
		case &quot;FIND_ALL&quot;:
			action = findAll();
			break;
		case &quot;FIRST&quot;:
			action = findFirst();
			break;
		default:
			action = findSomethingElse(pattern);
			break;
	}

	action.execute();
}

private void findSomethingElse(String pattern) {
	if (pattern.length() &lt; 3) {
		return findShort();
	} else if (pattern.contains(&quot;:ID:&quot;)) {
		return findForId();
	}
}
[/sourcecode]

Still not as concise as it can be. Not very good looking either. In terms of refactoring, a logical next step is to leverage a map to store all actions related to given patterns. From a performance perspective such change is should not really matter, because under the hood <a href="https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-3.html#jvms-3.10" target="_blank">javac compiles switch statement</a> to <code>tableswitch</code> or <code>lookupswitch</code> bytecode. A map with target behaviour stored as values successfully mimics basic functionality of switch statement.
[sourcecode lang="java"]
void findFor(String pattern) {
	Map&lt;String, Action&gt; actions = initMap();

	Action action = actions.getOrDefault(
			pattern,
			findSomethingElse(pattern)
	);

	action.execute();
}

private void findSomethingElse(String pattern) {
	if (pattern.length() &lt; 3) {
		return findShort();
	} else if (pattern.contains(&quot;:ID:&quot;)) {
		return findForId();
	}
}
[/sourcecode]

To achieve higher flexibility in terms of comparison operators, conjunction of <code>Predicate</code> and <code>Consumer</code> is required: predicate is responsible for matching against pattern and consumer serves as a behaviour. A collection of concrete implementations might be kept in a list and traversed using streams.
[sourcecode lang="java"]
void findFor(String pattern) {
	Collection&lt;ActionWithPredicate&gt; actions = initActions();

	actions.stream()
			.filter(actionWithPredicate -&gt; actionWithPredicate.test(pattern))
			.forEach(Action::execute);
}

private Collection&lt;ActionWithPredicate&gt; initActions() {
	return Stream
			.of(new ShortAction()) // other implementations omitted
			.collect(Collectors.toList());
}

private class ShortAction extends ActionWithPredicate {
	@Override
	public boolean test(String pattern) {
		return pattern.length() &lt; 3;
	}

	@Override
	public void execute() {
		// do something short
	}
}
	private class Action {
	public void execute() {
		// No-op
	}
}
[/sourcecode]