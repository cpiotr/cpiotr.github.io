---
ID: 109
post_title: persist vs merge
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2013-01-10 21:04:10
---
Podczas rozmowy o <a href="http://stackoverflow.com/questions/156689/do-you-have-a-common-base-class-for-hibernate-entities">uniwersalnej metodzie zapisu encji do bazy danych</a> spotkałem się z poglądem, że metody `persist()` i `merge()` interfejsu `EntityManager` pod względem utrwalania nowych encji działają tak samo. Nie <em>prawie tak samo</em>, nie <em>w zasadzie tak samo</em>, ale tak samo.
O ile dopuszczam możliwość, że rozmówca zastosował mało elegancki skrót myślowy, o tyle uważam, że warto przytoczyć podstawowe różnice pomiędzy rzeczonymi.

Metoda `persist` włącza obiekt encyjny podawany jako parametr do kontekstu mechanizmu trwałości (`PersistenceContext`), obiekt staje się zatem <em>zarządzany</em>. W przypadku metody `merge`, obiekt podawany w postaci parametru nie staje się zarządzany, a jedynie jako rezultat metody zwracana jest zarządzana kopia stanu obiektu.
```
BaseEntity entity = new BaseEntity(id, name);
entityManager.merge(entity);
boolean managed = entityManager.contains(entity);
// W tym miejscu zmienna managed ma wartosc FALSE.
```

```
BaseEntity entity = new BaseEntity(id, name);
entityManager.persist(entity);
boolean managed = entityManager.contains(entity);
// W tym miejscu zmienna managed ma wartosc TRUE.
```

Abstrahując od dodawania nowych obiektów do bazy danych, obie rozpatrywane metody różni na przykład sposób traktowania encji usuniętych(`persist` na nowo włącza encję do puli zarządzanych obiektów w kontekście mechanizmu trwałości, natomiast `merge` powoduje rzucenie wyjątku) oraz encji odłączonych od kontekstu (tym razem wywołanie `persist` zakończy się rzuceniem przez kontener wyjątku, podczas gdy `merge` spowoduje zapis do bazy danych).

Temat szerzej poruszany jest <a href="http://spitballer.blogspot.com/2010/04/jpa-persisting-vs-merging-entites.html">tu</a> i <a href="http://blog.xebia.com/2009/03/23/jpa-implementation-patterns-saving-detached-entities/">tu</a>.