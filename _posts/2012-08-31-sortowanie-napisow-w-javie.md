---
ID: 94
post_title: Sortowanie napisów w Javie
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2012/08/sortowanie-napisow-w-javie/
published: true
post_date: 2012-08-31 21:26:15
---
Wydawać by się mogło, że sortowanie napisów w UTF-8 jest już tematem tak znanym, że podejmowanie tego tematu zakrawa na powielanie i odkrywanie koła na nowo. Niedawno jednak spotkałem się z listą  napisów posortowaną alfabetycznie, z tym małym wyjątkiem, że zdanie zaczynające się od "Łatwo zauważyć..." znajdowało się na szarym końcu, zaraz za zdaniem "Zaobserwowano bowiem...".
Polskie znaki diakrytyczne po raz kolejny dały o sobie znać.

Często wykorzystywanym rozwiązaniem problemu sortowania tekstu jest scedowanie zadanie na mechanizm bazy danych (<code>... ORDER BY ...</code>) lub wykorzystanie własnego alfabetu (<a href="http://stackoverflow.com/questions/10645986/custom-sort-python" title="przykład" target="_blank">przykład</a>).
Język Java udostępnia jednak specjalną klasę porównującą tekst na podstawie podanych danych lokalizacyjnych - <code>String Collator</code> (<a href="http://download.java.net/jdk7/archive/b123/docs/api/java/text/Collator.html" target="_blank">link</a>).
Implementuje ona interfejs <code>Comparator</code>, toteż może z powodzeniem zostać wykorzystana podczas porządkowania napisów.

Przykładowe wykorzystanie:
[sourcecode language="java"]
List&lt;String&gt; data = new ArrayList&lt;String&gt;();
data.add(&quot;Zażółć&quot;);
data.add(&quot;Gęślą&quot;);
data.add(&quot;Jaźń&quot;);
data.add(&quot;Łotczepsię&quot;);

Collections.sort(data, Collator.getInstance(new Locale(&quot;PL&quot;)));
System.out.println(data);
[/sourcecode]

Wynikiem powyższego fragmentu kodu będzie:
[sourcecode language="java"]
[Gęślą, Jaźń, Łotczepsię, Zażółć]
[/sourcecode]