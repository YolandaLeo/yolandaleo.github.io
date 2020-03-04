---
layout: post
categories: 'development'
tag: java
title: Chaining CompletionStage of CompletableFuture
---
Since Java8, CompletableFuture is introduced to provide flexible chains of non-blocking operators. CompletableFuture is an implementation of both Future<T> and CompletionStage<T>. Let's see how CompletionStages are chained together and get things done.

Can you tell the print result of the following code sample?
<!--more-->

```java
public static void main(String[] args) {
        CompletableFuture future1 = CompletableFuture.supplyAsync(() -> {
            System.out.println("future1 run with thread " + Thread.currentThread().getName());
            return "future1";
        });

        CompletableFuture future2 = CompletableFuture.supplyAsync(() -> {
            System.out.println("future2 run with thread " + Thread.currentThread().getName());
            return "future2";
        }).whenCompleteAsync((res, ex) -> {
                System.out.println("future2 whenComplete");
            });

        CompletableFuture future3 = CompletableFuture.supplyAsync(() -> {
            System.out.println("future3 run with thread " + Thread.currentThread().getName());
            System.out.println();
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            throw new RuntimeException("test exception" + System.currentTimeMillis());
        }).whenComplete((res, es) -> {
            System.out.println("future3 whenComplete");
        });

        CompletableFuture future4 = CompletableFuture.runAsync(() -> {System.out.println("future4 supplyAsync with thread " + Thread.currentThread().getName());})
            .thenApply(a -> {System.out.println("future4.thenApply");
            return a;})
            .whenComplete((res, ex) -> {
            System.out.println("future4 whenComplete");
        });

        CompletableFuture.allOf(future1, future2, future3).whenComplete((res, ex) -> {
            System.out.println("all complete");
            if (ex != null) {
                System.out.println("complete with exception " + System.currentTimeMillis());
            }
        });
    }
```
A little tricky that main thread will exit before future3 completes, so the outputn will be:
```
future1 run with thread ForkJoinPool.commonPool-worker-1
future2 run with thread ForkJoinPool.commonPool-worker-1
future2 whenComplete
future3 run with thread ForkJoinPool.commonPool-worker-2

future4 supplyAsync with thread ForkJoinPool.commonPool-worker-1
future4.thenApply
future4 whenComplete
Disconnected from the target VM, address: '127.0.0.1:55896', transport: 'socket'
```

To change slightly in original code snippet:
```java
CompletableFuture.allOf(future1, future2, future3).whenComplete((res, ex) -> {
            System.out.println("all complete");
            if (ex != null) {
                System.out.println("complete with exception " + System.currentTimeMillis());
            }
        }).join();
```
We'll see the result of all futures to finish.
```
future1 run with thread ForkJoinPool.commonPool-worker-1
future2 run with thread ForkJoinPool.commonPool-worker-2
future2 whenComplete
future3 run with thread ForkJoinPool.commonPool-worker-2

future4 supplyAsync with thread ForkJoinPool.commonPool-worker-1
future4.thenApply
future4 whenComplete
future3 whenComplete
all complete
complete with exception 1583313791247
Exception in thread "main" java.util.concurrent.CompletionException: java.lang.RuntimeException: test exception1583313791246
	at java.util.concurrent.CompletableFuture.encodeThrowable(CompletableFuture.java:273)
	at java.util.concurrent.CompletableFuture.completeThrowable(CompletableFuture.java:280)
	at java.util.concurrent.CompletableFuture$AsyncSupply.run$$$capture(CompletableFuture.java:1592)
	at java.util.concurrent.CompletableFuture$AsyncSupply.run(CompletableFuture.java)
	at java.util.concurrent.CompletableFuture$AsyncSupply.exec(CompletableFuture.java:1582)
	at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:289)
	at java.util.concurrent.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1056)
	at java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1692)
	at java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:157)
Caused by: java.lang.RuntimeException: test exception1583313791246
	at util.FutureTest.lambda$main$3(FutureTest.java:32)
	at java.util.concurrent.CompletableFuture$AsyncSupply.run$$$capture(CompletableFuture.java:1590)
	... 6 more
Disconnected from the target VM, address: '127.0.0.1:60226', transport: 'socket'
```
We can see that, not only the complete callback of **allOf** operator executed, but also all of complete callbacks of **future2, future3**.

In the code, the methods *thenApply*, *whenComplete* and also lots of methods are from ComletionStage interface, also return a CompletionStage instance. For example:
```java
public CompletionStage<T> whenComplete
        (BiConsumer<? super T, ? super Throwable> action);
```
When all of future1, future2 and future3 completed, allOf function will collect all the CompletionStages and chain with another CompletionStage. This is useful when there are several tasks run parallely and need something as a unifiend callback when all tasks were done.
