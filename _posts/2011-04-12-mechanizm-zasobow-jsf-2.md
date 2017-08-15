---
ID: 57
post_title: Mechanizm zasobów JSF 2
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2011-04-12 10:26:55
---
Struktura aplikacji webowej wykorzystującej JSF 2 zawiera katalog zasobów (resources/) w ramach mechanizmu zarządzania zasobami. Zachowanie trzypoziomowej hierarchii katalogu zapewnia łatwe odwołania do wybranych zasobów poprzez tagi JSF 2.
<ul>
<li>resources/</li>
<ul>
<li>css/</li>
<ul>
<li>style.css</li>
<li>extra.css</li>
</ul>
</ul>
<ul>
<li>img/</li>
<ul>
<li>bg.gif</li>
<li>logo.png</li>
</ul>
</ul>
</ul>

Aby wyświetlić grafikę, należy wykorzystać tag
```
<h:graphicImage library="img" name="logo.png"/>
```

Aby odwołać się do zasobu w arkuszu CSS trzeba wykorzystać mechanizm języka wyrażeń (EL) wspierający dostęp do zasobów. Przykładowo:
```
	body {
		background-image: url('#{resource['img:bg.gif']}');
	}
```

Łatwo zauważyć, że format wyrażenia stosowanego do uzyskania referencji do zasobu jest następujący:
```
	#{resource['LIBRARY:FILENAME']}
```