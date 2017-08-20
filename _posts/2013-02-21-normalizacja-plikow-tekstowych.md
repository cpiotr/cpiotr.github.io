---
ID: 150
post_title: Normalizacja plików tekstowych
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2013-02-21 23:15:39
---
Zadanie normalizacji zawartości plików tekstowych przestaje być łatwe, kiedy brakuje informacji o źródłowym kodowaniu. Bez uprzedniej wiedzy o stronie kodowej danego zestawu znaków konwersja do UTF-8 nie ma szans na powodzenie.
Stosunkowo łatwe jest dokonanie rozróżnienia między kodowaniem pliku ASCII a UTF-8. Jednak rozpoznanie rozszerzeń ASCII (np. Windows-1250, ISO-8859-2) wymaga specjalnych metod (<em><a href="http://web.mit.edu/ghudson/old/dev/nokrb/third/firefox/extensions/universalchardet/doc/UniversalCharsetDetection.doc" target="_blank">S. Li, K. Momoi, "A composite approach to language/encoding detection"</a></em>). Dla zapewnienia pomyślności procesu detekcji wymagane jest uprzednie utworzenie zbioru struktur danych odwzorowujących odpowiednie strony kodowe z możliwością dokonania iteracji po wszystkich znakach danej strony. Ciąg znaków będący zestawem danych wejściowych zostaje poddany szeregowi porównań bazujących na modelach stron kodowych. Jako rezultat typowane jest kodowanie <em>najbardziej zbliżone statystycznie </em>do wprowadzonego tekstu.

Przykładem biblioteki realizującej zaprezentowane podejście detekcji kodowania jest <a href="http://site.icu-project.org/" target="_blank">ICU</a>. Poniżej fragment kodu wykrywający kodowanie w strumieniu danych z wykorzystaniem `icu4j`.

```
/** 
  * Prog akceptacji wykrytego kodowania, wyrazony w procentach. 
  * Wyznaczony empirycznie.
  */
private static final int THRESHOLD = 51;

private Charset guessCharset(InputStream is) {
	Charset detectedCharset = null;
	CharsetMatch match = null;
	
	try {
		// Odczyt danych
		CharsetDetector detector = new CharsetDetector();
		detector.setText(is);
		
		// Wywolanie detekcji
		match = detector.detect();
		
		// Czy kodowanie zostalo wykryte z prawdopodobienstwem 
		// przekraczajacym ustalony prog 
		if (match != null && match.getConfidence() > THRESHOLD) {
			detectedCharset = Charset.forName(match.getName());
		}
	} catch (IOException ioe) {
		LOG.error("Problem with InputStream", ioe);
	} catch (IllegalCharsetNameException icne) {
		LOG.warn("Charset not found: " + match.getName());
	} catch (UnsupportedCharsetException uce) {
		LOG.warn("Charset not supported: " + match.getName());
	}
	
	return detectedCharset;
}
```

Powszechna wiedza karze pamiętać, że statystyka nie działa dla małych zbiorów danych, dlatego wykrywanie należy przeprowadzać na odpowiednio dużych plikach (zawierających przynajmniej kilka zdań). Jako ciekawostkę dodam, że pojedyncze wywołanie detekcji z biblioteki `ICU` bada maksymalnie 8000 bajtów.