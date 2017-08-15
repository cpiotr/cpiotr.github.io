---
ID: 76
post_title: Potoki systemowe w Javie
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2011-10-06 19:46:06
---
Raz na jakiś czas przychodzi moment, w którym trzeba porzucić przenośność kodu (w tym wypadku niezależność platformową) na rzecz wydajności, mniejszego nakładu pracy, czy jakichkolwiek innych powodów. Całkiem niedawno spotkałem się z sytuacją, w której konieczne było uruchomienie poleceń powłoki systemu Linux. 
Do uruchamiania poleceń systemowych w języku Java służy klasa `Runtime`. Przykładowa składnia wywołania pozwalającego wylistować zawartość bieżącego katalogu ma postać:
```
Process process = Runtime.getRuntime().exec("ls -l .");
```
Podane polecenie uruchamiane jest jako nowy proces, dlatego też wywołanie ma charakter asynchroniczny. Aby zmusić program do oczekiwania na zakończenie polecenia należy wykorzystać metodę `waitFor()` klasy `Process`.
```
Process process = Runtime.getRuntime().exec("ls -l .");
int result = process.waitFor();
```
Problem pojawia się, kiedy polecenie ma charakter wywołania potokowego, np. 
```
ls -l . | grep .txt
```
Aby umożliwić poprawne wykonanie tak zbudowanego polecenia, należy zastosować poniższy schemat:
```
String command = "ls -l . | grep .txt";
String[] commands = {"/bin/bash", "-c", command};
Process process = Runtime.getRuntime().exec("ls -l .");
int result = process.waitFor();
```