# CompletableFuture-share

## 主動完成計算

### 創建 CompletableFuture 對象

> CompletableFuture.completedFuture 是一個靜態輔助方法，用來返回一個已經計算好的 CompletableFuture

```java
public static <U> CompletableFuture<U> completedFuture(U value)
```

而以下四個靜態方法用來為一段異步執行的代碼創建 CompletableFuture 對象：
```java
public static CompletableFuture<Void> runAsync(Runnable runnable)
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
```

以 Async 結尾並且沒有指定 Executor 的方法會使用 ForkJoinPool.commonPool() 作為它的線程池執行異步代碼。
runAsync 方法也好理解，它以 Runnable 函數式接口類型為參數，所以 CompletableFuture 的計算結果為空。
supplyAsync 方法以 Supplier<U> 函數式接口類型為參數, CompletableFuture 的計算結果類型為U。

```java
// supplyAsync-example
CompletableFuture<Integer> supplyAsyncFuture = CompletableFuture.supplyAsync(() -> {
    int i = 100;
    return i;
});

// runAsync-example
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> System.out.println("hello, runAsync."));
```

### 計算結果完成時的處理
> 當 CompletableFuture 的計算結果完成，或者拋出異常的時候，我們可以執行特定的 Action。主要是下面的方法：

```java
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
public CompletableFuture<T> exceptionally(Function<Throwable,? extends T> fn)
```

可以看到 Action 的類型是 BiConsumer<? super T,? super Throwable>，它可以處理正常的計算結果，或者異常情況。
方法不以 Async 結尾，意味著 Action 使用相同的線程執行，而 Async 可能會使用其它的線程去執行(如果使用相同的線程池，也可能會被同一個線程選中執行)。
注意這幾個方法都會返回 CompletableFuture，當 Action 執行完畢後它的結果返回原始的 CompletableFuture 的計算結果或者返回異常。

```java
CompletableFuture<Integer> supplyAsyncFuture = CompletableFuture.supplyAsync(() -> 10000);
//CompletableFuture<Integer> supplyAsyncFuture = CompletableFuture.supplyAsync(() -> 10000/0);

supplyAsyncFuture.whenComplete((v, e) -> {
    System.out.println("(whenComplete) v: " + v);
    System.out.println("(whenComplete) e: " + e);
});

supplyAsyncFuture.whenCompleteAsync((v, e) -> {
    System.out.println("(whenCompleteAsync) v: " + v);
    System.out.println("(whenCompleteAsync) e: " + e);
});

supplyAsyncFuture.exceptionally(e -> {
    System.out.println("(exceptionally) ex: " + e);
    return 0;
}).whenCompleteAsync((v, e) -> {
    System.out.println("(exceptionally) v: " + v);
    System.out.println("(exceptionally) e: " + e);
});
```

exceptionally 方法返回一個新的 CompletableFuture，當原始的 CompletableFuture 拋出異常的時候，
就會觸發這個 CompletableFuture 的計算，調用 function 計算值，否則如果原始的 CompletableFuture 正常計算完後，
這個新的 CompletableFuture 也計算完成，它的值和原始的 CompletableFuture 的計算的值相同。
也就是這個 exceptionally 方法用來處理異常的情況。

下面一組方法雖然也返回 CompletableFuture 對象，但是對象的值和原來的 CompletableFuture 計算的值不同。
當原先的 CompletableFuture 的值計算完成或者拋出異常的時候，會觸發這個 CompletableFuture 對象的計算，
結果由 BiFunction 參數計算而得。因此這組方法兼有 whenComplete 和轉換的兩個功能。

### 轉換
CompletableFuture 可以作為 monad(單子) 和 functor。由於回調風格的實現，我們不必因為等待一個計算完成而阻塞著調用線程，
而是告訴 CompletableFuture 當計算完成的時候請執行某個 function。
而且我們還可以將這些操作串聯起來，或者將 CompletableFuture 組合起來。

```java
public <U> CompletableFuture<U> 	thenApply(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> 	thenApplyAsync(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> 	thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)

public <U> CompletableFuture<U> 	handle(BiFunction<? super T,Throwable,? extends U> fn)
public <U> CompletableFuture<U> 	handleAsync(BiFunction<? super T,Throwable,? extends U> fn)
public <U> CompletableFuture<U> 	handleAsync(BiFunction<? super T,Throwable,? extends U> fn, Executor executor)
```

這一組函數的功能是當原來的 CompletableFuture 計算完後，將結果傳遞給函數 fn，將 fn 的結果作為新的 CompletableFuture 計算結果。
因此它的功能相當於將 CompletableFuture<T> 轉換成 CompletableFuture<U>。
這三個函數的區別和上面介紹的一樣，不以 Async 結尾的方法由原來的線程計算，以 Async 結尾的方法由默認的線程池 ForkJoinPool.commonPool()
或者指定的線程池 executor 運行。 Java 的 CompletableFuture 類總是遵循這樣的原則，下面就不一一贅述了。

使用例子如下：
```java
supplyAsyncFuture
    .thenApplyAsync(i->i*2)
    .thenApply(i->i+3)
    .whenComplete((v, e) -> out.println("(thenApplyFuture) v: " + v));
```

需要注意的是，這些轉換並不是馬上執行的，也不會阻塞，而是在前一個 stage 完成後繼續執行。
它們與 handle 方法的區別在於 handle 方法會處理正常計算值和異常，因此它可以屏蔽異常，避免異常繼續拋出。
而 thenApply 方法只是用來處理正常值，因此一旦有異常就會拋出。

```java
supplyAsyncFuture
    .thenApplyAsync(i->i*2)
    .thenApply(i->i/0)
    .handleAsync((v, e) -> {
        out.println(v);
        out.println(e);
        return 0;
    }).whenComplete((v, e) -> out.println("(thenApplyFuture2) v: " + v));
```

### 參考文檔
http://colobu.com/2016/02/29/Java-CompletableFuture/#主动完成计算