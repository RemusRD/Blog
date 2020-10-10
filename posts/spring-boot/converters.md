# Spring boot: A guide about ConversionService and type converters

## Introduction

To develop a service where the business logic is decoupled from the adapters that connect our domain with the outside world, we need some kind of conversion between messages and domain objects making it easier to replace the implementation.

In this blog post, I am going to show you the different ways Spring boot provides us to do it cleanly and elegantly and how it affects every stage of the development.

## The Converter<S, T> interface

the converter interface is pretty straight forward, it only contains a method that takes the input object *S* and returns the output object *T.* 

I personally like this interface because it forces you to only convert one type to another in each converter you develop, I have seen classes that are pretty large and convert a lot of types making them pretty hard to maintain.

Here, the S and the T are pretty similar, but for example, you would not want to have the Jackson annotations to be in the domain object to decouple the consumer from the domain itself, usually, you would never serialize directly the domain object to any adapter(an exception comes to my mind and I have seen it a lot and is when you use the same object for the domain and the DB).

```java
@Component
public class EmployeeToEmployeeResponseConverter implements Converter<Employee, EmployeeResponse> {
    @Override
    public EmployeeResponse convert(Employee employee) {
        final var subordinates =
                employee.getSubordinates()
                        .stream().map(this::convert)
                        .collect(Collectors.toUnmodifiableList());

        return new EmployeeResponse(employee.getName(), subordinates);
    }
}
```

In this case, we have here a simple converter that converts an Employee to some kind of response. Note that the converter is marked as a component, we will need that later. 

```java
public class Employee {
    private final String name;
    private final List<Employee> subordinates;
		//getters, constructor, equals...
}
```

An employee is just a data class that contains the name of the employee and a list of subordinates(other employees).

## Adding our custom converters to the ConversionService bean

We have declared our custom converter as a Spring bean but in order to be able to use it in the conversion service, we should configure the conversion service as follows:

```java
@Configuration
public class ConversionConfiguration {
    @Bean
    public ConversionService conversionService(List<Converter> converters) {
        final DefaultConversionService conversionService = new DefaultConversionService();
        converters.forEach(conversionService::addConverter);
        return conversionService;
    }
}
```

This configuration would override the default conversion service, that already contains some converters with our custom implementation that would add our custom converters exposed as components by default.

## Composing converters

Sometimes, it makes sense to have a bigger converter that needs to convert smaller entities itself, to not duplicate the code of the smaller entities across multiple converters, we could create a converter for each entity.

There are two ways to compose those converters:

### Injecting the converter itself

In this approach, we inject the converter by name, I like this one better because you can use the actual converter in the unit test making it a 'social test', however, mocking the response from the conversion service is an approach that I have personally used too.

```java
@Component
public class HierarchyToHierarchyResponseConverter implements Converter<Hierarchy, HierarchyResponse> {

    private final EmployeeToEmployeeResponseConverter responseConverter;

    public HierarchyToHierarchyResponseConverter(EmployeeToEmployeeResponseConverter responseConverter) {
        this.responseConverter = responseConverter;
    }

    @Override
    public HierarchyResponse convert(Hierarchy hierarchy) {
        return new HierarchyResponse(responseConverter.convert(hierarchy.getSupervisor()));
    }
}
```

### Injecting the ConversionService

Using the conversion service, you lose the type safety of the conversion and you could fail in runtime if you did not register the converter properly(you could fix this by doing an integration/acceptance test), but you also decouple them. I think is like the [inside out vs outside-in approach in testing](https://8thlight.com/blog/georgina-mcfadyen/2016/06/27/inside-out-tdd-vs-outside-in.html) so do whatever feels good for you.

```java
@Component
public class HierarchyToHierarchyResponseConverter implements Converter<Hierarchy, HierarchyResponse> {

    private final ConversionService conversionService;

    public HierarchyToHierarchyResponseConverter(ConversionService conversionService) {
        this.conversionService = conversionService;
    }

    @Override
    public HierarchyResponse convert(Hierarchy hierarchy) {
        return new HierarchyResponse(conversionService.convert(hierarchy.getSupervisor(), EmployeeResponse.class));
    }
}
```

Make sure if you use the conversionService bean to validate it using any kind of integration test where you make sure all beans are published properly into the spring context.

## Testing our custom converters

### Setting up the test

Depending on the approach you decided in the composition of converters, you would mock the response of the converters you depend on like so:

```java
@BeforeEach
    void setUp() {
        final var conversionService = mock(ConversionService.class);
        sut = new HierarchyToHierarchyResponseConverter(conversionService);
    }
```

Then, you could either, mock the conversion to point the real converter, or return a mock object.

If you decide to not use the conversionService inside, you would instantiate the real converter when creating your subject under test, however, keep in mind if you nest a lot of converters, it would be harder to maintain.

```java
@BeforeEach
void setUp() {
    sut = new HierarchyToHierarchyResponseConverter(new EmployeeToEmployeeResponseConverter());
}
```

### Asserting the result of the test

One tip I got from a colleague(thank you Sergio!) that I got when asserting the results of the converters, is that you should not assert against the expected object, instead, validate each field with the expected value to have a robust test.

```java
@Test
    void shouldConvert() {
        final Hierarchy simpleHierarchy = EmployeeFactory.simpleHierarchy();

        final EmployeeResponse result = sut.convert(simpleHierarchy).getSupervisor();

        assertThat(result.getName()).isEqualTo("Jonas");
        final EmployeeResponse sophie = result.getSubordinates().get(0);
        assertThat(sophie.getName()).isEqualTo("Sophie");
    } 
```

In this case, my factory is returning me a hierarchy where the employees look like Jonas → Sophie, so instead of creating a new Object with the whole hierarchy, I would test only the needed ones, as you are not forced to implement a proper equals method.

## Injecting our converters inside the adapters

if we added our custom converters to the conversionService bean we just would need to inject the modified bean into our adapter like so:

```java
public HierarchyController(HierarchyCommandService commandService, HierarchyQueryService queryService, ConversionService conversionService) {
        this.commandService = commandService;
        this.queryService = queryService;
        this.conversionService = conversionService;
    }

    @PostMapping("/employees/hierarchy")
    public ResponseEntity<HierarchyResponse> createHierarchy(@RequestBody Map<String, String> employees) {
        final Hierarchy hierarchy = commandService.retrieveEmployeeHierarchy(employees);
        final HierarchyResponse hierarchyResponse = conversionService.convert(hierarchy, HierarchyResponse.class);
        return ResponseEntity.ok(hierarchyResponse);
    }
```

## Conclusion

Spring boot provides several ways to perform type conversion helping you to keep your code decoupled and making testing easier by testing smaller pieces.
