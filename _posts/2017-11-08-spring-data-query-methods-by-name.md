---
layout: post
title: Spring Data Query Methods by Name
date: '2017-11-08'
summary: "Everything you need to know on Spring Data's strategy to generate queries from method names"
comments: true
---

There are two ways Spring Data repositories use to derive queries from repository's methods. The first is by using a declared query which is manually defined, such as by using `@Query` or `@NamedQuery` annotations. The second one is by deriving queries directly from the method name also called *query creation*.

In this blog post we will focus on the method name derivation approach in order to generate database queries so we have to write no [JPQL](https://docs.oracle.com/html/E13946_01/ejb3_langref.html) (Java Persistence Query Language) nor the worst case which is store-specific [SQL](https://www.w3schools.com/sql/).

All the source code for this demo is available on [GitHub](https://github.com/selzlein/spring-data-query-methods-demo).

## Spring Data

We first create a Spring Boot application. Then to make use of Spring Data, add its dependency followed by a few more that will soon be explained:

{% highlight xml %}
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
  <groupId>org.hibernate</groupId>
  <artifactId>hibernate-java8</artifactId>
</dependency>
<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
  <scope>runtime</scope>
</dependency>
{% endhighlight %}

In this demo we are going to use [H2](http://www.h2database.com/html/main.html) database, therefore its dependency, on in-memory mode so it gets recreated every time we start our application.

We also added Hibernate Java8 to support Java 8 specific features, mainly the new Date/Time api.

## Model

For this demo we will use two simple model classes: [`Customer`](https://github.com/selzlein/spring-data-query-methods-demo/blob/master/src/main/java/com/selzlein/djeison/springdataquerymethodsdemo/model/Address.java) and [`Address`](https://github.com/selzlein/spring-data-query-methods-demo/blob/master/src/main/java/com/selzlein/djeison/springdataquerymethodsdemo/model/Customer.java) where a `Customer` has one `Address`. 

{% highlight java %}
@Getter
@Setter
@Entity
public class Address implements Model {

  private static final long serialVersionUID = 1L;

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String street;

  private String zipCode;

  @OneToMany(mappedBy = "address", fetch = FetchType.LAZY)
  private Set<Customer> customer;

}
{% endhighlight %}

{% highlight java %}
@Setter
@Getter
@Entity
public class Customer implements Model {

  private static final long serialVersionUID = 1L;

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String firstName;

  private String lastName;

  private LocalDate birthday;

  @ManyToOne(fetch = FetchType.LAZY)
  private Address address;

}
{% endhighlight %}

## Repository

As per [Spring Data docs](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.definition), in order to define a repository you must define an interface which extends [`Repository`](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/Repository.html) and type it with your domain class and an identifier type. 

Spring Data provides several interfaces that extend `Repository` and expose auxiliar methods for the domain type you specified, such as: [CRUD](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html) and [paging and sorting](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/PagingAndSortingRepository.html) methods.

As detailed above, when defining a repository we must extend *Repository* interface. In our case, we will extend [*JpaRepository*](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/JpaRepository.html) which provides additional utility methods, like `findAll`.

We then create two repository interfaces: `CustomerRepository` and `AddressRepository`:

{% highlight java %}
@Repository
public interface AddressRepository extends JpaRepository<Address, Long> {

}
{% endhighlight %}

{% highlight java %}
@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {

}
{% endhighlight %}

## Query Creation

The query creation strategy derives a database query from a method name. For us to use this feature we have to follow some rules:

### Prefix

Method names should be prefixed with: `find…By`, `read…By`, `query…By`, `count…By`, and `get…By`.

The prefix `find` can be followed by further expressions such as `Distinct` keyword which sets unique results or result size limiting keywords like: `Top` and `First`. E.g.:

{% highlight java %}
@Repository
public interface AddressRepository extends JpaRepository<Address, Long> {

  Set<Address> findByZipCode(String zipCode);

  Address findFirstByStreet(String street);

  // ...
}
{% endhighlight %}

### Properties Expressions

The first `By` keyword in the method name works as a delimiter that indicates where the criteria starts. In this part we can define conditions on entity properties and concatenate them by using `And` and `Or` keywords.

{% highlight java %}
@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
  // ...

  Set<Customer> findByFirstNameAndLastName(String firstName, String lastName);

  // ...
}
{% endhighlight %}


The number of method parameters must match at least the number of conditions. As we saw in the last example, when there is two conditions there is at least two method parameter to be injected in the database query.

It is also possible to define constraints based on nested properties:

{% highlight java %}
@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
  // ...

  Set<Customer> findByAddress_ZipCode(String zipCode);

  // ...
}
{% endhighlight %}

This will filter based on the `zipCode` attribute which belongs to `address` attribute on `Customer`. The use of `_` is optional but it tells the query builder that there is a separation here. This way the query builder understands that `Address` and `ZipCode` are separate things. It also avoids ambiguity like looking for an `AddressZip` property.

There are several keywords that can be used when constraining an entity's property. In the following examples we see how to use `IgnoreCase` and `Containing` (works as a like function):

{% highlight java %}
@Repository
public interface AddressRepository extends JpaRepository<Address, Long> {
  // ...

  Set<Address> findByStreetIgnoreCaseContaining(String street);
	
  // ...
}
{% endhighlight %}

[Here](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation) you can find the full list of keywords supported in property expressions.

### Limiting results

We can limit the number of elements in the result of a query method by using keywords `First` and `Top`. It is possible to specify the maximum quantity in the result set by appending it to the keyword, like `First5` or `Top10`. E.g.:

{% highlight java %}
@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
  // ...

  Set<Customer> findFirst2ByLastName(String lastName);

  // ...
}
{% endhighlight %}

### OrderBy

It is possible to apply sorting by appending `OrderBy` at the end of the method name followed by which properties we want to compose this sorting and if this sorting is `Asc` or `Desc`. E.g.:

{% highlight java %}
@Repository
public interface AddressRepository extends JpaRepository<Address, Long> {
  // ...

  Optional<Address> findFirstByZipCodeOrStreetAllIgnoreCaseOrderByZipCodeDesc(String zipCode, String street);

}
{% endhighlight %}

### Pagination and Sorting

Query builder also supports receiving as method parameters Spring Data's [Pageable](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Pageable.html) and [Sort](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Sort.html) that will also be applied to the database query.

### Stream results

We can also use Java 8's [Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html) as a result type.

{% highlight java %}
@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
  // ...

  Stream<Customer> readAllByLastNameNotNull();

  // ...
}
{% endhighlight %}

