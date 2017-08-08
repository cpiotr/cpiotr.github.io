---
ID: 220
post_title: Clean unit tests
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2014/09/clean-unit-tests/
published: true
post_date: 2014-09-19 11:21:33
---
It is now considered common knowledge to focus on behaviour rather than on implementation details while writing code - both production and test one. Well established duet of frameworks (<a href="http://hamcrest.org/JavaHamcrest/" target="_blank">hamcrest</a> and <a href="http://junit.org/" target="_blank">junit</a>) is doing a great job in supporting the idea.
We have <code>org.junit.Assert.assertThat()</code> and <code>org.hamcrest.Matchers.*</code> within reach. However, few issues simply have to be kept in mind to avoid unpleasant suprises and to keep tests clean.

<strong>Get your versions right</strong>
It's not sufficient to rely on a provided Maven archetype - some of them are dependent on already rusty version of libraries. After adding additonal library, ambiguity on classpath might occur. <a href="http://tedvinke.wordpress.com/2013/12/17/mixing-junit-hamcrest-and-mockito-explaining-nosuchmethoderror/" target="_blank">Invalid version of class can then be selected</a>, which results in 
[sourcecode lang="java"]
java.lang.NoSuchMethodError: org/hamcrest/Matcher.describeMismatch(Ljava/lang/Object;Lorg/hamcrest/Description;)V
[/sourcecode]
Either use minimalistic pom descriptor in the first place or remember to verify your dependencies.

<strong>Write custom matchers to cover business logic</strong>
Hamcrest allows us to write succint assertions, instead of using bloated for-loops and multiple <code>assertXXX</code> methods.
It's trivial to replace the following code:
[sourcecode lang="java"]
// When
List&lt;Deal&gt; dealsExecutedToday = dealService.externalDealsExecutedAt(TODAY);

// Then
assertNotNull(dealsExecutedToday);
for (Deal deal : dealsExecutedToday) {
	assertNotNull(deal);
	assertEquals(now, deal.getTimestamp());
	
	// External deals
	assertEquals(2, deal.getTradeType());
	assertEquals(&quot;E&quot;, deal.getSourceIdType());
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
[/sourcecode]

with a bit more(?) readable one:
[sourcecode lang="java"]
// When
List&lt;Deal&gt; dealsExecutedToday = dealService.externalDealsExecutedAt(TODAY);
		
// Then
assertThat(dealsExecutedToday, 
		allOf(
			is(notNullValue()),
			is(not(empty())),
			everyItem(hasProperty(&quot;timestamp&quot;, equalTo(TODAY))),
			
			// External deals
			everyItem(hasProperty(&quot;tradeType&quot;, equalTo(2))),
			everyItem(hasProperty(&quot;sourceIdType&quot;, equalTo(&quot;E&quot;))),
			
			// Stock deals
			not(everyItem(hasProperty(&quot;instrumentType&quot;, isIn(STOCK_INSTRUMENTS))));
		)
);
[/sourcecode]

Still not pretty enough. There are some implementation details hanging around, which we can get rid of.
[sourcecode lang="java"]
// When
List&lt;Deal&gt; dealsExecutedToday = dealService.externalDealsExecutedAt(TODAY);
		
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
[/sourcecode]

Static methods used to get references to our custom matchers fit well in a dedicated utility class.
[sourcecode lang="java"]
class DealMatchers {
	public static Matcher&lt;Deal&gt; isExternalDeal() {
		return new TypeSafeMatcher&lt;Deal&gt;() {
			@Override
			public void describeTo(Description description) {
				description.appendText(&quot;external deal [Deal.TradeType = 2 AND Deal.SourceIdType = E]&quot;);
			}
			
			@Override
			protected boolean matchesSafely(Deal item) {
				return item.tradeType == 2 
						&amp;&amp; item.sourceIdType.equals(&quot;E&quot;),
			}
		};
	}
	
	public static Matcher&lt;Deal&gt; isStock() {
		return new TypeSafeMatcher&lt;Deal&gt;() {
			@Override
			public void describeTo(Description description) {
				description.appendText(&quot;stock deal [Deal.InstrumentType is on the list]&quot;);
			}
			
			@Override
			protected boolean matchesSafely(Deal item) {
				return STOCK_INSTRUMENTS.contains(deal.getIntrumentType());
			}
		};
	}
	
	public static Matcher&lt;Deal&gt; isExecutedAtDate(final Date date) {
		return new TypeSafeMatcher&lt;Deal&gt;() {
			@Override
			public void describeTo(Description description) {
				description.appendText(&quot;executed at&quot;).appendValue(date);
				
			}
	
			@Override
			protected boolean matchesSafely(Deal item) {
				return item.timestamp.equals(date);
			}
		};
	}
}
[/sourcecode]

<strong>Tweak existing matchers, when needed</strong>
The result of calling <code>Matcher.describeTo(...)</code> can be quite verbose, depending on the actual Matcher implementation. In fact it can make output completely unusable. Checking if an element is present in huge collection using <code>Matchers.isIn()</code> is a good example of such situation.
Classes in <code>org.hamcrest.core</code> package are not designed for extension, but there are not marked as final either. Nothing blocks us from extending <code>IsIn</code> matcher class and overriding desired methods:

[sourcecode lang="java"]
private static &lt;T&gt; Matcher&lt;T&gt; isIn(Collection&lt;T&gt; collection) {
	return new IsIn&lt;T&gt;(collection) {
		private static final int MAX_LENGTH = 40;
		
		@Override
		public void describeMismatch(Object item, Description description) {
			String itemAsString = String.valueOf(item);
			description.appendText(&quot;starting with &lt;&quot;)
					.appendText(itemAsString.substring(0, Math.min(MAX_LENGTH, itemAsString.length())))
					.appendText(itemAsString.length() &gt; MAX_LENGTH ? &quot;...&quot; : &quot;&quot;)
					.appendText(&quot;&gt; was not there&quot;);
		}
		
		@Override
		public void describeTo(Description buffer) {
			buffer.appendText(&quot;on an input list&quot;);
		}
	};
}
[/sourcecode]