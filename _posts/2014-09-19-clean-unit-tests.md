---
ID: 220
post_title: Clean unit tests
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2014-09-19 11:21:33
---
It is now considered common knowledge to focus on behaviour rather than on implementation details while writing code - both production and test one. Well established duet of frameworks (<a href="http://hamcrest.org/JavaHamcrest/" target="_blank">hamcrest</a> and <a href="http://junit.org/" target="_blank">junit</a>) is doing a great job in supporting the idea.
We have `org.junit.Assert.assertThat()` and `org.hamcrest.Matchers.*` within reach. However, few issues simply have to be kept in mind to avoid unpleasant suprises and to keep tests clean.

<strong>Get your versions right</strong>
It's not sufficient to rely on a provided Maven archetype - some of them are dependent on already rusty version of libraries. After adding additonal library, ambiguity on classpath might occur. <a href="http://tedvinke.wordpress.com/2013/12/17/mixing-junit-hamcrest-and-mockito-explaining-nosuchmethoderror/" target="_blank">Invalid version of class can then be selected</a>, which results in 
```
java.lang.NoSuchMethodError: org/hamcrest/Matcher.describeMismatch(Ljava/lang/Object;Lorg/hamcrest/Description;)V
```
Either use minimalistic pom descriptor in the first place or remember to verify your dependencies.

<strong>Write custom matchers to cover business logic</strong>
Hamcrest allows us to write succint assertions, instead of using bloated for-loops and multiple `assertXXX` methods.
It's trivial to replace the following code:
```
// When
List<Deal> dealsExecutedToday = dealService.externalDealsExecutedAt(TODAY);

// Then
assertNotNull(dealsExecutedToday);
for (Deal deal : dealsExecutedToday) {
	assertNotNull(deal);
	assertEquals(now, deal.getTimestamp());
	
	// External deals
	assertEquals(2, deal.getTradeType());
	assertEquals("E", deal.getSourceIdType());
}
// Not all deals are stock deals
boolean allStock = true;
for (Deal deal : dealsExecutedToday) {
	if (!STOCK_INSTRUMENTS.contains(deal.getIntrumentType())) {
		allStock = false;
		break;
	}
}
assertFalse(allStock);
```

with a bit more(?) readable one:
```
// When
List<Deal> dealsExecutedToday = dealService.externalDealsExecutedAt(TODAY);
		
// Then
assertThat(dealsExecutedToday, 
		allOf(
			is(notNullValue()),
			is(not(empty())),
			everyItem(hasProperty("timestamp", equalTo(TODAY))),
			
			// External deals
			everyItem(hasProperty("tradeType", equalTo(2))),
			everyItem(hasProperty("sourceIdType", equalTo("E"))),
			
			// Stock deals
			not(everyItem(hasProperty("instrumentType", isIn(STOCK_INSTRUMENTS))));
		)
);
```

Still not pretty enough. There are some implementation details hanging around, which we can get rid of.
```
// When
List<Deal> dealsExecutedToday = dealService.externalDealsExecutedAt(TODAY);
		
// Then
assertThat(dealsExecutedToday, 
		allOf(
			is(notNullValue()),
			is(not(collectionEmpty())),
			everyItem(isExecutedAtDate(TODAY)),
			everyItem(isExternalDeal()),
			not(everyItem(isStock()))
		)
);
```

Static methods used to get references to our custom matchers fit well in a dedicated utility class.
```
class DealMatchers {
	public static Matcher<Deal> isExternalDeal() {
		return new TypeSafeMatcher<Deal>() {
			@Override
			public void describeTo(Description description) {
				description.appendText("external deal [Deal.TradeType = 2 AND Deal.SourceIdType = E]");
			}
			
			@Override
			protected boolean matchesSafely(Deal item) {
				return item.tradeType == 2 
						&& item.sourceIdType.equals("E"),
			}
		};
	}
	
	public static Matcher<Deal> isStock() {
		return new TypeSafeMatcher<Deal>() {
			@Override
			public void describeTo(Description description) {
				description.appendText("stock deal [Deal.InstrumentType is on the list]");
			}
			
			@Override
			protected boolean matchesSafely(Deal item) {
				return STOCK_INSTRUMENTS.contains(deal.getIntrumentType());
			}
		};
	}
	
	public static Matcher<Deal> isExecutedAtDate(final Date date) {
		return new TypeSafeMatcher<Deal>() {
			@Override
			public void describeTo(Description description) {
				description.appendText("executed at").appendValue(date);
				
			}
	
			@Override
			protected boolean matchesSafely(Deal item) {
				return item.timestamp.equals(date);
			}
		};
	}
}
```

<strong>Tweak existing matchers, when needed</strong>
The result of calling `Matcher.describeTo(...)` can be quite verbose, depending on the actual Matcher implementation. In fact it can make output completely unusable. Checking if an element is present in huge collection using `Matchers.isIn()` is a good example of such situation.
Classes in `org.hamcrest.core` package are not designed for extension, but there are not marked as final either. Nothing blocks us from extending `IsIn` matcher class and overriding desired methods:

```
private static <T> Matcher<T> isIn(Collection<T> collection) {
	return new IsIn<T>(collection) {
		private static final int MAX_LENGTH = 40;
		
		@Override
		public void describeMismatch(Object item, Description description) {
			String itemAsString = String.valueOf(item);
			description.appendText("starting with <")
					.appendText(itemAsString.substring(0, Math.min(MAX_LENGTH, itemAsString.length())))
					.appendText(itemAsString.length() > MAX_LENGTH ? "..." : "")
					.appendText("> was not there");
		}
		
		@Override
		public void describeTo(Description buffer) {
			buffer.appendText("on an input list");
		}
	};
}
```