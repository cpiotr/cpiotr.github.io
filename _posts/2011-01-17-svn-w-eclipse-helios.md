---
ID: 45
post_title: SVN w Eclipse Helios
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2011-01-17 03:25:08
---
Środowisko Eclipse domyślnie nie zawiera wtyczki do komunikacji z repozytorium SVN. Dostępne są dwie najpopularniejsze: <a href="http://subclipse.tigris.org/">subclipse</a> oraz <a href="http://www.eclipse.org/subversive/">subversive</a>. 
Instalacja tej ostatniej w nowej wersji IDE (Helios) przysparza pewnych kłopotów, ponieważ po instalacji samej wtyczki nie sposób zainstalować dostawcy <em>Connectorów </em>- pojawia się komunikat o błędzie: <em>connectors are not available</em>. 

Rozwiązanie polega na instalacji developerskiej wersji wtyczki (<a href="http://www.eclipse.org/subversive/downloads.php#early_access">early access</a>), zamiast wtyczki przeznaczonej dla Eclipse Helios. Poprawny link dla menedżera wtyczek: <a href="http://download.eclipse.org/technology/subversive/0.7/update-site/">klik</a>.