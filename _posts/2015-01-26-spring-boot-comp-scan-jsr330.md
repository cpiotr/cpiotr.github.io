---
ID: 235
post_title: Spring Boot, @ComponentScan and JSR-330
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2015-01-26 20:57:13
---
There is always a slight fade of uncertainty in an automated behaviour. Something is sacrificed, every time you accept a given way of working as-is, without understanding or a few skeptical thoughts. Apart from solving performance issues, what explicit dependency injection frameworks (i.e <a href="http://square.github.io/dagger/" target="_blank">Dagger</a> or <a href="https://github.com/google/guice" target="_blank">Guice</a>) do, is assuring the state of things. You know exactly what will be injected for given interface.
<a href="http://projects.spring.io/spring-boot/" target="_blank">Spring Boot</a> in an excellent framework for rapid microservices development, which provides both explicit and implicit configuration of injectable components. Choosing the latter might turn out in an occasional shrug, unless you structure your packages according to <a href="http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-structuring-your-code" target="_blank">the documentation</a>.
It turns out, that by default <a href="http://projects.spring.io/spring-boot/" target="_blank">Spring Boot</a> scans through packages located below class annotated with no-parameter `@ComponentScan` in the class hierarchy. What's more, Spring provides a shorthand for annotations commonly used togheter - succint `@SpringBootApplication` can be used instead of

```
@Configuration
@ComponentScan
@EnableAutoConfiguration
```

While reasonable, not being aware of this fact guarantees a handful of intensive moments with console output, StackOverflow and documentation.

Spring Boot supports Dependecy Injection for Java (<a href="https://jcp.org/en/jsr/detail?id=330" target="_blank">JSR-330</a>), but it requires additional library. The following maven dependency is perfectly suitable:

```
<dependency>
	<groupId>javax.inject</groupId>
	<artifactId>javax.inject</artifactId>
	<version>1</version>
</dependency>
```