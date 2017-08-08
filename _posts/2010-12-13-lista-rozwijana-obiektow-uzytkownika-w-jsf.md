---
ID: 42
post_title: >
  Lista rozwijana obiektów użytkownika w
  JSF
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2010/12/lista-rozwijana-obiektow-uzytkownika-w-jsf/
published: true
post_date: 2010-12-13 02:48:23
---
Okazuje się, że wykorzystanie podstawowego komponentu JSF, jakim jest <a href="http://download.oracle.com/javaee/5/javaserverfaces/1.2/docs/tlddocs/h/selectOneMenu.html">lista rozwijana</a>, może sprawić problem.

Komponent <code>selectOneMenu</code> jest wypełniany obiektami klasy <code>SelectItem</code>. Wyświetlenie listy obiektów zdefiniowanej przez nas klasy sprowadza się do napisania odwzorowania kolekcji na listę obiektów <code>SelectItem</code>. Wykonanie tej czynności pozwoli poprawnie wyświetlić komponent, co zamyka drogę serwer -> przeglądarka klienta.

Komunikacja w drugą stronę przebiega następująco. Użytkownik wybiera pozycję z listy rozwijanej i zatwierdza formularz. Do serwera trafia żądanie HTTP z parametrem opisującym wybór użytkownika (w postaci tekstu). Serwer musi uzyskany tekst zamienić na obiekt klasy, którą wypełniona została lista rozwijana. Następnie musi sprawdzić, który obiekt z listy został wybrany.

Pierwsza część pracy serwera realizowana jest poprzez specjalne komponenty JSF zwane konwerterami. Programista musi zaimplementować konwerter dla klasy, którą wypełnił listę rozwijaną (<a href="http://www.ibm.com/developerworks/library/j-jsf3/#N10239">przykład</a>).

Sprawdzenie, który obiekt został wybrany z listy odbywa się poprzez iterację po wszystkich jej elementach i porównaniu poprzez metodę <code>equals()</code> pod kątem obiektu uzyskanego z konwertera. Dlatego ważne jest, aby w utworzonej klasie, której obiektami wypełniamy listę rozwijaną, nadpisać metodę <code>equals()</code>. Niewykonanie tego kroku może skutkować wystąpieniem błędu w fazie walidacji: <code>Validation Error: Value is not valid</code>