This allows us to iterate and apply aggregate functions to or iterate over the query result:

{% highlight java %}
  // ...
  @Test
  public void shouldStreamAllByLastNameNotNull() {
    try (Stream<Customer> customers = customerRepository.readAllByLastNameNotNull()) {
      customers.forEach(customer -> {
        assertNotNull(customer);
        assertNotNull(customer.getId());
      });
    }
  }
  // ...
{% endhighlight %}

### Fetch

`@EntityGraph` annotation allows us to load attributes. This will mix what is in the model entity with what is specified here.

In the following example we have two very similar methods. The basic difference is that `findByAddress_ZipCode` will not initialize the `Address` attribute in `Customer` model while `findWithAddressByAddress_ZipCode` will. That is so because `Address` is lazy by default as it is specified in `Customer` model and using `@EntityGraph` with `attributePath = "address"` tells Spring Data to load the `address` attribute.

{% highlight java %}
@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
  // ...

  Set<Customer> findByAddress_ZipCode(String zipCode);

  @EntityGraph(attributePaths = "address")
  Set<Customer> findWithAddressByAddress_ZipCode(String zipCode);

}
{% endhighlight %}

Notice also the term `WithAddress` in the method name. It is possible to put anything between the prefix (`find` here) and the keyword `By` in order to avoid duplicate method names. I chose `WithAddress` to make the method name intuitive.

## Wrapping up

You can find complete integration tests for the methods we implemented here in this demos' repo on [GitHub](https://github.com/selzlein/spring-data-query-methods-demo/tree/master/src/test/java/com/selzlein/djeison/springdataquerymethodsdemo/service)

Spring Data Query Methods generated by method names make it simple to build queries providing many features. We do not need to write any JPQL and when short they are highly readable.

On the other hand, they may become hard to read when you mix in too many keywords ending up with a huge method name.

I hope this writing helps and feedback is always welcome! :)

## References

Besides links throughout this writing, here are some articles that were of help when building this guide:

- [Spring Data JPA - Reference Documentation: Defining Query Methods](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.details)
- [Spring Data JPA - Reference Documentation: Query Creation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation)
- [Spring Data JPA Tutorial: Introduction to Query Methods](https://www.petrikainulainen.net/programming/spring-framework/spring-data-jpa-tutorial-introduction-to-query-methods/)
- [Spring Data JPA Tutorial: Creating Database Queries From Method Names](https://www.petrikainulainen.net/programming/spring-framework/spring-data-jpa-tutorial-creating-database-queries-from-method-names/)
- [Unit Testing With JUnit – Part 3 – Hamcrest Matchers](https://springframework.guru/unit-testing-junit-part-3-hamcrest-matchers/)
- [Spring Boot – Spring Data JPA with Hibernate and H2 Web Console](https://memorynotfound.com/spring-boot-spring-data-jpa-hibernate-h2-web-console/)
- [Using H2’s web console](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-sql.html#boot-features-sql-h2-console)
