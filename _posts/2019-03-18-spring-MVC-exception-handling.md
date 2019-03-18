---
title: "Spring Boot and Spring MVC: how to handle controller exceptions properly"
categories:
  - Back-end
tags:
  - java
  - spring-boot
---


## Spring Boot auto-configuration

Spring Boot application has a default configuration for error handling - [ErrorMvcAutoConfiguration](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/servlet/error/ErrorMvcAutoConfiguration.java).

What it basically does, if no additional configuration provided:

* it creates default global error controller  - [BasicErrorController](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/servlet/error/BasicErrorController.java)
* it creates default 'error' static view 'Whitelabel Error Page'.

`BasicErrorController` is wired to '/error' view by default. If there is no customized 'error' view in the application, in case of an exception thrown from any controller, the user lands to `/error` whitelabel page, filled with information by BasicErrorController.

## Customizing global error handler

`BasicErrorController` is a basic error controller rendering `ErrorAttributes`.
It can either render an html page or provide a serialized `ErrorAttributes` object. 

There are several ways to customize the behavior of error handling and/or the contents of the error view:

1. The Spring Boot application can provide its own error handling controller. It must implement `ErrorController` interface to automatically replace `BasicErrorController`.
2. The application can provide custom view servlet server page(s) and custom view content. For an html view, the page format can be either `jps` or `html` with [Thymleaf](https://www.thymeleaf.org/) to render model attributes filled by the error controller. 
3. The application can do both: customize the behavior by implementing `ErrorController` and customize the error page view and/or error model object.

### Error view page

## More fine grained exception handlers

`BasicErrorController` or `ErrorController` implementation is a **global error handler**. To handle more specific errors, Spring documentation suggest to use `@ExceptionHandler` method annotations or `@ResponseStatus` exception class annotations.

So we have a global error handler, `@ExceptionHandler` methods and `@ResponseStatus` exception classes working together.
But in which order? Which one has the effect first?

Let's see in details how it works.

***TBD** HandlerExceptionResolverComposite, text + img


### Convenient base class

And here it is: [ResponseEntityExceptionHandler](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ResponseEntityExceptionHandler.html)
> A convenient base class for @ControllerAdvice classes that wish to provide centralized exception handling across all @RequestMapping methods through @ExceptionHandler methods.
> This base class provides an @ExceptionHandler method for handling internal Spring MVC exceptions.

In other words, if we want centralized exception handling, e.g. everything in one class, we can use `ResponseEntityExceptionHandler` as a base class. We can use as is or override Spring MVC exception handlers provided by `ResponseEntityExceptionHandler`. For our custom class to work, it must be annotated with `@ControllerAdvice`.

Classes annotated with `@ControllerAdvice` can be used at the same time as the global error handler (`BasicErrorController` or `ErrorController` implementation). If our `@ControllerAdvice` annotated class (which can/or not extend `ResponseEntityExceptionHandler`) doesn't handle some exception, the exception goes to the global error handler.


## Takeaways

If application has a controller implementing `ErrorController` it **replaces** `BasicErrorController`.

If any exception occurs in error handling controller, it will go through Spring exception filter and finally if nothing is found this exception will be handled by the underlying application container, e.g. Tomcat. The underlying container will handle the exception and show some error page/message depending on its implementation.

Spring exception filters default order:

1. First Spring searches for an exception handler (a method annotated with `@ExceptionHandler`) within the classes annotated with `@ControllerAdvice`. This behavior is implemented in [ExceptionHandlerExceptionResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ExceptionHandlerExceptionResolver.html).
2. Then it checks if the thrown exception is annotated with `@ResponseStatus` or if it derives from `ResponseStatusException`. If so it is handled by [ResponseStatusExceptionResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/annotation/ResponseStatusExceptionResolver.html).
3. Then it goes through Spring MVC exceptions' default handlers. These are in [DefaultHandlerExceptionResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/support/DefaultHandlerExceptionResolver.html).
4. And at the end if nothing is found, the control is forwarded to the configured error page view with the global error handler behind it. This step is not executed if the exception comes from the error handler itself.
5. If no error view is found (e.g. global error handler is disabled) or step 4 is skipped, then the exception is handled by the container.