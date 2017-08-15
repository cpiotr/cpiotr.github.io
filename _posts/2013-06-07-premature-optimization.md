---
ID: 165
post_title: Premature Optimization
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2013-06-07 00:07:10
---
Optymalizacja kodu potrafi być jedną z ciekawszych składowych pracy związanej z programowaniem. Wysiłek, który przynosi łatwo mierzalne (np. czas wykonania) efekty jest niewątpliwie zadowalający, sprzyja wzrostowi ego i poczuciu zadowolenia.
Optymalizacja nie zawsze ma jednak sens. Donald Knuth określił <a href="http://c2.com/cgi/wiki?PrematureOptimization" target="_blank">przedwczesną optymalizację</a> źródłem wszelkiego zła (<a href="http://en.wikiquote.org/wiki/Donald_Knuth" target="_blank">link</a>). Mało który projekt informatyczny jest odrzucany z powodu słabej wydajności. Przyczyną górującą w niechlubnym rankingu przeszkód w realizacji projektów jest nieterminowość.
Rozsądne jest zatem podejście spełniające kryterium czasowe: najpierw dostarcz rozwiązanie, potem je optymalizuj. Taki mechanizm rozwiązywania problemów jest wykorzystywany przez człowieka na różnych płaszczyznach życia, gdzie przynosi <a href="http://blog.aaroniba.net/2011/07/06/a-lesson-from-my-ios-users-they-dont-teach-at-mit/" target="_blank">wymierne rezultaty</a>.
Nie miałbym sobie wiele do zarzucenia, gdybym zawsze postępował w myśl tej prostej zasady. Kiedy zasiadałem do zadania optymalizacji procedury bazodanowej, myślałem, że problem w tym przypadku mnie nie dotyczy. Ktoś w końcu podjął decyzję o konieczności przyspieszenia działania tego fragmentu kodu. Zapomniałem jednak, że mogę w ferworze walki ulepszyć nie to co trzeba.
Procedura bazodanowa składała się z dwóch zagnieżdżonych pętli, wewnątrz których zbierane były potrzebne dane i następował zapis nowego rekordu do tabeli. Postępując zgodnie z <a href="http://docs.oracle.com/cd/B13789_01/appdev.101/b10807/12_tune.htm" target="_blank">wytycznymi</a>, postanowiłem zredukować zagnieżdżone pętle i zamienić proste iterowanie na zbiorcze pobieranie rekordów z kursora (`BULK COLLECT`).
Pchnięty ciekawością, podjąłem decyzję o weryfikacji zastosowanych ulepszeń. O ile redukcja zagnieżdżonych pętli iterujących po kursorach przyniosła zmniejszenie czasu wykonania o rząd wielkości, to zastosowanie zbiorczego pobierania danych miało znikomy efekt. Dlaczego?
Wykonałem prosty test. Skonstruowałem trzy zapytania SQL:
<ul>
	<li>select na pojedynczej tabeli zwracający 13867 rekordów</li>
	<li>select z joinem na dwóch tabelach zwracający 31253 rekordów</li>
	<li>select z joinami na trzech tableach zwracający 103033 rekordów</li>
</ul>
Stworzyłem dwie procedury: jedną z prostymi pętlami iterującymi po kursorze z zapytaniem (SIMPLE LOOP) oraz jej zoptymalizowany odpowiednik ze zbiorczym pobieraniem danych (BULK COLLECT).
Pojedynczy test składał się ze 100 następujących po sobie wywołań procedury. Dla każdej procedury test został wykonany po 5 razy.
Dla każdego z testowanych zbiorów danych, rezultaty były bardzo zbliżone. Uśrednione wyniki znajdują się na poniższym wykresie.
<p style="text-align: center;">
<a href="http://ciruk.pl/blog/wp-content/uploads/2013/06/plsql_loops_cmp.png" target="_blank"><img class="size-medium wp-image-166 aligncenter" style="border: 1px solid black;" alt="plsql_loops_cmp" src="http://ciruk.pl/blog/wp-content/uploads/2013/06/plsql_loops_cmp-300x168.png" width="300" height="168" /></a></p>
Okazało się, że przyczyną tak szybkiego wykonania prostych pętli jest globalny poziom optymalizacji w silniku bazy danych, który powodował automatyczną zamianę na pętlę typu BULK-COLLECT (<a href="http://plsql-challenge.blogspot.com/2011/01/cursor-for-loop-automatic-optimization.html" target="_blank">link</a>).