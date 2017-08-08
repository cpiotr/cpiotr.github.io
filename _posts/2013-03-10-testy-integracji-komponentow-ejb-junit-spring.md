---
ID: 158
post_title: >
  Testy integracji komponentów EJB
  (JUnit, Spring)
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2013/03/testy-integracji-komponentow-ejb-junit-spring/
published: true
post_date: 2013-03-10 00:13:55
---
Nie każdy fragment kodu można zweryfikować za pomocą testów jednostkowych. Do oceny poprawności komunikacji implementowanej funkcjonalności z innymi komponentami służą <a href="http://en.wikipedia.org/wiki/Integration_testing" target="_blank">testy integracji</a> (<a href="http://stackoverflow.com/a/520116" target="_blank">http://stackoverflow.com/a/520116</a>). 

Różnica ta niekoniecznie musi się przekładać na podział technologiczny. Do testów integracji z powodzeniem można zaprząc bibliotekę <a href="http://junit.org/" target="_blank">JUnit</a>. Jeśli testy pokrywają operacje wykorzystujące dobrodziejstwa kontenera EJB (<a href="http://docs.oracle.com/javaee/5/tutorial/doc/bnabo.html#bnabp">http://docs.oracle.com/javaee/5/tutorial/doc/bnabo.html#bnabp</a>), istnieje mało wygodna możliwość osadzenia testów na docelowym serwerze aplikacji (np. <a href="http://www.junitee.org/" target="_blank">JUnit EE</a>), jednak komplikuje to proces wytwarzania, zamiast go wspierać.

Wygodnym podejściem jest natomiast użycie kontenera Spring do zapewnienia, na przykład, transakcyjności czy trwałości. Najprostsza forma realizacji opisanego podejścia zakłada istnienie zaledwie trzech plików: kod klasy testującej, deskryptor dla kontenera Spring oraz deskryptor mechanizmu trwałości (<code>persistance.xml</code>).

[sourcecode lang="xml"]
&lt;?xml version=&quot;1.0&quot; encoding=&quot;UTF-8&quot;?&gt;
&lt;persistence version=&quot;2.0&quot;&gt;
	&lt;!-- Nazwa jednostki trwalosci --&gt;
	&lt;persistence-unit name=&quot;film-ejb&quot;&gt;
		&lt;!-- Dostawca, ja wykorzystuje Hibernate --&gt;
		&lt;provider&gt;org.hibernate.ejb.HibernatePersistence&lt;/provider&gt;
		
		&lt;!-- Obslugiwane klasy --&gt;
		&lt;!-- Konieczne jest uwzglednienie klas bazowych --&gt;
		&lt;class&gt;pl.ciruk.films.entity.BaseEntity&lt;/class&gt;
		&lt;class&gt;pl.ciruk.films.entity.User&lt;/class&gt;
		&lt;class&gt;pl.ciruk.films.entity.Film&lt;/class&gt;

		&lt;!-- Wlasciwosci Hibernate --&gt;
		&lt;properties&gt;
			&lt;!-- Sterownik MySQL --&gt;
			&lt;property name=&quot;hibernate.connection.driver_class&quot; value=&quot;com.mysql.jdbc.Driver&quot;/&gt;
			&lt;property name=&quot;hibernate.dialect&quot; value=&quot;org.hibernate.dialect.MySQLDialect&quot;/&gt;
			
			&lt;!-- Adres URL bazy danych --&gt;
			&lt;property name=&quot;hibernate.connection.url&quot; value=&quot;jdbc:mysql://host:3306/dbschema&quot;/&gt;
			
			&lt;!-- Uzytkownik/haslo --&gt;
			&lt;property name=&quot;hibernate.connection.username&quot; value=&quot;user&quot;/&gt;
			&lt;property name=&quot;hibernate.connection.password&quot; value=&quot;password&quot;/&gt;
			
			&lt;property name=&quot;hibernate.archive.autodetection&quot; value=&quot;class, hbm&quot;/&gt;
			&lt;property name=&quot;hibernate.show_sql&quot; value=&quot;true&quot;/&gt;
			&lt;property name=&quot;hibernate.c3p0.min_size&quot; value=&quot;5&quot;/&gt;
			&lt;property name=&quot;hibernate.c3p0.max_size&quot; value=&quot;20&quot;/&gt;
			&lt;property name=&quot;hibernate.c3p0.timeout&quot; value=&quot;300&quot;/&gt;
			&lt;property name=&quot;hibernate.c3p0.max_statements&quot; value=&quot;50&quot;/&gt;
			&lt;property name=&quot;hibernate.c3p0.idle_test_period&quot; value=&quot;3000&quot;/&gt;
		&lt;/properties&gt;
	&lt;/persistence-unit&gt;
&lt;/persistence&gt;
[/sourcecode]

[sourcecode lang="xml"]
&lt;?xml version=&quot;1.0&quot; encoding=&quot;UTF-8&quot;?&gt;
&lt;beans&gt;
	&lt;!-- Interfejsy testowanych klas uslugowych EJB --&gt;
    &lt;bean id=&quot;FilmServiceLocal&quot; class=&quot;pl.ciruk.films.ejb.impl.FilmServiceBean&quot;&gt;&lt;/bean&gt;
    
	&lt;!-- Zarzadca encji --&gt;
    &lt;bean class=&quot;org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean&quot; id=&quot;entityManagerFactory&quot;&gt;
		&lt;!-- Nazwa jednostki trwalosci --&gt;
        &lt;property name=&quot;persistenceUnitName&quot; value=&quot;film-ejb&quot;/&gt;
		
		&lt;!-- Wskazanie deskryptora jednostki trwalosci --&gt;
        &lt;property name=&quot;persistenceXmlLocation&quot; value=&quot;classpath:TEST-INF/persistence.xml&quot;/&gt;
    &lt;/bean&gt;
    
	&lt;!-- Zarzadca transakcji --&gt;
    &lt;bean class=&quot;org.springframework.orm.jpa.JpaTransactionManager&quot; id=&quot;transactionManager&quot;&gt;
        &lt;property name=&quot;entityManagerFactory&quot; ref=&quot;entityManagerFactory&quot;/&gt;
    &lt;/bean&gt;
&lt;/beans&gt;
[/sourcecode]

[sourcecode lang="java"]
//pakiet i importy

// Uruchamianie testu w kontenerze Spring
@RunWith(SpringJUnit4ClassRunner.class)
// Wskazanie konfiguracji 
@ContextConfiguration(locations = {
	&quot;classpath:/TEST-INF/FilmServiceBeanTest-context.xml&quot;
}) 
public class FilmServiceBeanTest {

	/** Testowana klasa uslugowa. */
	@EJB
	FilmServiceLocal service;
	
	@PersistenceContext
	EntityManager em;
	
	/** Metoda sprawdzajaca zapis danych do bazy. */
	@Transactional
	@Test
	public void shouldSaveSampleFilm() {
		// Nowy testowy film
		Film f = new Film();
		f.setInsertionDate(new Date());
		f.setLabel(&quot;Testowy film&quot;);
		f.setTitle(&quot;Testowy tytuł&quot;);
		f.setType(FilmType.F);
		
		// Zapytanie wyszukujace film na podstawie tytulu i opisu
		Query query = em.createQuery(&quot;select f from Film f where f.title = :title and f.label = :label&quot;).setParameter(&quot;title&quot;, f.getTitle()).setParameter(&quot;label&quot;, f.getLabel());
		
		// Sprawdzenie, czy nie ma jeszcze w bazie testowego filmu
		List&lt;Film&gt; films = query.getResultList();
		assertNotNull(films);
		assertTrue(films.isEmpty());
		
		// Zapis testowego filmu
		boolean saved = service.save(f);
		assertTrue(saved);
		
		// Weryfikacja zapisu
		films = query.getResultList();
		assertNotNull(films);
		assertFalse(films.isEmpty());
		assertEquals(f, films.get(0));
	}
}
[/sourcecode]