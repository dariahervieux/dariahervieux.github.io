---
title: "Error handling: choosing the right place."
categories:
  - Back-end
  - Software design
tags:
  - sw-design
---

> **TL;DR** Choosing the right place and the right approach to handle an error is crucial when it comes to detecting the error and providing a consistent behavior for business logic. 
> The key is to thoroughly analyse the use case and the desired behavior and to determine the level of responsibility of each treatment, participating in the implementation of the use-case, for resolving the potential error.
> It's the use case and business rules which dictate which treatments have low responsibility or hight responsibility. 
> Low responsibility means propagation or generation of an error to pass it over to the caller.
High responsibility means transforming an error into a meaningful result.

**Table of contents**
- [The Problem](#the-problem)
- [Choosing the right place](#choosing-the-right-place)
  - [Examples](#examples)
    - [Example 1: handling the error -  service returning a result for search query](#example-1-handling-the-error---service-returning-a-result-for-search-query)
    - [Example 2: propagating the error -  web service to consult the details of a book](#example-2-propagating-the-error---web-service-to-consult-the-details-of-a-book)
- [Takeaways](#takeaways)
- [Resources](#resources)
- [Credits](#credits)



# The Problem

Have you ever seen the code like this?

```java
public Result doSomething(String mandatoryParameter1, Input someMandatoryInput) {
  if (mandatoryParameter1 == null || someMandatoryInput == null) {
    return null;
  }
  // some business logic...
}
```

Or like this?

```java
  public class ExternalService {
    public void doSomethingForMe() throws ApplicationException {
      //...
      if (somethingBadHappened) {
        throw new ApplicationException("I'd better let you know that I can't handle it!");
      }
    }
  }

  public class UsingExternalService {
    //...
    public List<Result> calculateSomething() {
      ...
      try {
        externalService.doSomethingForMe();
      } catch(ApplicationException e){
        log.warn("Should never happen, ignoring exception.", e);
        return Collections.<Result>emptyList();
      }
    }
  }
```

Don't you think that it looks and feels so wrong! Why? because in both cases the service which "gracefully handles" an error stoles the privilege of handling the error from the caller, which could take care of the error in its specific context. Now the caller is in a situation where it doesn't know that there was an error in the first place and it has a response which finally doesn't worth much, since it's absolutely undistinguishable from the default 'no result'. Really confusing.

So what it is all about?

It's about choosing the right place for taking the responsibility for error handling.

# Choosing the right place

Choosing the right place can be almost straightforward. With a bit of meditation :).

Each use case implementation is a chain of calls to different services. Each high-level service call represents a stack of calls to different application layers. An error can occur at any level: it can be a business error (business constraints are not satisfied), a technical error (NullPointerException), a third party library exception. 
When a starting point of a potential error  is identified, we need to walk through the call stack bottom up from the identified point and up to the highest level, and identify the responsibility of each treatment in the stack for handling the error:

* low responsibility - the current treatment can do nothing *meaningful* with the error, other than propagating as is it of transforming it into another error
* hight responsibility - the error can be transformed into a result, which is meaningful for the caller.

{% include figure image_path="asset/images/articles/Errors/error-bubbling.svg" alt="Error bubbling." caption="Error bubbling." %}

Here are questions which help determine the level of responsibility:

1. Do business rules or use-case requirements allow the current treatment (the callee) handle the error so that the result is *still meaningful* for the caller?
2. Are there any rules/requirements which demand the current treatement to generate and/or to propagate the error? 


## Examples

Let's look at some examples.
Imagine the application for, well, without trying to be original, a book store.
We have a monolith backend (written in Java 😉)with some RESTful APIs and a single-page application frontend.

Our example application is made-up to demonstrate the reflection of choosing the place for error handling, so it not trying to propose the real-life use-case scenario implementation.

### Example 1: handling the error -  service returning a result for search query
Use case: user searches for books using some criteria.

At the backend, let's imagine that we need to run a  treatment for each book in a list of results: for example we are checking if the book is available in a warehouse, because we want our customers to see only available items they can order. And for that we call a third party service with the following API:

``` java
public interface ThirdPartyWarehouseManager {
  /*
  * Returns 'true' if the item with the specified reference is in the stock.
  * @throws ServiceUnavailable if the service is unavailable for whatever reason.
  */
  public boolean isAvailable(String reference) throws ServiceUnavailable;
}
```
And here are backend services:

```java

public class WarehouseAvailabilityChecker {
  private final ThirdPartyWarehouseManager warehouseManager;

  //...

  public boolean isAvailable(Book book) {
    try {
      return warehouseManager.isAvailable(book.getReference());
    } catch (ServiceUnavailable e) {
      // propagate or handle the exception? if handling, what will be the meaningful result to return?
    }
  }
}

public class BookSearch {
  private final BooksDataStorage storage;
  private final WarehouseAvailabilityChecker availabilityChecker;

  //....

  public List<Book> search(Criteria criteria) {
    List<Book> books = storage.findByCriteria(criteria);
    List<Book> booksAvailable = new ArrayList<>();
    // I'm not using lambdas here intentionally :)
    for(Book b: books) {
      if(availabilityChecker.isAvailable(b) { // what should happen if there is an error?
        booksAvailable.add(b);
      }
    }
    return booksAvailable;
  }
}
```

Let's start with the callee, the `WarehouseAvailabilityChecker` service.
It seems obvious that if the third party service is unavailable, we can't respond neither `true` nor `false`, since, well, we don't know. In similar situations it could be tempting to respond the default value, but, doing so is completely misguiding. In this case for this particular interface declaration we'll propagate the exception wrapping it into an application exception:

```java
public class WarehouseAvailabilityChecker {
  private final ThirdPartyWarehouseManager warehouseManager;

  //...

  public boolean isAvailable(Book book) throws WarehouseException {
    try {
      return warehouseManager.isAvailable(book.getReference());
    } catch (ServiceUnavailable e) {
      throw new WarehouseException("Information is not available", e);
    }
  }
}
```
Now the caller need to decide what it should do with the exception. Here there is no general recipe. All depends on the context. In our case we'd like to show to the end user only those book which we know are available for sure. And if there is an error with checking some books, we prefer to show the available ones anyway. 
In my case web service for book search calls `BookSearch` service and passes the JSON serialized result to the user. I will omit its implementation in the example.

```java
public class BookSearch {
  private final BooksDataStorage storage;
  private final WarehouseAvailabilityChecker availabilityChecker;

  //....

  public List<Book> search(Criteria criteria) {
    List<Book> books = storage.findByCriteria(criteria);
    List<Book> booksAvailable = new ArrayList<>();

    for(Book b: books) {
      try {
        if (availabilityChecker.isAvailable(b)) { // what should happen if there is an error?
          booksAvailable.add(b);
        }
      } catch (ServiceUnavailable e) {
        log.info("Error while checking Availability for the book {}, ignoring the error.", b.getTitle());
      }
    }

    return booksAvailable;
  }
}
```

We can explore other possibilities which can give us the same desired result.
For example by reconsidering  `WarehouseAvailabilityChecker` interface:

```java

public class WarehouseAvailabilityChecker {
  public enum Availability {
    YES,
    NO,
    UNKNOWN
  }
  private final ThirdPartyWarehouseManager warehouseManager;

  //...

  public Availability isAvailable(Book book) throws WarehouseException {
    try {
      return warehouseManager.isAvailable(book.getReference()) ? Availability.YES : Availability.NO;
    } catch (ServiceUnavailable e) {
      log.info("Information is not available, reason: {}", e.getMessage());
      return Availability.UNKNOWN;
    }
  }
}
```

And the corresponding search service will look a bit simpler:

```java
public class BookSearch {
  private final BookStorage storage;
  private final WarehouseAvailabilityChecker availabilityChecker;
  //....

  public List<Book> search(Criteria criteria) {
    List<Book> books = storage.findByCriteria(criteria);
    List<Book> booksAvailable = books.stream() 
                                .filter(b -> availabilityChecker.isAvailable(b) == Availability.YES)
                                .collect(Collectors.toList());
    return booksAvailable;
  }
}
```

### Example 2: propagating the error -  web service to consult the details of a book

Now another user case: showing details of a book chosen by the user.

Book data is retrieved from the data base. And sometimes (due to high load) the database doesn't respond and we have the connection timeout error.
My persistence service is implemented using JPA with Hibernate. And the web service is based on Spring MVC.
Here I'll concentrate on two two problems to solve: requested book is not found and connection to data base is timed out.

```java
public class BookStorageImpl implements BookStorage {
  // using jpa with hibernate 
  private BookRepository repository;
  
  //....

  public Book getByReference(Reference reference) {
    // how to handle the error which here?
    try { 
      return repository.getOne(reference.getValue());
    } catch (EntityNotFoundException  enfe) { // runtime exception thrown by the persistence provider
      // what can we do here ?
    } catch (QueryTimeoutException qte) { // runtime exception thrown by the persistence provider
      // what can we do here ?
    }  
  }
}

public class BookAccess {
  BookStorageAccess storage;
  //....

  @Transactional(readOnly = true)
  public BookDetails getDetails(Reference reference) {
    // how to handle errors ?
    Book book = storage.getByReference(criteria);
    return book.getDetails();
  }
}
```

It is worth mentionning that [`EntityNotFoundException`](https://docs.jboss.org/hibernate/jpa/2.2/api/javax/persistence/EntityNotFoundException.html) and [`QueryTimeoutException`](https://docs.jboss.org/hibernate/jpa/2.2/api/javax/persistence/QueryTimeoutException.html) are runtime exceptions and they would be propagated up to web service and, if not caught, would cause Internal Server Error (HTTP code 500) returned to the front application.
That default behavior doesn't suite me, because :

* I'd like my web service to return Not Found error (HTTP code 404) in case the requested book doesn't exist.
* I want Service Unavailable code (HTTP code 503) if the data base is not responding.

To acheave this I prefer to catch persistance provider exceptions at persistance layer and transform them in business exception.This ways I decouple persistance layer from the rest of the application and define a clear interface. So I introduce two business exceptions `BookNotFoundException` and `StorageNotAvailableException`.

```java
public interface BookStorage {
  Book getByReference(Reference reference) throws BookNotFoundException, StorageNotAvailableException;
}
```

In a business layer I will simply propagate new business exceptions. And on web service layer I will create exception handlers to transform them into desired error responses.
You can find more about creating exception handlers in a Spring Web application in my article ["Spring Boot and Spring MVC: how to handle controller exceptions properly"](./2019-03-18-spring-MVC-exception-handling.md).

So that's how my services look now:

```java
public class BookStorage {
  // using jpa with hibernate 
  BookRepository repository;
  
  //....

  public Book getByReference(Reference reference) throws BookNotFoundException {
    try { 
      return repository.getOne(reference.getValue());
    } catch (EntityNotFoundException  enfe) {
      throw new BookNotFoundException( String.format("No book with the reference %s found", reference.getValue()), enfe);
    } catch (QueryTimeoutException qte) { 
      throw new StorageNotAvailableException("Storage is service is unavailable", qte);
    } 
  }
}

public class BookAccess {
  BookStorageAccess storage;
  //....

  @Transactional(readOnly = true)
  public BookDetails getDetails(Reference reference) throws BookNotFoundException, StorageNotAvailableException {
    Objects.requireNotNull("Reference is mandatory, can not ne null");
    Book book = storage.getByReference(criteria);
    return book.getDetails();
  }
}
```



# Takeaways 

1. There is no one-fit-all recipe for error handling. Everything depends on the context: business rules and constraints and the use-case requirements.
2. Always think trough all error-handling scenarios on the  way of the error up to the highest layer of the application (usually presentation or web services layer). To find the right place to handle the error think of: 
   *  How the application should react to the error in each particular piece of code?
   *  If there are several reactions possible, which one suites best to fulfill the use-case requirements and the business rules?
   *  What will be the impact of this reaction to upper layers (if any)?


# Resources

* [Oracle Java SE tutorial: Exceptions](https://docs.oracle.com/javase/tutorial/essential/exceptions/index.html)


# Credits

Diagram is created using [draw.io](draw.io)