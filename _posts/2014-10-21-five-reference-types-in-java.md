---
ID: 225
post_title: >
  There are (at least) five reference
  types in Java
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2014/10/five-reference-types-in-java/
published: true
post_date: 2014-10-21 11:06:00
---
There are some fancy reference types in Java beside the commonly used strong references. 
A great effort have been put to properly describe <a href="http://stackoverflow.com/questions/9809074/java-difference-between-strong-soft-weak-phantom-reference" target="_blank">three indefinite reference types</a>, which are a part of public API: <a href="http://docs.oracle.com/javase/8/docs/api/java/lang/ref/SoftReference.html" target="_blank">SoftReferences</a>, <a href="http://docs.oracle.com/javase/8/docs/api/java/lang/ref/WeakReference.html" target="_blank">WeakReferences</a> and <a href="http://docs.oracle.com/javase/8/docs/api/java/lang/ref/PhantomReference.html" target="_blank">PhatomReferences</a>. I'm not here to introduce them one more time, no worries.
I would only like to point out one additional reference type residing in <code>java.lang.ref</code> package, for the sake of completeness.

<code>FinalReference </code>and utility class <code>Finalizer</code>, which extends it, both are package protected. <code>Finalizers </code>are used by JVM to keep track of instances of classes which override the <code>finalize()</code> method, surprisingly.

To spot the little devils, one must examine the heap during program execution. Finalizers are created while your objects are instantiated and they keep a reference to them. Which means, they have a slight impact on memory footprint, and therefore on Garbage Collection.
Such impact is illustrated by the example below.

[sourcecode lang="java"]
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;


public class FinalizerTest {
	@Override
	protected void finalize() throws Throwable {
		super.finalize();
	}
	
	public static final void main(String[] args) throws Exception {
		BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
		
		for (int i = 0; i &lt; 10_000; i++) {
			FinalizerTest ft = new FinalizerTest();
		}
		
		reader.readLine();
		System.gc();
		
		reader.readLine();
	}
}
[/sourcecode]

Let's compile and run the program.
[sourcecode lang="java"]
bash.exe&quot;-3.1$ javac FinalizerTest.java
bash.exe&quot;-3.1$ java -cp . -verbose:gc -XX:+PrintGCTimeStamps -XX:+PrintGCDetails FinalizerTest

0.073: [GC (Allocation Failure) [PSYoungGen: 512K-&gt;448K(1024K)] 512K-&gt;456K(126464K), 0.0024916 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
[/sourcecode]

While the program waits for user input, it's a good time to get process id for further investigation.
[sourcecode lang="java"]
bash.exe&quot;-3.1$ jps
8708 FinalizerTest
5560 Jps
8120 org.eclipse.equinox.launcher_1.3.0.v20140415-2008.jar
[/sourcecode]

All right, 8708 it is. With default heap size settings, young generation GC should have already been performed, which can be verified with jstat.
[sourcecode lang="java"]
bash.exe&quot;-3.1$ jstat -gccause 8708
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC

  0.00  87.51  97.17   0.01  49.51  51.50      1    0.002     0    0.000    0.002 Allocation Failure   No GC
[/sourcecode]

Heap (filtered) looks as follow
[sourcecode lang="java"]
bash.exe&quot;-3.1$ jmap -histo 8708 | grep '\.Finalizer\|Test'
   1:         10006         400240  java.lang.ref.Finalizer
   2:         10000         160000  FinalizerTest
  41:             1            384  java.lang.ref.Finalizer$FinalizerThread
[/sourcecode]

None of custom objects was swept yet. OK, let's hit enter to trigger Full GC.
[sourcecode lang="java"]
70.283: [GC (System.gc()) [PSYoungGen: 945K-&gt;496K(1536K)] 953K-&gt;792K(126976K), 0.0054988 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
70.289: [Full GC (System.gc()) [PSYoungGen: 496K-&gt;0K(1536K)] [ParOldGen: 296K-&gt;771K(125440K)] 792K-&gt;771K(126976K), [Metaspace: 2415K-&gt;2415K(1056768K)], 0.0185376 secs] [Times: user=0.03 sys=0.00, real=0.02 secs]
[/sourcecode]

jstat can confirm, what jvm already printed.
[sourcecode lang="java"]
bash.exe&quot;-3.1$ jstat -gccause 8708
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC

  0.00   0.00   1.73   0.61  49.66  51.61      2    0.006     1    0.018    0.024 System.gc()          No GC
[/sourcecode]

Histogram of classes on heap now looks as follow
[sourcecode lang="java"]
bash.exe&quot;-3.1$ jmap -histo 8708 | grep '\.Finalizer\|Test'
   1:          8622         344880  java.lang.ref.Finalizer
   2:          8618         137888  FinalizerTest
  41:             1            384  java.lang.ref.Finalizer$FinalizerThread
[/sourcecode]
We didn't get rid of all the obsolete references, some additional cycles are required to clean up the mess.

The same class but without implementation of <code>finalize()</code> method is cleaned up rather seamlessly.
[sourcecode lang="java"]
# After young generation GC
   3:          5160          82560  FinalizerTest
  40:             1            384  java.lang.ref.Finalizer$FinalizerThread
  49:             6            240  java.lang.ref.Finalizer
  
# After full GC
  39:             1            384  java.lang.ref.Finalizer$FinalizerThread
  53:             4            160  java.lang.ref.Finalizer
[/sourcecode]