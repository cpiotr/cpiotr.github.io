---
ID: 100
post_title: 'ORA-01704: string literal too long'
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2012-10-16 20:10:08
---
Jak długi może być literał w PL-SQL? Nigdy wcześniej nie stawiałem sobie podobnego pytania (zupełnie nie rozumiem, dlaczego), jednak aktualizując naprawdę duży blok tekstu przechowywany w bazie danych natrafiłem na problem.
Problem nazywa się `ORA-01704: string literal too long` i oznacza, że literał nie może być dłuższy niż 4000 znaków. To dużo, ale niestety nie wystarczająco.
Pierwszym pomysłem rozwikłania tej rzuconej przez Oracle kłody była konkatenacja literałów.
```
UPDATE
	db_table
SET
	tbl_dane = to_clob('Część tekstu...' || 'Kolejna część' || /* itd. */)
WHERE
	tbl_code = 'TAJNY_KOD'
```
Jednak nie tędy droga: rezultatem konkatenacji stałych literałów jest również literał. Na szczęście okazało się, że CLOBy również można łączyć.
```
UPDATE
	db_table
SET
	tbl_dane = to_clob('Część tekstu...') || to_clob('Kolejna część') /* itd. */
WHERE
	tbl_code = 'TAJNY_KOD'
```