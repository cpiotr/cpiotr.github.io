---
ID: 199
post_title: Creating instances of utility classes
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2014/04/creating-instances-of-utility-classes/
published: true
post_date: 2014-04-23 20:52:40
---
One of the first books I've read regarding Java was <a href="http://www.amazon.com/Effective-Java-Edition-Joshua-Bloch/dp/0321356683" target="_blank">Effective Java</a> by <a href="http://en.wikipedia.org/wiki/Joshua_Bloch" target="_blank">Joschua Bloch</a>. In this excellent read author suggests to enforce noninstantiability of utility classes with a private constructor.

[sourcecode language="java"]
public class SampleUtilityClass {
	private SampleUtilityClass() {
		// Important feature - fail execution in case of accidental instatiation.
		throw new AssertionError();
	}
}
[/sourcecode]

Unfortunately, some useful <a href="http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/nio/file/Files.java" target="_blank">utility classes</a>, introduced in Java 1.7 seem to lack this additional assertion.
Let's have some fun (sic!) and try to create <code>java.nio.file.Files</code> objects.

[sourcecode language="java"]
public static void main(String[] args) {
	instantiate(java.nio.file.Files.class);
}
[/sourcecode]

[sourcecode language="java"]
public static &lt;E&gt; void instantiate(Class&lt;E&gt; classToBeInstantiated) {
	for (Constructor&lt;?&gt; constructor : classToBeInstantiated.getDeclaredConstructors()) {
		// No more private, sorry.
		constructor.setAccessible(true);
		
		try {
			Object[] parameters = new Object[constructor.getParameterCount()];
			E instance = (E) constructor.newInstance(parameters);
			
			System.out.println(&quot;Created instance: &quot; + instance);
		} catch (InstantiationException | IllegalAccessException | IllegalArgumentException | InvocationTargetException e) {
			e.printStackTrace();
		}
	}
}
[/sourcecode]
[sourcecode]
Created instance: java.nio.file.Files@15db9742 
[/sourcecode]

Great, it worked. Out of curiosity, let's try to create some objects of allegedly safe SampleUtilityClass.
[sourcecode language="java"]
public static void main(String[] args) {
	instantiate(SampleUtilityClass.class);
}
[/sourcecode]

[sourcecode]
java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:408)
	at pl.ciruk.blog.noninst.NoninstantiableUtils.instantiate(NoninstantiableUtils.java:19)
	at pl.ciruk.blog.noninst.NoninstantiableUtils.main(NoninstantiableUtils.java:46)
Caused by: java.lang.AssertionError
	at pl.ciruk.blog.noninst.SampleUtilityClass.&lt;init&gt;(SampleUtilityClass.java:5)
	... 6 more
[/sourcecode]

As assumed, the class cannot be instantiated this way. I'm not out of ideas, though. Safe way didn't work out, the unsafe one comes next in the line.
[sourcecode language="java"]
public static void main(String[] args) {
	instantiate(java.nio.file.Files.class);
	instantiateWithUnsafe(SampleUtilityClass.class);
}
[/sourcecode]

[sourcecode language="java"]
public static &lt;E&gt; void instantiateWithUnsafe(Class&lt;E&gt; classToBeInstantiated) {
	try {
		E instance = (E) getUnsafe().allocateInstance(classToBeInstantiated);
		System.out.println(&quot;Created instance: &quot; + instance);
	} catch (InstantiationException | NoSuchFieldException | SecurityException | IllegalArgumentException | IllegalAccessException e) {
		e.printStackTrace();
	}
}

private static sun.misc.Unsafe getUnsafe() throws NoSuchFieldException, SecurityException, IllegalArgumentException, IllegalAccessException {
	Field theUnsafe = sun.misc.Unsafe.class.getDeclaredField(&quot;theUnsafe&quot;);
	theUnsafe.setAccessible(true);
	sun.misc.Unsafe unsafe = (sun.misc.Unsafe) theUnsafe.get(null);
	return unsafe;
}
[/sourcecode]


[sourcecode]
Created instance: java.nio.file.Files@15db9742
Created instance: pl.ciruk.blog.noninst.SampleUtilityClass@7852e922
[/sourcecode]

Another piece of unnecessary garbage was created, splendid. But how to prevent such instantiation? It turns out, that abstract modifier on class level comes in handy when combined with private constructor which throws an error. However, marking class as abstract only to avoid instantiation is somehow unelegant and suggests that class is meant to be subclassed. Lesser evil must be chosen, I suppose.


[sourcecode language="java"]
package pl.ciruk.blog.noninst;

import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;

public abstract class NoninstantiableUtils {
	private NoninstantiableUtils() {
		throw new AssertionError();
	}
	
	public static &lt;E&gt; void instantiate(Class&lt;E&gt; classToBeInstantiated) {
		for (Constructor&lt;?&gt; constructor : classToBeInstantiated.getDeclaredConstructors()) {
			// No more private, sorry.
			constructor.setAccessible(true);
			
			try {
				Object[] parameters = new Object[constructor.getParameterCount()];
				E instance = (E) constructor.newInstance(parameters);
				
				System.out.println(&quot;Created instance: &quot; + instance);
			} catch (InstantiationException | IllegalAccessException | IllegalArgumentException | InvocationTargetException e) {
				e.printStackTrace();
			}
		}
	}
	
	public static &lt;E&gt; void instantiateWithUnsafe(Class&lt;E&gt; classToBeInstantiated) {
		try {
			E instance = (E) getUnsafe().allocateInstance(classToBeInstantiated);
			System.out.println(&quot;Created instance: &quot; + instance);
		} catch (InstantiationException | NoSuchFieldException | SecurityException | IllegalArgumentException | IllegalAccessException e) {
			e.printStackTrace();
		}
	}
	
	private static sun.misc.Unsafe getUnsafe() throws NoSuchFieldException, SecurityException, IllegalArgumentException, IllegalAccessException {
		Field theUnsafe = sun.misc.Unsafe.class.getDeclaredField(&quot;theUnsafe&quot;);
		theUnsafe.setAccessible(true);
		sun.misc.Unsafe unsafe = (sun.misc.Unsafe) theUnsafe.get(null);
		return unsafe;
	}
	
	public static void main(String[] args) {
		instantiate(java.nio.file.Files.class);
		instantiateWithUnsafe(SampleUtilityClass.class);
		instantiateWithUnsafe(NoninstantiableUtils.class);
	}
}
[/sourcecode]

[sourcecode language="java"]
package pl.ciruk.blog.noninst;

public final class SampleUtilityClass {
	private SampleUtilityClass() {
		throw new AssertionError();
	}
}
[/sourcecode]