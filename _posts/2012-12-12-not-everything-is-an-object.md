---
ID: 104
post_title: (Not) Everything is an object
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2012-12-12 21:55:41
---
Podczas trawersowania drzewa hierarchii dziedziczenia klas w Javie natknąłem się na pewną interesującą właściwość.
Jak wiadomo, w Javie <a href="http://www.javaworld.com/javaworld/jw-09-2000/jw-0908-eckelobjects.html" title="wszystko jest obiektem" target="_blank">wszystko jest obiektem</a>. Dlatego też wydawać by się mogło, że aby dotrzeć w drzewie hierarchii dziedziczenia od liści do korzenia <a href="http://stackoverflow.com/a/5679311" title="należy" target="_blank">należy</a> przechodzić do rodzica klasy dopóty, dopóki klasą nie jest `java.lang.Object`.
```
public static String getTopClass(Object obj) {
	if obj == null) {
		throw new IllegalArgumentException();
	}
	
	Class<?> clazz = obj.getClass();
	while (!Object.class.equals(clazz)) {
		clazz = clazz.getSuperclass();
	}
	return clazz.getSimpleName();
}
```

Podejście to sprawdza się w przypadku, gdy wejściowym parametrem jest instancja klasy. Co jednak w sytuacji, gdy nie mamy dostępu do obiektu, a jedynie do jego klasy? Opisana sytuacja ma szanse wystąpić w aplikacji wykorzystującej <a href="http://docs.oracle.com/javase/tutorial/reflect/index.html" title="mechanizm refleksji" target="_blank">mechanizm refleksji</a>. Ponieważ preferowanym podejściem w Javie jest tzw. <a href="http://stackoverflow.com/questions/147468/why-should-the-interface-for-a-java-class-be-prefered" target="_blank">implementacja do interfejsu</a>, może się zdarzyć, że na wejściu metody trawersującej graf hierarchii klas pojawi się właśnie interfejs. Klasa `Object` nie jest nadrzędna do interfejsu i pętla z powyższego fragmentu kodu spowodowałaby wystąpienie wyjątku podczas działania programu.
Możliwych rozwiązań jest z całą pewnością bez liku, toteż poniżej pozwolę sobie zamieścić bodaj najprostsze:
```
public static String getTopClass(Class<?> clazz) {
	if (clazz == null) {
		throw new IllegalArgumentException();
	}
	
	if (clazz.getSuperclass() == null) {
		return clazz.getName();
	}
	
	return getTopClass(clazz.getSuperclass());
}
```