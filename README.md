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

### 參考文檔
http://colobu.com/2016/02/29/Java-CompletableFuture/#主动完成计算