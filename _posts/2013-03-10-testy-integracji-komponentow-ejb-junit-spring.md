---
ID: 158
post_title: >
  Testy integracji komponentów EJB
  (JUnit, Spring)
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2013-03-10 00:13:55
---
Nie każdy fragment kodu można zweryfikować za pomocą testów jednostkowych. Do oceny poprawności komunikacji implementowanej funkcjonalności z innymi komponentami służą <a href="http://en.wikipedia.org/wiki/Integration_testing" target="_blank">testy integracji</a> (<a href="http://stackoverflow.com/a/520116" target="_blank">http://stackoverflow.com/a/520116</a>). 

Różnica ta niekoniecznie musi się przekładać na podział technologiczny. Do testów integracji z powodzeniem można zaprząc bibliotekę <a href="http://junit.org/" target="_blank">JUnit</a>. Jeśli testy pokrywają operacje wykorzystujące dobrodziejstwa kontenera EJB (<a href="http://docs.oracle.com/javaee/5/tutorial/doc/bnabo.html#bnabp">http://docs.oracle.com/javaee/5/tutorial/doc/bnabo.html#bnabp</a>), istnieje mało wygodna możliwość osadzenia testów na docelowym serwerze aplikacji (np. <a href="http://www.junitee.org/" target="_blank">JUnit EE</a>), jednak komplikuje to proces wytwarzania, zamiast go wspierać.

Wygodnym podejściem jest natomiast użycie kontenera Spring do zapewnienia, na przykład, transakcyjności czy trwałości. Najprostsza forma realizacji opisanego podejścia zakłada istnienie zaledwie trzech plików: kod klasy testującej, deskryptor dla kontenera Spring oraz deskryptor mechanizmu trwałości (`persistance.xml`).

```
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="2.0">
	<!-- Nazwa jednostki trwalosci -->
	<persistence-unit name="film-ejb">
		<!-- Dostawca, ja wykorzystuje Hibernate -->
		<provider>org.hibernate.ejb.HibernatePersistence</provider>
		
		<!-- Obslugiwane klasy -->
		<!-- Konieczne jest uwzglednienie klas bazowych -->
		<class>pl.ciruk.films.entity.BaseEntity</class>
		<class>pl.ciruk.films.entity.User</class>
		<class>pl.ciruk.films.entity.Film</class>

		<!-- Wlasciwosci Hibernate -->
		<properties>
			<!-- Sterownik MySQL -->
			<property name="hibernate.connection.driver_class" value="com.mysql.jdbc.Driver"/>
			<property name="hibernate.dialect" value="org.hibernate.dialect.MySQLDialect"/>
			
			<!-- Adres URL bazy danych -->
			<property name="hibernate.connection.url" value="jdbc:mysql://host:3306/dbschema"/>
			
			<!-- Uzytkownik/haslo -->
			<property name="hibernate.connection.username" value="user"/>
			<property name="hibernate.connection.password" value="password"/>
			
			<property name="hibernate.archive.autodetection" value="class, hbm"/>
			<property name="hibernate.show_sql" value="true"/>
			<property name="hibernate.c3p0.min_size" value="5"/>
			<property name="hibernate.c3p0.max_size" value="20"/>
			<property name="hibernate.c3p0.timeout" value="300"/>
			<property name="hibernate.c3p0.max_statements" value="50"/>
			<property name="hibernate.c3p0.idle_test_period" value="3000"/>
		</properties>
	</persistence-unit>
</persistence>
```

```
<?xml version="1.0" encoding="UTF-8"?>
<beans>
	<!-- Interfejsy testowanych klas uslugowych EJB -->
    <bean id="FilmServiceLocal" class="pl.ciruk.films.ejb.impl.FilmServiceBean"></bean>
    
	<!-- Zarzadca encji -->
    <bean class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean" id="entityManagerFactory">
		<!-- Nazwa jednostki trwalosci -->
        <property name="persistenceUnitName" value="film-ejb"/>
		
		<!-- Wskazanie deskryptora jednostki trwalosci -->
        <property name="persistenceXmlLocation" value="classpath:TEST-INF/persistence.xml"/>
    </bean>
    
	<!-- Zarzadca transakcji -->
    <bean class="org.springframework.orm.jpa.JpaTransactionManager" id="transactionManager">
        <property name="entityManagerFactory" ref="entityManagerFactory"/>
    </bean>
</beans>
```

```
//pakiet i importy

// Uruchamianie testu w kontenerze Spring
@RunWith(SpringJUnit4ClassRunner.class)
// Wskazanie konfiguracji 
@ContextConfiguration(locations = {
	"classpath:/TEST-INF/FilmServiceBeanTest-context.xml"
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
		f.setLabel("Testowy film");
		f.setTitle("Testowy tytuł");
		f.setType(FilmType.F);
		
		// Zapytanie wyszukujace film na podstawie tytulu i opisu
		Query query = em.createQuery("select f from Film f where f.title = :title and f.label = :label").setParameter("title", f.getTitle()).setParameter("label", f.getLabel());
		
		// Sprawdzenie, czy nie ma jeszcze w bazie testowego filmu
		List<Film> films = query.getResultList();
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
```