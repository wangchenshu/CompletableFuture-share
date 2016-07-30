# CompletableFuture-share

## 主動完成計算

### 創建CompletableFuture對象

> CompletableFuture.completedFuture是一個靜態輔助方法，用來返回一個已經計算好的CompletableFuture

```java
public static <U> CompletableFuture<U> completedFuture(U value)
```

而以下四個靜態方法用來為一段異步執行的代碼創建 CompletableFuture對象：
```java
public static CompletableFuture<Void> runAsync(Runnable runnable)
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
```

以Async結尾並且沒有指定Executor的方法會使用ForkJoinPool.commonPool()作為它的線程池執行異步代碼。
runAsync方法也好理解，它以Runnable函數式接口類型為參數，所以CompletableFuture的計算結果為空。
supplyAsync方法以Supplier<U>函數式接口類型為參數,CompletableFuture的計算結果類型為U。

```java
// supplyAsync-example
CompletableFuture<Integer> supplyAsyncFuture = CompletableFuture.supplyAsync(() -> {
    int i = 100;
    return i;
});

// runAsync-example
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> out.println("hello, runAsync."));
```

### 計算結果完成時的處理
> 當CompletableFuture的計算結果完成，或者拋出異常的時候，我們可以執行特定的Action。主要是下面的方法：

```java
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
public CompletableFuture<T> exceptionally(Function<Throwable,? extends T> fn)
```

可以看到Action的類型是BiConsumer<? super T,? super Throwable>，它可以處理正常的計算結果，或者異常情況。
方法不以Async結尾，意味著Action使用相同的線程執行，而Async可能會使用其它的線程去執行(如果使用相同的線程池，也可能會被同一個線程選中執行)。
注意這幾個方法都會返回CompletableFuture，當Action執行完畢後它的結果返回原始的CompletableFuture的計算結果或者返回異常。

```java
//CompletableFuture<Integer> supplyAsyncFuture = CompletableFuture.supplyAsync(() -> 10000);
CompletableFuture<Integer> supplyAsyncFuture = CompletableFuture.supplyAsync(() -> 10000/0);

supplyAsyncFuture.whenComplete((v, e) -> {
    out.println("(whenComplete) v: " + v);
    out.println("(whenComplete) e: " + e);
});

supplyAsyncFuture.whenCompleteAsync((v, e) -> {
    out.println("(whenCompleteAsync) v: " + v);
    out.println("(whenCompleteAsync) e: " + e);
});

supplyAsyncFuture.exceptionally(e -> {
    out.println("(exceptionally) ex: " + e);
    return 0;
}).whenCompleteAsync((v, e) -> {
    out.println("(exceptionally) v: " + v);
    out.println("(exceptionally) e: " + e);
});
```
### 參考文檔
http://colobu.com/2016/02/29/Java-CompletableFuture/#主动完成计算