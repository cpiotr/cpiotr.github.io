---
ID: 80
post_title: >
  Wykorzystanie opcjonalnego edytora do
  edycji CRONa
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2011/11/wykorzystanie-opcjonalnego-edytora-do-edycji-crona/
published: true
post_date: 2011-11-21 20:47:26
---
Mając ograniczony dostęp do konta shella, edycja tabeli crona odbywa się w domyślnym edytorze systemowym. Niestety istnieje szansa, że nie będzie to edytor spełniający nasze wymagania. Można zmienić preferowany program do edycji na czas trwania sesji poprzez wywołanie komendy:
[sourcecode language="bash"]
env EDITOR=vim crontab -e
[/sourcecode]