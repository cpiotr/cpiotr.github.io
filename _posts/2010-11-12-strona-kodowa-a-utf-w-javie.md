---
ID: 36
post_title: Strona kodowa, a UTF w Javie
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2010/11/strona-kodowa-a-utf-w-javie/
published: true
post_date: 2010-11-12 15:46:05
---
W portalu tworzonym w technologii JEE zaistniała niecodzienna sytuacja. Po dniu przerwy w działaniu, spowodowanym pracami administracyjnymi, do bazy danych zaczęły trafiać teksty bez polskich liter (w ich miejscu pojawiały się znaki zapytania). Podobno prace administracyjne polegały na zmianie dysków, więc gdzie tkwił problem?

Analiza logów pozwoliła stwierdzić, że tekst wpisany przez użytkownika na stronie internetowej został źle zakodowany już po stronie portalu, jednak w tym czasie nie dokonano żadnych zmian ze strony programistów.
Obiekty typu <code>String</code> w Javie są kodowane w <code>Unicode</code>, więc teoretycznie żadne konflikty nie powinny wystąpić. Jeśli jednak sekwencja bajtów zamieniana jest na tekst, Java <a href="http://download.oracle.com/javase/6/docs/api/java/lang/String.html#String%28byte[]%29">używa</a> domyślnego kodowania systemu, na którym jest uruchomiona maszyna wirtualna. W praktyce wygląda to tak, że jeżeli żaden znak ze strony kodowej nie odpowiada zadanym bajtom, Java zamienia je na znak zapytania. Rozwiązanie jest więc coraz bliżej.

Sprawdzone zostało kodowanie na środowisku wykonawczym portalu:
[sourcecode language="bash"]
echo $LANG
$&gt; en_EN.UTF-8
[/sourcecode]
Pierwszą myślą, która okazała się wystarczającym rozwiązaniem, była zmiana wartości zmiennej systemowej. W skrypcie uruchomieniowym Tomcata (<code>catalina.sh</code>) dodane została polecenie:
[sourcecode language="bash"]
export LANG=pl_PL.utf8
[/sourcecode]
Po zrestartowaniu serwera tomcat portal funkcjonował już poprawnie.