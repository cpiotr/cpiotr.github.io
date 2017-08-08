---
ID: 18
post_title: Problem dat w JSF 1.2
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2010/11/problem-dat-w-jsf-1-2/
published: true
post_date: 2010-11-04 00:02:54
---
Załóżmy, że mamy tabelę w bazie danych, która w jednej z kolumn przechowuje daty. Dokładność, jaką potrzebujemy to jeden dzień. Czas nie gra roli.

[sourcecode language="sql"]
CREATE TABLE IF NOT EXISTS `blog_login_history` (
	`lohi_last_login` date default NULL
);
[/sourcecode]

Tabelę wypełniamy danymi
[sourcecode language="sql"]
INSERT INTO 
	`blog_login_history` (`lohi_last_login`)
VALUES
	(date_format('2010-08-07', '%Y-%m-%d')),
	(date_format('2010-05-20', '%Y-%m-%d')),
	(date_format('2010-04-11', '%Y-%m-%d')),
	(date_format('2010-10-01', '%Y-%m-%d'))
;
[/sourcecode]

W aplikacji webowej pobieramy dane z bazy i chcemy je wyświetlić na stronie JSF. 
[sourcecode language="java"]
&lt;h:outputText value=&quot;#{bean.loginHistory.lastLogin}&quot; /&gt;
[/sourcecode]

Efekt zapewne nie odpowiada większości użytkowników, ponieważ wypisany zostaje napis z metody <code>Date.toString()</code>. JSF posiada zestaw komponentów wspomagających programistów, w skład którego wchodzi konwerter <code>javax.faces.convert.DateTimeConverter</code>. Spójrzmy więc, jak przedstawia się on w akcji.
<pre lang="Java" class="1">
<h:outputText value="#{bean.loginHistory.lastLogin}">
	<f:convertDateTime pattern="yyyy-MM-dd" />
</h:outputText>
[/sourcecode]
Wynik jest zadowalający, gdyby nie pewien szkopuł. Jeśli serwer webowy znajduje się w innej strefie czasowej, niż GMT, wyświetlone daty mogą nie zgadzać się z przechowywanymi w bazie danych.
O ile samo wyświetlanie nie jest problemem krytycznym, o tyle zapis błędnych danych takim się staje. W sytuacji, gdy użytkownik pobierze dane z bazy, nie zmodyfikuje daty i wykona ich ponowny zapis - w bazie danych zostaną zapisane daty inne, niż wczytane.

Dzieje się tak dlatego, że konwerter <code>javax.faces.convert.DateTimeConverter</code> domyślnie używa strefy czasowej GMT. 
<pre lang="Java" class="182">
private static final TimeZone DEFAULT_TIME_ZONE = TimeZone.getTimeZone("GMT");
[/sourcecode]

Istnieje wiele sposobów rozwiązania opisanego problemu, m.in.:<ul>
	<li>utworzenie własnego konwertera dat (<a href="http://www.ibm.com/developerworks/java/library/j-jsf3/#N10239">link</a>)</li>
	<li>podanie w standardowym konwerterze strefy czasowej
[sourcecode language="java"]
&lt;f:convertDateTime pattern=&quot;yyyy-MM-dd&quot; timeZone=&quot;GMT+2&quot;/&gt;
[/sourcecode]
W tym wypadku warto w globalnym beanie stworzyć parametr wyznaczający domyślną strefę czasową serwera:
<pre lang="Java" class="1">
import java.util.TimeZone;

public class Globals {
	public TimeZone getTimeZone() {
		return TimeZone.getDefault();
	}
}
[/sourcecode]

Wówczas wywołanie konwertera przybierze postać:
[sourcecode language="java"]
&lt;f:convertDateTime pattern=&quot;yyyy-MM-dd&quot; timeZone=&quot;#{globals.timeZone}&quot;/&gt;
[/sourcecode]
</li>
</ul>