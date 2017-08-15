---
ID: 347
post_title: >
  Observing class loading preparation
  phase
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2017-02-06 20:37:53
---
During `javac` compilation, Java class is translated into a sequence of instructions and corresponding <em>metadata</em>. This intermediate representations allows JVM to dynamically extend the runtime environment with custom classes and execute code created in different languages (e.g. Scala or groovy). A `*.class` file can be considered as a key-value pair which has a specific name and corresponding binary representation.
To be able to execute any code, JVM needs to find that representation and read it, which is called loading. 
Loaded class is not usable at that point, because it may be invalid or it may refer to external types. Putting the loaded class in the runtime context is called linking. 
The next step is to initialize static variables for the class and run static initialization code blocks.

The whole process is described in details in <a href="https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html">chapter 5 of Java Language Specifiaction</a>. The topic is covered eagerly in countless articles and books. It seems, however, that one of the subphases of the linking phase does not get as much coverage as the others. 

During the linking phase there are different activities performed by the JVM. Due to security reasons the first subphase in responsible for extensive bytecode verification. Preparation consists of static field creation in initialization of their default values (for example: 0 for `int`, `false` for `boolean`, etc.). Constraints are imposed on the overridden methods return types as well. Finally, all types referenced by the class being linked are resolved.

The subphase responsible for initializing static fields to its default value is often forgotten. While imposing type constraints on methods seems like dark arts from high-level code perspective, the step when the defaults are set is tangible and can be observed. 
The following sample is by no means a good coding practice since it might introduce unexpected behavior. The class presented below holds an instance of itself as a static member. It has additional static field and a non-static field as well. Non-static field refers to the static one, which at that point is initialized to its default value, zero. Changing the order of static fields would eliminate the error. 

```
public class UnexpectedValue {
    static final UnexpectedValue instance = new UnexpectedValue();

    static int DEFAULT_VALUE = 123;

    int member;

    public UnexpectedValue() {
        this.member = DEFAULT_VALUE * 10;
    }
}
```

While we're tempted to think that the `valueOfMember` should be set to `1230`, it actually holds 0, because of unexpected value from `DEFAULT_VALUE`.
```
@Test
public void shouldResolveToDefaultIntValue() throws Exception {
    int valueOfMember = UnexpectedValue.instance.member;

    assertThat(valueOfMember, is(equalTo(defaultIntValue())));
}

int defaultIntValue() {
    return 0;
}
```