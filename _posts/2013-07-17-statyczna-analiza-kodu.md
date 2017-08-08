---
ID: 175
post_title: Statyczna analiza kodu
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2013/07/statyczna-analiza-kodu/
published: true
post_date: 2013-07-17 19:18:25
---
Statyczna analiza kodu źródłowego jest procesem weryfikacji programu prowadzonym bez konieczności jego uruchomienia. Bazuje na zestawie wzorców często popełnianych błędów i złych praktyk. Narzędzia wspierające statyczną analizę pozwalają na przeprowadzenie wstępnej <em>code review</em>, systematycznie używane prowadzą do poprawy jakości wytwarzanego oprogramowania.

Dobry i kompleksowy zestaw zapewniający znaczne pokrycie programistycznych niedoskonałości tworzą:
<ul>
	<li>PMD(<a href="http://pmd.sourceforge.net/" target="_blank">http://pmd.sourceforge.net/</a>) - operuje bezpośrednio na kodzie źródłowym, pomaga w wykryciu uchybień obejmujących nieprzestrzeganie konwencji nazewnictwa, zbyt długie listy parametrów w metodach, wysoką <a href="http://en.wikipedia.org/wiki/Cyclomatic_complexity" target="_blank">złożoność cyklomatyczną</a>, i wiele innych (<a href="http://pmd.sourceforge.net/pmd-5.0.4/rules/index.html" target="_blank">http://pmd.sourceforge.net/pmd-5.0.4/rules/index.html</a>)</li>
</ul>
<ul>
	<li>FindBugs(<a href="http://findbugs.sourceforge.net/" target="_blank">http://findbugs.sourceforge.net/</a>) - obszarem jego działania jest <em>bytecode</em>, w związku z czym umożliwia detekcję nieco innej grupy błędów: rozważa hierarchię klas (wykrywa <em>serializowalne</em> klasy, których klasa nadrzędna nie udostępnia bezparametrowego konstruktora), implementację metod klonujących, nieprawidłowe rzutowanie danych i wiele innych (<a href="http://findbugs.sourceforge.net/bugDescriptions.html" target="_blank">http://findbugs.sourceforge.net/bugDescriptions.html</a>).</li>
</ul>
Oba narzędzia dostępne są w postaci wtyczek do środowiska Eclipse.