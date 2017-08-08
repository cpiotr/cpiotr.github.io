---
ID: 144
post_title: Metatabela ALL_TAB_COLUMNS
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2013/01/metatabela-all_tab_columns/
published: true
post_date: 2013-01-30 21:47:24
---
Komentowanie jest istotnym elementem pracy programisty i jednocześnie nagminnie pomijanym. O ile w kodzie aplikacji można natrafić na dryfujące w morzu instrukcji opisy, wyjaśnienia i zażalenia, o tyle w obiektach bazodanowych zjawisko to jest zdecydowanie rzadsze. 
Raz na jakiś zaniedbane tabele puchną i wypływają na powierzchnię, strasząc i zobowiązując do uzupełnienia.
Można, rzecz jasna, przejrzeć wszystkie struktury, wychwytując elementy pozbawione komentarza. Dla pozostałych, ceniących swój czas projektanci bazy danych Oracle przygotowali <em>metatabele</em>: <a href="http://docs.oracle.com/cd/B19306_01/server.102/b14237/statviews_2094.htm" title="ALL_TAB_COLUMNS" target="_blank">ALL_TAB_COLUMNS</a> oraz <a href="http://docs.oracle.com/cd/B28359_01/server.111/b28320/statviews_1036.htm" title="ALL_COL_COLUMNS" target="_blank">ALL_COL_COLUMNS</a>. 

[sourcecode lang="sql"]
SELECT
	tab_col.table_name, tab_col.column_name, TRIM(col_comm.comments)
FROM
	all_tab_columns tab_col
	JOIN all_col_comments col_comm ON (col_comm.table_name = tab_col.table_name AND col_comm.column_name = tab_col.column_name)
WHERE
	tab_col.owner = USER
	AND (col_comm.comments IS null OR LENGTH(TRIM(col_comm.comments)) = 0)
[/sourcecode]