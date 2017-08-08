---
ID: 65
post_title: >
  Brak odpowiedzi Eclipse przy edycji
  stron z faceletami
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2011/04/brak-odpowiedzi-eclipse-przy-edycji-stron-z-faceletami/
published: true
post_date: 2011-04-19 00:34:16
---
Eclipse w wersji Helios uruchomiony na systemie Windows 7 ma problemy z walidacją stron JSP z faceletami. Niefortunne zamknięcie IDE z aktywną zakładką prezentującą taką właśnie stronę może spowodować komplikacje przy ponownym uruchomieniu. W moim przypadku po wybraniu właściwej przestrzeni pracy (workspace) Eclipse ładował wszystkie elementy po czym przestawał odpowiadać. Problem nie występował w przypadku innych workspace'ów.
Aby móc z powrotem pracować, zmieniłem ustawienia edytora zapisane w pliku:
[sourcecode language="bash"]
{WORKSPACE_PATH}/.metadata/.plugins/org.eclipse.ui.workbench/workbench.xml
[/sourcecode]

W sekcji:
[sourcecode language="xml"]
&lt;editor activePart=&quot;true&quot; ... /&gt;
[/sourcecode]
należy zmienić identyfikator na Edytor Jednostki Kompilacji:
[sourcecode language="xml"]
id=&quot;org.eclipse.jdt.ui.CompilationUnitEditor&quot;
[/sourcecode]
nastomiast wszystkie atrybuty wskazujące plik JSP zastąpić dowolnym plikiem klasy lub ścieżką do niego.
Konieczna jest również zmiana ścieżki przypisanej do znacznika <code>&lt;input&gt;</code> położonego bezpośrednio za edytowanym znacznikiem <code>&lt;editor&gt;</code>.

Warto rozważyć zmianę częstości walidacji stron JSP z faceletami (Window->Preferences->Validation)

Przykład poprawnie zmienionych znaczników wygląda następująco:
[sourcecode language="xml"]
&lt;editor id=&quot;org.eclipse.jdt.ui.CompilationUnitEditor&quot; name=&quot;MyClass.java&quot; partName=&quot;MyClass.java&quot; path=&quot;E:/workspace/Portal/src/pl/ciruk/portal/web/bean/MyClass.java&quot; title=&quot;MyClass.java&quot; tooltip=&quot;Portal/src/pl/ciruk/portal/web/bean/MyClass.java&quot; workbook=&quot;DefaultEditorWorkbook&quot;&gt;
&lt;input factoryID=&quot;org.eclipse.ui.part.FileEditorInputFactory&quot; path=&quot;/Portal/src/pl/ciruk/portal/web/bean/MyClass.java&quot;/&gt;
[/sourcecode]