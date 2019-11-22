---
layout: post
title:  "Can I override private code snippets of a parent class?"
categories: development
tag: java
---
<!--more-->
There is a class containing hundreds of code lines with several private functions inside, also a Counter metric inline. In the child class, I want to implement a different protected function, but I also want to use a different Counter. 

Parent class looks like:
```java
class Parent {
    private static Counter counter = Counter.build("parent_service_result_count",
        "Count the response of parent service, group by its status.")
        .labelNames("status")
        .register();
    public List<String> output(List<IngestionEvent> ingestionEventList) {
        //do something and call remote service
        return handleResponse(resp);
    }
    private ServiceResponse handleResponse(Response response) {
        if (payload.statusCode() == HttpStatus.OK.value()) {
            //do something
            counter.labels("OK").inc();
        } else if (payload.statusCode() == HttpStatus.ALREADY_REPORTED.value()) {
            //do something
            counter.labels("Conflict").inc();
        } else if (payload.statusCode() >= 500) {
            //do something
            counter.labels("Error").inc();
        } else {
            //do something
            counter.labels("Unknown").inc();
        }
    }
}

```

One word popped up in my mind: override.
Can I override the private counter variable with a new one defined in child class like:
```java
class Child {
    private static Counter counter = Counter.build("child_service_result_count",
        "Count the response of child service, group by its status.")
        .labelNames("status")
        .register();
```
This does not work because the override in Java is not erasing parent field and pointing to a child field. Defining a same variable in child class and parent class in Java, means we hide the value in super, but we can still get the value with super.parentValue; the method of parent class still access to its own value.

When I updated the access function with a getter method, the override works because override is defined in method category.
```java
class Parent {
    private static Counter counter = Counter.build("parent_service_result_count",
        "Count the response of parent service, group by its status.")
        .labelNames("status")
        .register();
    //This is the difference
    public Counter getCounter() {
        return counter;
    }
    public List<String> output(List<IngestionEvent> ingestionEventList) {
        //do something and call remote service
        return handleResponse(resp);
    }
    private ServiceResponse handleResponse(Response response) {
        if (payload.statusCode() == HttpStatus.OK.value()) {
            //do something
            getCounter().labels("OK").inc();
        } else if (payload.statusCode() == HttpStatus.ALREADY_REPORTED.value()) {
            //do something
            getCounter().labels("Conflict").inc();
        } else if (payload.statusCode() >= 500) {
            //do something
            getCounter().labels("Error").inc();
        } else {
            //do something
            getCounter().labels("Unknown").inc();
        }
    }
}

class Child {
    private static Counter counter = Counter.build("child_service_result_count",
        "Count the response of child service, group by its status.")
        .labelNames("status")
        .register();
    @Override
    public Counter getCounter() {
        return counter;
    }
```
To understand how the code works, create experiments is the best way:)