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
### 參考文檔
http://colobu.com/2016/02/29/Java-CompletableFuture/#主动完成计算