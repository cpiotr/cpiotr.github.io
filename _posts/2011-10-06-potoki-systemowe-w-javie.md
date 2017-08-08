---
ID: 76
post_title: Potoki systemowe w Javie
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2011/10/potoki-systemowe-w-javie/
published: true
post_date: 2011-10-06 19:46:06
---
Raz na jakiś czas przychodzi moment, w którym trzeba porzucić przenośność kodu (w tym wypadku niezależność platformową) na rzecz wydajności, mniejszego nakładu pracy, czy jakichkolwiek innych powodów. Całkiem niedawno spotkałem się z sytuacją, w której konieczne było uruchomienie poleceń powłoki systemu Linux. 
Do uruchamiania poleceń systemowych w języku Java służy klasa <code>Runtime</code>. Przykładowa składnia wywołania pozwalającego wylistować zawartość bieżącego katalogu ma postać:
[sourcecode language="java"]
	Process process = Runtime.getRuntime().exec(&quot;ls -l .&quot;);
[/sourcecode]
Podane polecenie uruchamiane jest jako nowy proces, dlatego też wywołanie ma charakter asynchroniczny. Aby zmusić program do oczekiwania na zakończenie polecenia należy wykorzystać metodę <code>waitFor()</code> klasy <code>Process</code>.
[sourcecode language="java"]
	Process process = Runtime.getRuntime().exec(&quot;ls -l .&quot;);
	int result = process.waitFor();
[/sourcecode]
Problem pojawia się, kiedy polecenie ma charakter wywołania potokowego, np. 
[sourcecode language="bash"]
	ls -l . | grep .txt
[/sourcecode]
Aby umożliwić poprawne wykonanie tak zbudowanego polecenia, należy zastosować poniższy schemat:
[sourcecode language="java"]
	String command = &quot;ls -l . | grep .txt&quot;;
	String[] commands = {&quot;/bin/bash&quot;, &quot;-c&quot;, command};
	Process process = Runtime.getRuntime().exec(&quot;ls -l .&quot;);
	int result = process.waitFor();
[/sourcecode]