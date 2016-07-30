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
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)

public <U> CompletableFuture<U> handle(BiFunction<? super T,Throwable,? extends U> fn)
public <U> CompletableFuture<U> handleAsync(BiFunction<? super T,Throwable,? extends U> fn)
public <U> CompletableFuture<U> handleAsync(BiFunction<? super T,Throwable,? extends U> fn, Executor executor)
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
    .whenComplete((v, e) -> System.out.println("(thenApplyFuture) v: " + v));
```

需要注意的是，這些轉換並不是馬上執行的，也不會阻塞，而是在前一個 stage 完成後繼續執行。
它們與 handle 方法的區別在於 handle 方法會處理正常計算值和異常，因此它可以屏蔽異常，避免異常繼續拋出。
而 thenApply 方法只是用來處理正常值，因此一旦有異常就會拋出。

```java
supplyAsyncFuture
    .thenApplyAsync(i->i*2)
    .thenApply(i->i/0)
    .handleAsync((v, e) -> {
        System.out.println(v);
        System.out.println(e);
        return 0;
    }).whenComplete((v, e) -> System.out.println("(thenApplyFuture2) v: " + v));
```

### 純消費(執行Action)
上面的方法是當計算完成的時候，會生成新的計算結果 (thenApply, handle)，或者返回同樣的計算結果 whenComplete，
CompletableFuture 還提供了一種處理結果的方法，只對結果執行 Action ,而不返回新的計算值，因此計算值為 Void:

```java
public CompletableFuture<Void> thenAccept(Consumer<? super T> action)
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action)
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor)
```

看它的參數類型也就明白了，它們是函數式接口 Consumer，這個接口只​​有輸入，沒有返回值。

```java
supplyAsyncFuture.thenAccept(i -> System.out.println("(thenAccept) thenAccept: " + i));
supplyAsyncFuture.thenAcceptAsync(i -> System.out.println("(thenAcceptAsync) thenAcceptAsync: " + i));
```

thenAcceptBoth 以及相關方法提供了類似的功能，當兩個 CompletionStage 都正常完成計算的時候，就會執行提供的 action，
它用來組合另外一個異步的結果。
runAfterBoth 是當兩個 CompletionStage 都正常完成計算的時候,執行一個 Runnable，這個Runnable並不使用計算的結果。

```java
public <U> CompletableFuture<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action)
public <U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action)
public <U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action, Executor executor)
public     CompletableFuture<Void> runAfterBoth(CompletionStage<?> other,  Runnable action)
```

例子如下：
```java
supplyAsyncFuture.thenAcceptBoth(CompletableFuture.completedFuture(20), (x, y) -> {
    System.out.println("(thenAcceptBoth) x: " + x);
    System.out.println("(thenAcceptBoth) y: " + y);
});
```

更徹底地，下面一組方法當計算完成的時候會執行一個 Runnable,與 thenAccept 不同，Runnable 並不使用 CompletableFuture 計算的結果。
```java
public CompletableFuture<Void> thenRun(Runnable action)
public CompletableFuture<Void> thenRunAsync(Runnable action)
public CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor)
```

因此先前的 CompletableFuture 計算的結果被忽略了,這個方法返回 CompletableFuture<Void> 類型的對象。
```java
supplyAsyncFuture.thenRun(() -> System.out.println("all finished."));
```

因此，你可以根據方法的參數的類型來加速你的記憶。 Runnable 類型的參數會忽略計算的結果，Consumer 是純消費計算結果，
BiConsumer 會組合另外一個 CompletionStage 純消費，Function 會對計算結果做轉換，BiFunction 會組合另外一個 CompletionStage 的計算結果做轉換。

### 组合
```java
public <U> CompletableFuture<U> thenCompose(Function<? super T,? extends CompletionStage<U>> fn)
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T,? extends CompletionStage<U>> fn)
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T,? extends CompletionStage<U>> fn, Executor executor)
```

這一組方法接受一個 Function 作為參數，這個 Function 的輸入是當前的 CompletableFuture 的計算值，返回結果將是一個新的CompletableFuture，
這個新的 CompletableFuture 會組合原來的 CompletableFuture 和函數返回的 CompletableFuture。因此它的功能類似:

記住，thenCompose 返回的對象並不一是函數fn返回的對象，如果原來的 CompletableFuture 還沒有計算出來，它就會生成一個新的組合後的 CompletableFuture。

例子如下：
```java
supplyAsyncFuture
    .thenCompose(i -> CompletableFuture.supplyAsync(() -> i + 3))
    .whenCompleteAsync((v, e) -> System.out.println("(thenCompose) v: " + v));
