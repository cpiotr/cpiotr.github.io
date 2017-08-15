---
ID: 315
post_title: 'What can&#8217;t JIT do?'
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2016-11-29 21:03:45
---
Just-In-Time compiler is a part of HotSpot's execution engine responsible for compiling hot methods to native code. It comes in two flavours which differ in compilation pace and optimization methods.
Client compiler (C1) starts compiling sooner to achieve better performance in short term. However, it makes decision based on constrained data - which can potentially degrade performance in the long run.
Server compiler (C2) collects data longer, thus it's able to detect hot spots with better probability. Although, it's not well-suited for short-running applications.
Commencing the release of Java 7, tiered compilation was introduced to combine client and server compilers. Such combination does not solve all the performance problems related to compilation. In specific scenarios better results are obtained by having this feature disabled.
You can take a peek under the hood of JIT by instrumenting the JVM to be a bit more verbose. The following set of parameters will result in compilation log file creation:
```
-XX:+UnlockDiagnosticVMOptions
-XX:+LogCompilation
-XX:+TraceClassLoading
-XX:+PrintAssembly
```
The file can be read by talented individuals or interpreted and presented to mere humans by handy tool called <a href="https://github.com/AdoptOpenJDK/jitwatch">JITwatch</a>.

Generated binary code resides in non-heap memory region called code cache. The size of code cache is limited, which means that JVM sometimes has to remove already compiled methods in order to make room for the hotter ones.
Shrinking free space is, however, only one reason for de-optimization. JIT can decide on its own, that optimization techniques applied to given method are in fact not helpful. Most optimizations introduced by C2 are speculative.
Having said that, JIT is robust and comprehensive - it has nearly 100 optimizations under its belt. Among them there is an entire group dedicated to removing instructions: autobox elimination, dead code elimination, null check elimination, etc. What JIT cannot do is to eliminate loop when <em>termination expression</em> (i.e. boolean condition in the loop) involves 64-bit value comparison. In that case, JIT fails to eliminate even empty loops. 

`For` loops are directly translated to bytecode instructions. The following listings present bytecode for sample empty loops when iterated over integer and long value.
They seem rather straightforward. In case of iteration which uses integer as an incremented value, JIT has no problems eliminating empty loop. Consider the listing below. The code was run with `-XX:+PrintCompilation` flag, which produces <a href="https://gist.github.com/chrisvest/2932907">peculiar output</a>.
After the first iteration, a call to `intLoop` is removed.
```
@Test(timeout = 100_000L)
public void shouldEliminateEmptyIntLoop() throws Exception {
	for (int i = 0; i < 100; i++) {
		runAndMeasureTime(this::intLoop);
	}
}

private void intLoop() {
	for (int i = 0; i < 20_000_000; i++) {
		// no-op
	}
}
```
```
    236  382 %     3       pl.ciruk.blog.jitest.EmptyLoopTest::intLoop @ 2 (15 bytes)
    236  383       3       pl.ciruk.blog.jitest.EmptyLoopTest::intLoop (15 bytes)
    236  384 %     4       pl.ciruk.blog.jitest.EmptyLoopTest::intLoop @ 2 (15 bytes)
    237  382 %     3       pl.ciruk.blog.jitest.EmptyLoopTest::intLoop @ -2 (15 bytes)   made not entrant
    237  384 %     4       pl.ciruk.blog.jitest.EmptyLoopTest::intLoop @ -2 (15 bytes)   made not entrant
2 ms
    237  385 %     4       pl.ciruk.blog.jitest.EmptyLoopTest::intLoop @ 2 (15 bytes)
    238  386       4       pl.ciruk.blog.jitest.EmptyLoopTest::intLoop (15 bytes)
1 ms
    238  383       3       pl.ciruk.blog.jitest.EmptyLoopTest::intLoop (15 bytes)   made not entrant
0 ms   
0 ms
0 ms
0 ms
...
```

However, when long values come into play, JIT seems helpless.
```
@Test(timeout = 100_000L)
public void shouldEliminateEmptyLongLoop() throws Exception {
	for (int i = 0; i < 100; i++) {
		runAndMeasureTime(this::longLoop);
	}
}
private void longLoop() {
	for (long i = 0; i < 20_000_000L; i++) {
		// no-op
	}
}
```
```
    231  391 %     3       pl.ciruk.blog.jitest.EmptyLoopTest::longLoop @ 2 (18 bytes)
    231  392       3       pl.ciruk.blog.jitest.EmptyLoopTest::longLoop (18 bytes)
    231  393 %     4       pl.ciruk.blog.jitest.EmptyLoopTest::longLoop @ 2 (18 bytes)
    231  391 %     3       pl.ciruk.blog.jitest.EmptyLoopTest::longLoop @ -2 (18 bytes)   made not entrant
    242  393 %     4       pl.ciruk.blog.jitest.EmptyLoopTest::longLoop @ -2 (18 bytes)   made not entrant
12 ms
    243  394 %     4       pl.ciruk.blog.jitest.EmptyLoopTest::longLoop @ 2 (18 bytes)
    243  395       4       pl.ciruk.blog.jitest.EmptyLoopTest::longLoop (18 bytes)
    244  392       3       pl.ciruk.blog.jitest.EmptyLoopTest::longLoop (18 bytes)   made not entrant
10 ms
6 ms
7 ms
7 ms
6 ms
8 ms
...
```

The same situation with JIT struggling to help, but without any result, can be observed when the value in a loop is an integer, but the termination clause contains comparison to a long one. 
```
@Test(timeout = 100_000L)
public void shouldEliminateEmptyIntToLongLoop() throws Exception {
	for (int i = 0; i < 100; i++) {
		runAndMeasureTime(this::intToLongLoop);
	}
}
private void intToLongLoop() {
	for (int i = 0; i < 20_000_000L; i++) {
		// no-op
	}
}
```

This strange misbehavior is really hard to understand. Bytecode generated by javac in case of integer in the loop is similar to the one generated for iterating over long.
```
void loop();
    descriptor: ()V
    flags:
    Code:
      stack=2, locals=2, args_size=1
         0: iconst_0
         1: istore_1
         2: iload_1
         3: ldc           #2                  // int 20000000
         5: if_icmpge     14
         8: iinc          1, 1
        11: goto          2
        14: return
```

The only difference is the way local variable gets incremented. There's no single bytecode instruction for incrementing a long value, so it needs to be replaced with load var, load one, add them and store the result. Still, it operates only on local variable. 
```
void loopOnlyLongs();
    descriptor: ()V
    flags:
    Code:
      stack=4, locals=3, args_size=1
         0: lconst_0
         1: lstore_1
         2: lload_1
         3: ldc2_w        #3                  // long 20000000l
         6: lcmp
         7: ifge          17
        10: lload_1
        11: lconst_1
        12: ladd
        13: lstore_1
        14: goto          2
        17: return
```