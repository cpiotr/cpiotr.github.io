---
ID: 51
post_title: Ekstrakcja tabel z dokumentów
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2011/02/ekstrakcja-tabel-z-dokumentow/
published: true
post_date: 2011-02-27 01:09:29
---
Przetwarzanie dokumentów pakietów biurowych MS Office oraz OpenOffice.org może odbywać się przy wykorzystaniu specjalnych bibliotek wspierających każdy z pakietów, odpowiednio: <a href="http://poi.apache.org/">Apache POI</a> oraz <a href="http://odftoolkit.org/">ODF Toolkit</a>. Biblioteki umożliwiają zapis i odczyt dokumentów w formie API dla języka Java.

<strong>Tabele w plikach DOC</strong>
[sourcecode language="java"]
HWPFDocument document = new HWPFDocument(new BufferedInputStream(new FileInputStream(file)));
// Zakresem przetwarzania jest caly dokument
Range range = document.getOverallRange();

// Uroki POI - iterator tabel
TableIterator tableIterator = new TableIterator(range);

// Klasa TableIterator nie zawiera informacji o liczbie tabel, 
// dlatego należy policzyć je 'ręcznie'
int tableCount = 0;

while (tableIterator.hasNext()) {
	tableCount++;
	Table table = tableIterator.next();
	
	// Iteracja wierszy
	for (int rowIndex = 0; rowIndex &lt; table.numRows(); rowIndex++) {
		TableRow row = table.getRow(rowIndex);
		
		// Iteracja komórek
		for (int cellIndex = 0; cellIndex &lt; row.numCells(); cellIndex++) {
			TableCell cell = row.getCell(cellIndex);
			
			// Iteracja paragrafów
			for (int parIndex = 0; parIndex &lt; cell.numParagraphs(); parIndex++) {
				Paragraph paragraph = cell.getParagraph(parIndex);
				
				// Iteracja fragmentów tekstu
				for (int runIndex = 0; runIndex &lt; paragraph.numCharacterRuns(); runIndex++) {
					CharacterRun run = paragraph.getCharacterRun(runIndex);
					
					// Treść tekstu komórki
					run.text();
				}
			}
		}
	}
}
[/sourcecode]

<strong>Tabele w plikach DOCX</strong>
W przypadku dokumentów w standardzie OOXML API wygląda na przyjemniejsze:
[sourcecode language="java"]
XWPFDocument document = new XWPFDocument(new BufferedInputStream(new FileInputStream(file)));

if (document.getTables() != null) {
	// Iteracja tabel w dokumencie
	for (XWPFTable table : document.getTables()) {
		if (table.getRows() != null) {
			// Iteracja wierszy w tabeli
			for (XWPFTableRow row : table.getRows()) {
				if (row.getTableCells() != null) {
					// Iteracja komórek w wierszu
					for (XWPFTableCell cell : row.getTableCells()) {
						// Tekst z komórki
						cell.getText();
					}
				}
			}
		}
	}
}
[/sourcecode]

<strong>Tabele w plikach ODT</strong>
Przetwarzanie dokumentów z pakietu OOo sprowadza się do parsowania XML. Znajomość <a href="http://odftoolkit.org/projects/odfdom/pages/Layers#The_ODF_XML_Layer">struktury dokumentów</a> ułatwia wyodrębnianie kolejno zagnieżdżonych elementów.
[sourcecode language="java"]
OdfDocument odfDocument = OdfDocument.loadDocument(file);
		
// Pobranie zawartosci w postaci drzewa DOM.
OdfFileDom odfDOM = odfDocument.getContentDom();

// Inicjacja XPath.
XPath xpath = odfDocument.getXPath();

// Przeszukiwanie dokumentu pod katem tabel
NodeList tables = (NodeList) xpath.evaluate(&quot;//table:table&quot;, odfDOM, XPathConstants.NODESET);

// Iteracja tabel
for (int i = 0; i &lt; tables.getLength(); i++) {
	TableTableElement table = (TableTableElement) tables.item(i);
	NodeList rows = table.getElementsByTagName(TableTableRowElement.ELEMENT_NAME.getQName());
	
	// Iteracja wierszy
	for (int rowIndex = 0; rowIndex &lt; rows.getLength(); rowIndex++) {
		TableTableRowElement row = (TableTableRowElement) rows.item(rowIndex);
		NodeList cells = row.getElementsByTagName(TableTableCellElement.ELEMENT_NAME.getQName());
		
		// Iteracja komórek
		for (int cellIndex = 0; cellIndex &lt; cells.getLength(); cellIndex++) {
			TableTableCellElement cell = (TableTableCellElement) cells.item(cellIndex);
			// Treść komórki
			cell.getTextContent();
		}
	}
}
[/sourcecode]