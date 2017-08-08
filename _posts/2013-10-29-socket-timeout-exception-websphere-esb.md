---
ID: 179
post_title: 'javax.xml.ws.WebServiceException: java.net.SocketTimeoutException: Async operation timed out'
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2013/10/socket-timeout-exception-websphere-esb/
published: true
post_date: 2013-10-29 23:12:44
---
Implementacja integracji długo wykonujących się usług za pomocą narzędzi IBM (Websphere ESB, Integration Designer) wymaga zmiany konfiguracji domyślnych limitów czasu oczekiwania na odpowiedź.

W przypadku usług JAX-WS, w których komunikacja bazuje na portokole HTTP, kontener czeka na wynik usługi nie dłużej niż 300 sekund. Po przekroczeniu tego czasu przepływ mediacji integracyjnej kończy się wyzwoleniem terminalu błędu z komunikatem 
[sourcecode lang="java"]
javax.xml.ws.WebServiceException: java.net.SocketTimeoutException: Async operation timed out.
[/sourcecode]

Warto zwrócić uwagę, że jest to błąd związany z socketami, a nie z wywołaniem usługi, czy transakcjami (konfiguracja przebiega inaczej).

Limit czasu oczekiwania zaszyty jest w stałych polach klas użytkowych serwera aplikacji (<a href="http://pic.dhe.ibm.com/infocenter/wasinfo/v7r0/index.jsp?topic=%2Fcom.ibm.websphere.soafep.multiplatform.doc%2Finfo%2Fae%2Fae%2Frwbs_jaxwstimeouts.html" target="_blank">link</a><wbr />). Szczęśliwie, można się do nich odwołać za pomocą literałów konfigurowalnych z linii poleceń.

Strojenie tych parametrów sprowadza się do dodania właściwości maszyny wirtualnej.

1. Servers / Application Servers / nazwa_serwera / Java and Process Management / Process Definition / Java Virtual Machine / Custom Properties.

2. Dodanie lub edycja właściwości: 
[sourcecode lang="java"]
name=timeout value=600[czas w sekundach]
[/sourcecode]

3. Restart serwera