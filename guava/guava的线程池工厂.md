# guava的线程池工厂

## MoreExecutors工厂类

todo

## DirectExecutor「同步」

```java
/**
 * An {@link Executor} that runs each task in the thread that invokes {@link Executor#execute
 * execute}.
 */
@GwtCompatible
@ElementTypesAreNonnullByDefault
enum DirectExecutor implements Executor {
  INSTANCE;

  //由调用线程同步执行
  @Override
  public void execute(Runnable command) {
    command.run();
  }

  @Override
  public String toString() {
    return "MoreExecutors.directExecutor()";
  }
}
```