```

而下面的一組方法 thenCombine 用來複合另外一個 CompletionStage 的結果。
兩個 CompletionStage 是並行執行的，它們之間並沒有先後依賴順序，other 並不會等待先前的 CompletableFuture 執行完畢後再執行。

```java
public <U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
public <U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
public <U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor)
```

其實從功能上來講,它們的功能更類似 thenAcceptBoth，只不過 thenAcceptBoth 是純消費，它的函數參數沒有返回值，而 thenCombine 的函數參數 fn 有返回值。

```java
thenCombineFuture1.thenCombine(thenCombineFuture2, (x, y) -> x + y)
    .whenComplete((v, e) -> System.out.println("(thenCombine) v: " + v));
```

### Either
thenAcceptBoth 和 runAfterBoth 是當兩個 CompletableFuture 都計算完成，而我們下面要了解的方法是當任意一個 CompletableFuture 計算完成的時候就會執行。

```java
public CompletableFuture<Void> 	acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action)
public CompletableFuture<Void> 	acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action)
public CompletableFuture<Void> 	acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor)
public <U> CompletableFuture<U> applyToEither(CompletionStage<? extends T> other, Function<? super T,U> fn)
public <U> CompletableFuture<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn)
public <U> CompletableFuture<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn, Executor executor)
```


acceptEither 方法是當任意一個 CompletionStage 完成的時候，action 這個消費者就會被執行。這個方法返回 CompletableFuture<Void>
applyToEither 方法是當任意一個 CompletionStage完成的時候，fn 會被執行，它的返回值會當作新的 CompletableFuture<U> 的計算結果。

下面這個例子有時會輸出 1000,有時候會輸出 2000,哪個 Future 先完成就會根據它的結果計算。

```java
Random rand = new Random();
CompletableFuture<Integer> applyToEitherFuture1 = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(jobSleep + rand.nextInt(1000));
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 1000;
});
CompletableFuture<Integer> applyToEitherFuture2 = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(jobSleep + rand.nextInt(1000));
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 2000;
});

applyToEitherFuture1.applyToEither(applyToEitherFuture2, i -> i)
    .whenComplete((v, e) -> System.out.println("(applyToEither) v: " + v)) ;
```

### 輔助方法 allOf 和 anyOf
前面我們已經介紹了幾個靜態方法：completedFuture、runAsync、supplyAsync,
下面介紹的這兩個方法用來組合多個 CompletableFuture。

```java
public static CompletableFuture<Void>   allOf(CompletableFuture<?>... cfs)
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)
```

allOf 方法是當所有的 CompletableFuture 都執行完後執行計算。
anyOf 方法是當任意一個 CompletableFuture 執行完後就會執行計算，計算的結果相同。

下面的代碼運行結果有時是 100, 有時是 "abc"。但是 anyOf 和 applyToEither 不同。
anyOf 接受任意多的 CompletableFuture 但是 applyToEither 只是判斷兩個 CompletableFuture,
anyOf 返回值的計算結果是參數中其中一個 CompletableFuture 的計算結果，applyToEither 返回值的計算結果卻是要經過 fn 處理的。
當然還有靜態方法的區別，線程池的選擇等。

```java
CompletableFuture.anyOf(applyToEitherFuture1, applyToEitherFuture2)
    .whenComplete((v, e) -> System.out.println("(anyOf) v: " + v)) ;
```

### 參考文檔
#### http://colobu.com/2016/02/29/Java-CompletableFuture/#主动完成计算，
#### 並使用自已的 sample code 展示.
#### sample code: https://github.com/wangchenshu/mycompletablefuture