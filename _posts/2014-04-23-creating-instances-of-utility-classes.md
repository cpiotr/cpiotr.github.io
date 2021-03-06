---
ID: 199
post_title: Creating instances of utility classes
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2014-04-23 20:52:40
---
One of the first books related to Java I've read was <a href="http://www.amazon.com/Effective-Java-Edition-Joshua-Bloch/dp/0321356683" target="_blank">Effective Java</a> by <a href="http://en.wikipedia.org/wiki/Joshua_Bloch" target="_blank">Joschua Bloch</a>. In this excellent read author suggests to enforce noninstantiability of utility classes with a private constructor.

```
public class SampleUtilityClass {
	private SampleUtilityClass() {
		// Important feature - fail execution in case of accidental instatiation.
		throw new AssertionError();
	}
}
```

Unfortunately, some useful <a href="http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/nio/file/Files.java" target="_blank">utility classes</a>, introduced in Java 1.7 seem to lack this additional assertion.
Let's have some fun (sic!) and try to create `java.nio.file.Files` objects.

```
public static void main(String[] args) {
	instantiate(java.nio.file.Files.class);
}
```

```
public static <E> void instantiate(Class<E> classToBeInstantiated) {
	for (Constructor<?> constructor : classToBeInstantiated.getDeclaredConstructors()) {
		// No more private, sorry.
		constructor.setAccessible(true);
		
		try {
			Object[] parameters = new Object[constructor.getParameterCount()];
			E instance = (E) constructor.newInstance(parameters);
			
			System.out.println("Created instance: " + instance);
		} catch (InstantiationException | IllegalAccessException | IllegalArgumentException | InvocationTargetException e) {
			e.printStackTrace();
		}
	}
}
```
```
Created instance: java.nio.file.Files@15db9742 
```

Great, it worked. Out of curiosity, let's try to create some objects of allegedly safe SampleUtilityClass.
```
public static void main(String[] args) {
	instantiate(SampleUtilityClass.class);
}
```

```
java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:408)
	at pl.ciruk.blog.noninst.NoninstantiableUtils.instantiate(NoninstantiableUtils.java:19)
	at pl.ciruk.blog.noninst.NoninstantiableUtils.main(NoninstantiableUtils.java:46)
Caused by: java.lang.AssertionError
	at pl.ciruk.blog.noninst.SampleUtilityClass.<init>(SampleUtilityClass.java:5)
	... 6 more
```

As assumed, the class cannot be instantiated this way. I'm not out of ideas, though. Safe way didn't work out, the unsafe one comes next in the line.
```
public static void main(String[] args) {
	instantiate(java.nio.file.Files.class);
	instantiateWithUnsafe(SampleUtilityClass.class);
}
```

```
public static <E> void instantiateWithUnsafe(Class<E> classToBeInstantiated) {
	try {
		E instance = (E) getUnsafe().allocateInstance(classToBeInstantiated);
		System.out.println("Created instance: " + instance);
	} catch (InstantiationException | NoSuchFieldException | SecurityException | IllegalArgumentException | IllegalAccessException e) {
		e.printStackTrace();
	}
}

private static sun.misc.Unsafe getUnsafe() throws NoSuchFieldException, SecurityException, IllegalArgumentException, IllegalAccessException {
	Field theUnsafe = sun.misc.Unsafe.class.getDeclaredField("theUnsafe");
	theUnsafe.setAccessible(true);
	sun.misc.Unsafe unsafe = (sun.misc.Unsafe) theUnsafe.get(null);
	return unsafe;
}
```


```
Created instance: java.nio.file.Files@15db9742
Created instance: pl.ciruk.blog.noninst.SampleUtilityClass@7852e922
```

Another piece of unnecessary garbage was created, splendid. But how to prevent such instantiation? It turns out, that abstract modifier on class level comes in handy when combined with private constructor which throws an error. However, marking class as abstract only to avoid instantiation is somehow unelegant and suggests that class is meant to be subclassed. Lesser evil must be chosen, I suppose.


```
package pl.ciruk.blog.noninst;

import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;

public abstract class NoninstantiableUtils {
	private NoninstantiableUtils() {
		throw new AssertionError();
	}
	
	public static <E> void instantiate(Class<E> classToBeInstantiated) {
		for (Constructor<?> constructor : classToBeInstantiated.getDeclaredConstructors()) {
			// No more private, sorry.
			constructor.setAccessible(true);
			
			try {
				Object[] parameters = new Object[constructor.getParameterCount()];
				E instance = (E) constructor.newInstance(parameters);
				
				System.out.println("Created instance: " + instance);
			} catch (InstantiationException | IllegalAccessException | IllegalArgumentException | InvocationTargetException e) {
				e.printStackTrace();
			}
		}
	}
	
	public static <E> void instantiateWithUnsafe(Class<E> classToBeInstantiated) {
		try {
			E instance = (E) getUnsafe().allocateInstance(classToBeInstantiated);
			System.out.println("Created instance: " + instance);
		} catch (InstantiationException | NoSuchFieldException | SecurityException | IllegalArgumentException | IllegalAccessException e) {
			e.printStackTrace();
		}
	}
	
	private static sun.misc.Unsafe getUnsafe() throws NoSuchFieldException, SecurityException, IllegalArgumentException, IllegalAccessException {
		Field theUnsafe = sun.misc.Unsafe.class.getDeclaredField("theUnsafe");
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
```

```
package pl.ciruk.blog.noninst;

public final class SampleUtilityClass {
	private SampleUtilityClass() {
		throw new AssertionError();
	}
}
```