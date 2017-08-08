---
ID: 83
post_title: Bean Validation (JSR 303)
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2012/07/bean-validation-jsr-303/
published: true
post_date: 2012-07-12 21:13:41
---
<a href="http://docs.oracle.com/javaee/6/tutorial/doc/gircz.html" title="Bean Validation" target="_blank">Bean Validation</a> (JSR 303) to świetny sposób na zapewnienie mechanizmu sprawdzania popraności danych. Nowoczesny, intuicyjny i uwzględniony w standardzie JEE6. 
Oprócz standardowo dostępnych walidatorów istnieje możliwość rozszerzenia funkcjonalności poprzez stworzenie własnych adnotacji i odpowiadającej im implementacji (<a href="http://docs.jboss.org/hibernate/validator/4.0.1/reference/en/html/validator-customconstraints.html" title="przykład implementacji" target="_blank">przykład implementacji</a>).

Załóżmy, że napisaliśmy już odpowiednią adnotację:
[sourcecode language="java"]
@Target( { METHOD, FIELD, ANNOTATION_TYPE })
@Retention(RUNTIME)
@Constraint(validatedBy = ValidDataValidator.class)
@Documented
public @interface ValidData {

    String message() default &quot;{pl.ciruk.blog.constraints.validdata}&quot;;

    Class&lt;?&gt;[] groups() default {};

    Class&lt;? extends Payload&gt;[] payload() default {};
    
    String propertyName();

}
[/sourcecode]

I odpowiadającą jej implementację klasy walidującej:
[sourcecode language="java"]
public class ValidDataValidator implements ConstraintValidator&lt;ValidData, MyEntity&gt; {

    private ValidData annotation;

    public void initialize(ValidData annotation) {
        this.annotation = annotation;
    }

    public boolean isValid(MyEntity obj, ConstraintValidatorContext ctx) {
		// Zalozmy, ze obiekt nie moze byc null
		if (obj == null) {
			// Nie przekazujemy domyslnego komunikatu o bledzie
			ctx.disableDefaultConstraintViolation() ;
			
			ctx.buildConstraintViolationWithTemplate(&quot;Podana encja jest rowna NULL&quot;).addNode(annotation.propertyName()).	addConstraintViolation(); 
		}
    }
}
[/sourcecode]

Prosta klasa z polem podlegającym walidacji:
[sourcecode language="java"]
public class TestPojo {
	@ValidData(propertyName=&quot;myEntity&quot;)
	private MyEntity entity;
}
[/sourcecode]

Następnie wywołujemy mechaznim walidacji:
[sourcecode language="java"]
public class TestLogic {
	public void validate(TestPojo pojo) {
		ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        validator = factory.getValidator();
		
		Set&lt;ConstraintViolation&lt;TestPojo&gt;&gt; violations = validator.validate(pojo);
		
		// Wykryto bledy walidacji
		if (!violations.isEmpty()) {
			for (ConstraintViolation&lt;TestPojo&gt; violation : violations) {
				// Chcemy uzyskac informacje o polu, ktore nie przeszlo walidacji (wartosc atrybutu 'propertyName')
				String path = violation.getPropertyPath().toString();
				...
				
				// Nalezy natomiast unikac ponizszego podejscia:
				while(violation.getPropertyPath().iterator.hasNext()) {
					String node = violation.getPropertyPath().iterator.next().toString();
					
					// Petla sie nie konczy
				}
			}
		}
	}
}
[/sourcecode]

Lekko mylący jest fakt, że <code>ConstraintViolation.getPropertyPath()</code> implementuje interfejs <code>Iterable</code>, a próba iteracji może zakończyć się nieskończoną pętlą (dla sytuacji jak na przykładzie).