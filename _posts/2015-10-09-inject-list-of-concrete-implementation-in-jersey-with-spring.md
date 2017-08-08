---
ID: 269
post_title: >
  Inject list of concrete implementation
  in Jersey with Spring
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2015/10/inject-list-of-concrete-implementation-in-jersey-with-spring/
published: true
post_date: 2015-10-09 21:26:08
---
Spring collection injection in Jersey resources

One of the greatest virtues of Spring Framework is its comprehensiveness - it tries to incorporate many approaches to a given issue.  In case of dependency injections, both proprietary and JSR-330 annotations are supported.
Same bias applies to Spring Boot, which comes bundled with an extensive set of preconfigured components: one can choose to leverage JAX-RS restful services over Spring MVC ones. 
There are <a href="https://spring.io/blog/2014/11/23/bootiful-java-ee-support-in-spring-boot-1-2">blog posts</a> that illustrate the sheer simplicity of integration of Spring Boot and Jersey (JAX-RS reference implementation). And it really works pretty neat, you can define beans in a Spring <code>@Configuration</code> and they will be injected into your <code>@Named</code> Jersey resource.
There is, however, at least one excellent feature that is not supported in such setup. 
Provided that you have an interface and a number of concrete classes available as injectable beans, <a href="http://docs.spring.io/spring-framework/docs/current/spring-framework-reference/html/beans.html#beans-generics-as-qualifiers">you can inject all of them as list</a>. Sadly, this splendid functionality is not supported when using Jersey with Spring container and here's why.
Jersey uses <a href="https://jersey.java.net/documentation/latest/ioc.html">its own</a> dependency injection container called <a href="https://hk2.java.net/">HK2</a>, which is a JSR-330 implementation. 
Despite having such IoC mechanism, Jersey contains a bridge to Spring, which allows to fetch beans from the context. So what's the problem?
HK2 can let Spring manage its beans, but only if they're annotated with <code><a href="https://github.com/jersey/jersey/blob/master/ext/spring4/src/main/java/org/glassfish/jersey/server/spring/SpringComponentProvider.java">@Component</a></code>. Otherwise, a default Jersey scope (<code>REQUEST</code>) is assumed and dependencies are resolved by HK2. And HK2 does not have the possibility to collect concrete implementation classes into a list.
It means that the following class will work as expected, creating one instance of resource managed by Spring:
[sourcecode lang="java"]
@Path(ComponentIdentity.PATH)
@Component
public class ComponentIdentity {

	public static final String PATH = &quot;/identity/component&quot;;

	private List&lt;Logger&gt; loggers;

	@Autowired
	public ComponentIdentity(List&lt;Logger&gt; loggers) {
		this.loggers = loggers;
	}

	@GET
	@Produces(MediaType.APPLICATION_JSON)
	public Response get() {
		return Response.ok(hashCode()).build();
	}

	@GET
	@Path(&quot;/numberOfLoggers&quot;)
	@Produces(MediaType.APPLICATION_JSON)
	public Response numberOfInjectedLoggers() {
		return Response.ok(loggers.stream().count()).build();
	}
}
[/sourcecode]

But the next example results in one instance being created by Spring at the application start-up (managed by Spring Container), and one instance being created for every request to the resource (managed by HK2). 
[sourcecode lang="java"]
@Path(NamedIdentity.PATH)
@Named
public class NamedIdentity {

	public static final String PATH = &quot;/identity/named&quot;;

	private List&lt;Logger&gt; loggers;

	@Inject
	public NamedIdentity(List&lt;Logger&gt; loggers) {
		this.loggers = loggers;
	}

	@GET
	@Produces(MediaType.APPLICATION_JSON)
	public Response get() {
		return Response.ok(hashCode()).build();
	}

	@GET
	@Path(&quot;/numberOfLoggers&quot;)
	@Produces(MediaType.APPLICATION_JSON)
	public Response numberOfInjectedLoggers() {
		return Response.ok(loggers.stream().count()).build();
	}
}
[/sourcecode]

If using <code>@Component</code> somehow hurts your feelings you can leverage <code>@PostConstruct</code> lifecycle hook to store Spring-managed instance in a static context and delegate further calls to it.
[sourcecode lang="java"]
@Path(NamedIdentity.PATH)
@Named
@Singleton
public class NamedIdentity {

	public static final String PATH = &quot;/identity/named&quot;;
	
	public static NamedIdentity SELF;

	private List&lt;Logger&gt; loggers;

	@Inject
	public NamedIdentity(List&lt;Logger&gt; loggers) {
		this.loggers = Optional.ofNullable(SELF)
				.map(ni -&gt; ni.loggers)
				.orElse(loggers);
	}

	@PostConstruct
	void keepFirstInstance() {
		if (SELF == null) {
			SELF = this;
		}
	}

	@GET
	@Produces(MediaType.APPLICATION_JSON)
	public Response get() {
		return Response.ok(hashCode()).build();
	}

	@GET
	@Path(&quot;/numberOfLoggers&quot;)
	@Produces(MediaType.APPLICATION_JSON)
	public Response numberOfInjectedLoggers() {
		return Response.ok(loggers.stream().count()).build();
	}
}

[/sourcecode]