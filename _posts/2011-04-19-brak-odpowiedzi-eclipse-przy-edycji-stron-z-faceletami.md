---
ID: 65
post_title: >
  Brak odpowiedzi Eclipse przy edycji
  stron z faceletami
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2011-04-19 00:34:16
---
Eclipse w wersji Helios uruchomiony na systemie Windows 7 ma problemy z walidacją stron JSP z faceletami. Niefortunne zamknięcie IDE z aktywną zakładką prezentującą taką właśnie stronę może spowodować komplikacje przy ponownym uruchomieniu. W moim przypadku po wybraniu właściwej przestrzeni pracy (workspace) Eclipse ładował wszystkie elementy po czym przestawał odpowiadać. Problem nie występował w przypadku innych workspace'ów.
Aby móc z powrotem pracować, zmieniłem ustawienia edytora zapisane w pliku:
```
{WORKSPACE_PATH}/.metadata/.plugins/org.eclipse.ui.workbench/workbench.xml
```

W sekcji:
```
<editor activePart="true" ... />
```
należy zmienić identyfikator na Edytor Jednostki Kompilacji:
```
id="org.eclipse.jdt.ui.CompilationUnitEditor"
```
nastomiast wszystkie atrybuty wskazujące plik JSP zastąpić dowolnym plikiem klasy lub ścieżką do niego.
Konieczna jest również zmiana ścieżki przypisanej do znacznika `<input>` położonego bezpośrednio za edytowanym znacznikiem `<editor>`.

Warto rozważyć zmianę częstości walidacji stron JSP z faceletami (Window->Preferences->Validation)

Przykład poprawnie zmienionych znaczników wygląda następująco:
```
<editor id="org.eclipse.jdt.ui.CompilationUnitEditor" name="MyClass.java" partName="MyClass.java" path="E:/workspace/Portal/src/pl/ciruk/portal/web/bean/MyClass.java" title="MyClass.java" tooltip="Portal/src/pl/ciruk/portal/web/bean/MyClass.java" workbook="DefaultEditorWorkbook">
<input factoryID="org.eclipse.ui.part.FileEditorInputFactory" path="/Portal/src/pl/ciruk/portal/web/bean/MyClass.java"/>
```