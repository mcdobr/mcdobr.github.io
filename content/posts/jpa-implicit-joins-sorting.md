---
title: "JPA Sorting & implicit joins"
date: 2022-09-18T14:00:00+03:00
tags: []
featured_image: ""
description: "Path navigability defaults to inner join when sorting in JPA"
draft: false
---

On a GET call for an HTTP API usually clients would need the functionality to filter and sort by certain fields. First, let's suppose you have the following entities: Person, Address and City. Let's assume that one person can have many addresses, and each address can have one city.

Using [Spring Data's PagingAndSortingRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/PagingAndSortingRepository.html) the service layer method would look something like this:

```
@RequiredArgsContructor
public class PersonService {
    @Autowired
    private PersonRepository personRepository;

    public List<PersonDto> findAll() {
        List<Person> persons = personRepository.findAll(Sort.by(Person_.address + "." Adress_.city + "." + City_.name));
        return persons.stream()
            .map(person -> convertToDto(person))
            .collect(Collectors.toList());
    }

    // Handle conversion
    // ...
}
```
If you are wondering where do the Person_, Address_, City_ classes come from, you should look into [JPA Metamodel generator](https://docs.jboss.org/hibernate/orm/5.3/topical/html_single/metamodelgen/MetamodelGenerator.html) which generates these classes for you.


The [JPA specification subchapter for path expressions](https://jakarta.ee/specifications/persistence/3.0/jakarta-persistence-spec-3.0.html#path-expressions) states the following:

```
Path expression navigability is composed using “inner join” semantics. That is, if the value of a non-terminal field in the path expression is null, the path is considered to have no value, and does not participate in the determination of the result.
```

The above call of the findAll function would have the following JPQL:

```
select p
from Person p
left join p.addresses a
left join a.city c
order by p.addresses.city.name
```

The above query would have the result set implicitly filtered by the `order by` because the default is to use implicit joins, ignoring nulls along the "join chain".

The query can be modified to also return the persons which might not have any known address:

```
select p
from Person p
left join p.addresses a
left join a.city c
order by c.name
```

The example given is fairly simple, even though not immediately obvious when debugging. At least it wasn't to me. However, it is something to take into account especially if you allow clients to specify the sort criteria (sort order(s) and sort field(s)) and you are dynamically creating the queries with something like [Spring's Sort util](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Sort.html) by specifying a path chain.