---
layout: post
categories: 'development'
tag: java
title: "How to leverage CompletableFuture to reach non-blocking service dependent system?"
---

For a typical risk evaluation service system, it always depends on several remote services to enrich user or transaction data in order to get a full picture of risk evaluation data.
Here's a scenery that how our service works:

When an evaluation request comes:
1. purchase service is called to get a picture of user purchase; -- A
2. instrument service is called to get full data of user payment instrument; -- B
3. blacklist service is called to check if the payment instrument is blacklisted; -- C
4. blacklist service is called to check if the user purchase shipping address phone is blacklisted; -- D

In this case, we get a DAG graph of execution:
request -> A --> B --> C
             --> D
<!--more-->
If you use one single thread to process the whole workflow, then congratulations you are blocking each call --
    suppose A takes 10 ms, B takes 50ms, C takes 20ms, D takes 50ms, then each request to your service will get at least 130ms response time. Sad story?

To take advantage of multi-threading, in first glance, I may choose to execute the whole workflow in parallel --
```java
    After get the response of A...
    Future<T> fu_B = new Future<> {
        //to execute B
    }
    Future<T> fu_D = new Future<> {
        //to execute D
    }
    C_response = call_C(fu_B.get())
    result.builder().a(A).b(fu_B.get()).c(C).d(fu_D.get())
```

But we can still see blocking points before C, still waiting for B's call to get response.
In Java8, CompletableFuture is introduced to enhance Future and system parallism.
Rewrite the workflow using CompletableFuture:
```java
    ExecutorService executorService = Executors.newCachedThreadPool();
    CompletableFuture<> future_A = CompletableFuture.supplyAsync(new Supplier<Integer>() {
                @Override public Integer get() {
                    System.out.println("Thread: " + Thread.currentThread() + "runAsync A ---");
                    return 1;
                }
            }, executorService);
    CompletableFuture<> future_B = future_A
                .thenApplyAsync(t -> {
                //do B's job using A's response
            }, executorService).thenApplyAsync(t -> {
                //do C's job using B's response
            }, executorService);
    CompletableFuture<> result_future = future_A.thenApplyAsync(t -> {
        //do D's job using A's response
    }).thenComposeAsync(future_B, executorService);
    
```
This is using different threads from executorService to run each node of the workflow. If we do not asign an ExecutorService, the CompletableFuture will use common ForkJoinPool instead.

Here I want to emphasize the difference between thenXX method groups and thenXXAsync.
Take thenApply and thenApplyAsync comparison for example:
```java
public static void main(String[] args) throws ExecutionException, InterruptedException {

        for (Integer i = 0; i < 5; i++) {
            CompletableFuture future = CompletableFuture.supplyAsync(new Supplier<Integer>() {
                @Override public Integer get() {
                    System.out.println("Thread: " + Thread.currentThread() + "runAsync ---");
                    return 1;
                }
            }, executorService);

            future.thenApply(t -> {
                System.out.println("Thread: " + Thread.currentThread() + "thenApply --- 111");
                return 1;
            });

            future.thenApplyAsync(t -> {
                System.out.println("Thread: " + Thread.currentThread() + "thenApplyAsync --- 222");
                return 1;
            }, executorService);

            future.get();
        }

    }
```
Output:
```
Thread: Thread[my-named-thread-0,5,main]runAsync ---
Thread: Thread[main,5,main]thenApply --- 111
Thread: Thread[my-named-thread-0,5,main]thenApplyAsync --- 222
Thread: Thread[my-named-thread-1,5,main]runAsync ---
Thread: Thread[my-named-thread-1,5,main]thenApply --- 111
Thread: Thread[my-named-thread-0,5,main]thenApplyAsync --- 222
Thread: Thread[my-named-thread-0,5,main]runAsync ---
Thread: Thread[main,5,main]thenApply --- 111
Thread: Thread[my-named-thread-1,5,main]thenApplyAsync --- 222
Thread: Thread[my-named-thread-2,5,main]runAsync ---
Thread: Thread[main,5,main]thenApply --- 111
Thread: Thread[my-named-thread-1,5,main]thenApplyAsync --- 222
Thread: Thread[my-named-thread-2,5,main]runAsync ---
Thread: Thread[main,5,main]thenApply --- 111
Thread: Thread[my-named-thread-1,5,main]thenApplyAsync --- 222
```
When chaining a new stage B to stage A using thenApply() method, the thread executing stage B will be chosen from [Current main thread, the thread executing stageA] depends on whether the stage A is complete.

My first glance assumption was stageB will use the same thread of stageA, but CompletableFuture makes no garantee on that. When designing a non-blocking system, we should carefully choose among such a lot of methods and take care of possible side effects.
