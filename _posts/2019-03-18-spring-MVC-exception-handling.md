---
title: "Spring Boot and Spring MVC: how to handle controller exceptions properly"
categories:
  - Back-end
tags:
  - java
  - spring-boot
---

> **TL;DR** In depth explanation of exception handling for Spring MVC in a Spring Boot application: exception handler types, handlers ordering, available customization means.

 **Table of contents**

* [Spring Boot auto-configuration for error handling](#spring-boot-auto-configuration-for-error-handling)
* [Error page customization](#error-page-customization)
  - [Customizing global error handler](#customizing-global-error-handler)
  - [Customizing error view page](#customizing-error-view-page)
* [More customization: fine grained exception handlers](#more-customization-fine-grained-exception-handlers)
  - [Spring MVC exception handlers standard configuration](#spring-mvc-exception-handlers-standard-configuration)
  - [Convenient base class for centralized exception handling](#convenient-base-class-for-centralized-exception-handling)
* [Takeaways](#takeaways)

## Spring Boot auto-configuration for error handling

Spring Boot application has a default configuration for error handling - [ErrorMvcAutoConfiguration](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/servlet/error/ErrorMvcAutoConfiguration.java).

What it basically does, if no additional configuration provided:

* it creates default global error controller  - [BasicErrorController](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/servlet/error/BasicErrorController.java)
* it creates default 'error' static view 'Whitelabel Error Page'.

`BasicErrorController` is wired to '/error' view by default. If there is no customized 'error' view in the application, in case of an exception thrown from any controller, the user lands to `/error` whitelabel page, filled with information by BasicErrorController.

## Error page customization

As we saw earlier, Spring MVC' default error page is `/error`. Default controller behind `/error` endpoint is `BasicErrorController`.
Let's look how we can customize error handling.

### Customizing global error handler

`BasicErrorController` is a basic error controller rendering `ErrorAttributes`.
It can either render an html page or provide a serialized `ErrorAttributes` object. 

There are several ways to customize the behavior of error handling and/or the contents of the error view:

1. The Spring Boot application can provide its own error handling controller. It must implement `ErrorController` interface to automatically replace `BasicErrorController`.
2. The application can provide custom view servlet server page(s) and custom view content. For an html view, the page format can be either `jps` or `html` with [Thymleaf](https://www.thymeleaf.org/) to render model attributes filled by the error controller. 
3. The application can do both: customize the behavior by implementing `ErrorController` and customize the error page view and/or error model object.

### Customizing error view page

The default error page, which you already saw perhaps, is a 'Whitelabel error page' static view (created in the code directly) configured within `ErrorMvcAutoConfiguration`. It is served by default by `/error` endpoint.

You can keep the endpoint and just replace the static view by a custom page.
For it to be picked automatically without any configuration, the page must be called `error.html` and it must be placed in `src/main/resources/templates`folder.

And here is an example of a view we could come up to:

```html
<!DOCTYPE html>
<html>
    <body>
        <h1>Oups! An error has occurred trying to process your request. </h1>
        <h2>No panic. We are already working on it!</h2>
        <a href="/">Back to Home</a>
    </body>
</html>
```

> :bulb: once you have replaced the default static view, you can disable id with `server.error.whitelabel.enabled=false` in the application.properties. 

> :bulb: if you with to name your global error page differently, you can use `server.error.path` application property. For example: `server.error.path=/oups`, which should have the corresponding `oups.html` view.

Another approach for error handling is to have several error views, each for a particular error. It is always preferable to provide error details as accurate as possible for user to understand and to respond accordingly.
For example, we can provide pages for particular errors: `400.html`, `404.html`, `500.html` and a generic `error.html` view.

In this case we can either redirect to these pages **from the error controller**:

```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
CustomizedErrorController implements ErrorController {
    ...
    @GetMapping(produces = MediaType.TEXT_HTML_VALUE)
    public String handleError(HttpServletRequest request, Model model) {
        ...
        //retrieving http error status code set by the underlying container
        Object status = request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);

        if (status != null) {
            HttpStatus httpStatus = HttpStatus.resolve((int) status);
            switch(httpStatus) {
                case HttpStatus.BAD_REQUEST:
                    return "400";
                case HttpStatus.NOT_FOUND:
                    return "404";
                case HttpStatus.INTERNAL_SERVER_ERROR:
                    return "500";
            }
        }
        return "error";
    }
}
```

Or we can redirect from our **application business controllers directly**, if the expected result is a page:
```java
@Controller
@RequestMapping("/user}")
UserController implements ErrorController {
    ...
    @GetMapping(produces = MediaType.TEXT_HTML_VALUE)
    public String getUser(HttpServletRequest request, Model model) {
        User user = getUser();
        if(user == null) {
            return "404";
        }
        model.addAttribute("user", user);
        return "user"; // user information page
    }
}
```


## More customization: fine grained exception handlers

`BasicErrorController` or `ErrorController` implementation is a **global error handler**. To handle more specific errors, Spring documentation suggest to use `@ExceptionHandler` method annotations or `@ResponseStatus` exception class annotations.

So we have a global error handler, `@ExceptionHandler` methods and `@ResponseStatus` exception classes working together.
But in which order? Which one has the effect first?

Let's see in details how it works.

### Spring MVC exception handlers standard configuration

[WebMvcConfigurationSupport](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurationSupport.html) provides configuration behind the Spring MVC Java config, which is enabled by [@EnableWebMvc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/EnableWebMvc.html).

This class registers several beans including the exception handler bean - [HandlerExceptionResolverComposite]().
The composite exception resolver delegates exception handling to the the list of handlers it holds. Exception is passed through the list of handlers until one of them returns a result (something which is not `null`). This result is considered as the final result of error processing.
`WebMvcConfigurationSupport` adds the following exception handlers into `HandlerExceptionResolverComposite` in the following order:
1. ExceptionHandlerExceptionResolver -  for handling exceptions through @ExceptionHandler annotated methods.
2. ResponseStatusExceptionResolver - for exceptions annotated with @ResponseStatus
3. DefaultHandlerExceptionResolver - for resolving known Spring exception types (e.g. `HttpRequestMethodNotSupportedException`)

If none of these handlers returns a result, the exception is passed to the underlying container.

The underlying container forwards the exception to the registered error page. In our case this is  `\error`.
Starting from this point we already know [how](#error-page) the exception is handled afterwards. Great! 

Here is a diagram summarizing Spring MVC default behavior for handling application exceptions:

{% include figure image_path="assets/images/Spring-MVC-default-exception-handling.svg" alt="Spring MVC exception handling - default configuration" caption="Spring MVC exception handling - default configuration" %}

### Convenient base class for centralized exception handling

To simplify exception handling using @ExceptionHandler, Spring proposes [ResponseEntityExceptionHandler](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ResponseEntityExceptionHandler.html)
> A convenient base class for @ControllerAdvice classes that wish to provide centralized exception handling across all @RequestMapping methods through @ExceptionHandler methods.
> This base class provides an @ExceptionHandler method for handling internal Spring MVC exceptions.

In other words, if we want centralized exception handling, e.g. everything in one class, we can use `ResponseEntityExceptionHandler` as a base class. We can use as is or override Spring MVC exception handlers provided by `ResponseEntityExceptionHandler`. For our custom class to work, it must be annotated with `@ControllerAdvice`.

Classes annotated with `@ControllerAdvice` can be used at the same time as the global error handler (`BasicErrorController` or `ErrorController` implementation). If our `@ControllerAdvice` annotated class (which can/or not extend `ResponseEntityExceptionHandler`) doesn't handle some exception, the exception goes to the global error handler.


## Takeaways

If application has a controller implementing `ErrorController` it **replaces** `BasicErrorController`.

If any exception occurs in error handling controller, it will go through Spring exception handlers filter and finally if nothing is found this exception will be handled by the underlying application container, e.g. Tomcat. The underlying container will handle the exception and show some error page/message depending on its implementation.

Spring exception handlers default order:

1. First Spring searches for an exception handler (a method annotated with `@ExceptionHandler`) within the classes annotated with `@ControllerAdvice`. This behavior is implemented in [ExceptionHandlerExceptionResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ExceptionHandlerExceptionResolver.html).
2. Then it checks if the thrown exception is annotated with `@ResponseStatus` or if it derives from `ResponseStatusException`. If so it is handled by [ResponseStatusExceptionResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/annotation/ResponseStatusExceptionResolver.html).
3. Then it goes through Spring MVC exceptions' default handlers. These are in [DefaultHandlerExceptionResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/support/DefaultHandlerExceptionResolver.html).
4. And at the end if nothing is found, the control is forwarded to the configured error page view with the global error handler behind it. This step is not executed if the exception comes from the error handler itself.
5. If no error view is found (e.g. global error handler is disabled) or step 4 is skipped, then the exception is handled by the container.