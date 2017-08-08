---
ID: 91
post_title: ORA-30649:missing directory keyword
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2012/08/ora-30649missing-directory-keyword/
published: true
post_date: 2012-08-03 21:25:44
---
Zgodnie ze standardem SQL polecenie realizujące funkcjonalność dodawania kolumny do tabeli ma postać:
[sourcecode language="sql"]
ALTER TABLE nazwa_tabeli
ADD nazwa_kolumny typ_danych
[/sourcecode]

Jednak wywołanie polecenia w DBMS Oracle w następującej postaci:
[sourcecode language="sql"]
ALTER TABLE nazwa_tabeli
ADD nazwa_kolumny typ_danych not null default 'T'
[/sourcecode]
kończy się błędem z komunikatem
[sourcecode language="plain"]
'ORA-30649:missing directory keyword'
[/sourcecode]

Niepisanym standardem dla komunikatów o błędach płynących z systemu zarządzania bazą danych jest ich dyskretna niejednoznaczność.
Jak się okazuje, oczekiwana składnia polecenia różni się kolejnością atrybutów dodatkowych.

Prawidłowy zapis wygląda następująco:
[sourcecode language="sql"]
ALTER TABLE nazwa_tabeli
ADD nazwa_kolumny typ_danych default 'T' not null
[/sourcecode]