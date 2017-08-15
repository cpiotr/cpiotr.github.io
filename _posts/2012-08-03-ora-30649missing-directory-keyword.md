---
ID: 91
post_title: ORA-30649:missing directory keyword
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2012-08-03 21:25:44
---
Zgodnie ze standardem SQL polecenie realizujące funkcjonalność dodawania kolumny do tabeli ma postać:
```
ALTER TABLE nazwa_tabeli
ADD nazwa_kolumny typ_danych
```

Jednak wywołanie polecenia w DBMS Oracle w następującej postaci:
```
ALTER TABLE nazwa_tabeli
ADD nazwa_kolumny typ_danych not null default 'T'
```
kończy się błędem z komunikatem
```
'ORA-30649:missing directory keyword'
```

Niepisanym standardem dla komunikatów o błędach płynących z systemu zarządzania bazą danych jest ich dyskretna niejednoznaczność.
Jak się okazuje, oczekiwana składnia polecenia różni się kolejnością atrybutów dodatkowych.

Prawidłowy zapis wygląda następująco:
```
ALTER TABLE nazwa_tabeli
ADD nazwa_kolumny typ_danych default 'T' not null
```