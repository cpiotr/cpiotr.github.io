---
ID: 240
post_title: Parallel queries with Hibernate and C3P0
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2015/02/parallel-queries-with-hibernate-and-c3p0/
published: true
post_date: 2015-02-27 11:23:50
---
It's not always profitable to process data in parallel. Context switching and communication does not come free of charge. One must look for tasks that are IO-bound, because all the time CPU is spending waiting for data, can be used differently.
Assumptions must be verified, we don't want to knock at the wrong door. Every profiler can provide a view inside application threads, and once we've spotted the sluggish one, we're good to go.
I've recently encountered a Java 7 application (so no parallel streams at the moment) that reads around 100k entities from database, process it a little and output a CSV file. The process, however, took slightly more than 40 minutes, because for each record a custom DB function was called. Let's say it's not the fastest one.
Java Mission Control confirmed that most of that time is spent waiting for the database. In this context, the database was read-only. There were no transactions involved, so exploiting multiple threads for parallel reads was a natural step forward.

A number of database sessions is required to execute SQL queries in parallel. Connection pooling mechanism (such as C3P0) allows working with multiple sessions at once.
[sourcecode]
&lt;dependency&gt;
	&lt;groupId&gt;org.hibernate&lt;/groupId&gt;
	&lt;artifactId&gt;hibernate-c3p0&lt;/artifactId&gt;
	&lt;version&gt;4.3.8.Final&lt;/version&gt;
&lt;/dependency&gt;
[/sourcecode]

[sourcecode lang="java"]
Query idsQuery = entityManager.createNamedQuery(Entity.NamedQueries.GET_ALL_IDS);

return getByIds(idsQuery.getResultList());
[/sourcecode]

[sourcecode lang="java"]
private List&lt;Entity&gt; getByIds(List&lt;String&gt; entitiesIds) {

	Collection&lt;FetchListOfEntities&gt; tasks = transform(
			Lists.partition(entitiesIds, 500),
			new Function&lt;List&lt;String&gt;, FetchListOfEntities&gt;() {
				@Override
				public FetchListOfEntities apply(List&lt;String&gt; ids) {
					return FetchListOfEntities.by(ids);
				}
			}
	);

	List&lt;Entity&gt; entities = Lists.newArrayListWithExpectedSize(entitiesIds.size());
	try {
		for (Future&lt;Collection&lt;Entity&gt;&gt; taskResult : executorService.invokeAll(tasks)) {
			entities.addAll(taskResult.get());
		}
		executorService.shutdown();
		executorService.awaitTermination(15L, TimeUnit.SECONDS);
	} catch (InterruptedException e) {
		LOG.error(&quot;getByIds - interrupted&quot;, e);
	} catch (ExecutionException e) {
		LOG.error(&quot;getByIds - Error in FetchListOfEntities&quot;, e);
	}

	return entities;
}
[/sourcecode]

[sourcecode lang="java"]
public class FetchListOfEntities implements Callable&lt;Collection&lt;Entity&gt;&gt; {
	private EntityManagerFactory emf;
	private Collection&lt;String&gt; ids;

	static FetchListOfEntities by(Collection&lt;String&gt; ids) {
		FetchListOfEntities worker = new FetchListOfEntities();
		worker.ids = ids;
		return worker;
	}

	@Override
    public Collection&lt;Entity&gt; call() throws Exception {
		return retrieveEntitiesInSeparateSession();
	}

	private Collection&lt;Entity&gt; retrieveEntitiesInSeparateSession() {
		EntityManager em = emf.createEntityManager();

		Collection&lt;Entity&gt; entities = Collections.emptyList();
		try {
			entities = em.createNamedQuery(Entity.NamedQueries.GET_BY_IDS)
					.setParameter(&quot;ids&quot;, ids)
					.getResultList();
		} finally {
			em.close();
		}
		return entities;
	}
}
[/sourcecode]

Such changes resulted in an observable speedup. This time processing took only 7 minutes, given 16 threads were used on a 8-core machine.

Sample C3P0 properties can be found below.
[sourcecode lang="java"]
c3p0.acquireIncrement=5
c3p0.idleConnectionTestPeriod=100
c3p0.initialPoolSize=20
c3p0.maxPoolSize=100
c3p0.maxIdleTime=300
c3p0.maxStatements=50
c3p0.minPoolSize=10
c3p0.debugUnreturnedConnectionStackTraces=true
c3p0.unreturnedConnectionTimeout=3600
[/sourcecode]