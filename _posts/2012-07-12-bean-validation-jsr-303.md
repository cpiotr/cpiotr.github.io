---
ID: 83
post_title: Bean Validation (JSR 303)
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2012-07-12 21:13:41
---
<a href="http://docs.oracle.com/javaee/6/tutorial/doc/gircz.html" title="Bean Validation" target="_blank">Bean Validation</a> (JSR 303) to świetny sposób na zapewnienie mechanizmu sprawdzania popraności danych. Nowoczesny, intuicyjny i uwzględniony w standardzie JEE6. 
Oprócz standardowo dostępnych walidatorów istnieje możliwość rozszerzenia funkcjonalności poprzez stworzenie własnych adnotacji i odpowiadającej im implementacji (<a href="http://docs.jboss.org/hibernate/validator/4.0.1/reference/en/html/validator-customconstraints.html" title="przykład implementacji" target="_blank">przykład implementacji</a>).

Załóżmy, że napisaliśmy już odpowiednią adnotację:
```
@Target( { METHOD, FIELD, ANNOTATION_TYPE })
@Retention(RUNTIME)
@Constraint(validatedBy = ValidDataValidator.class)
@Documented
public @interface ValidData {

    String message() default "{pl.ciruk.blog.constraints.validdata}";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
    
    String propertyName();

}
```

I odpowiadającą jej implementację klasy walidującej:
```
public class ValidDataValidator implements ConstraintValidator<ValidData, MyEntity> {

    private ValidData annotation;

    public void initialize(ValidData annotation) {
        this.annotation = annotation;
    }

    public boolean isValid(MyEntity obj, ConstraintValidatorContext ctx) {
		// Zalozmy, ze obiekt nie moze byc null
		if (obj == null) {
			// Nie przekazujemy domyslnego komunikatu o bledzie
			ctx.disableDefaultConstraintViolation() ;
			
			ctx.buildConstraintViolationWithTemplate("Podana encja jest rowna NULL").addNode(annotation.propertyName()).	addConstraintViolation();
		}
    }
}
```

Prosta klasa z polem podlegającym walidacji:
```
public class TestPojo {
	@ValidData(propertyName="myEntity")
	private MyEntity entity;
}
```

Następnie wywołujemy mechaznim walidacji:
```
public class TestLogic {
	public void validate(TestPojo pojo) {
		ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        validator = factory.getValidator();
		
		Set<ConstraintViolation<TestPojo>> violations = validator.validate(pojo);
		
		// Wykryto bledy walidacji
		if (!violations.isEmpty()) {
			for (ConstraintViolation<TestPojo> violation : violations) {
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
```

Lekko mylący jest fakt, że `ConstraintViolation.getPropertyPath()` implementuje interfejs `Iterable`, a próba iteracji może zakończyć się nieskończoną pętlą (dla sytuacji jak na przykładzie).