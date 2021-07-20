# RateLimiter源码解析

## 成员

```java
/**
 * 底层秒表
 * 主要用于睡眠
 */
private final SleepingStopwatch stopwatch;

// 排他锁Object
private volatile @Nullable Object mutexDoNotUseDirectly;
```

### 排他锁对象

```java
//就是一个双端检测的单例对象
private Object mutex() {
  Object mutex = mutexDoNotUseDirectly;
  if (mutex == null) {
    synchronized (this) {
      mutex = mutexDoNotUseDirectly;
      if (mutex == null) {
        mutexDoNotUseDirectly = mutex = new Object();
      }
    }
  }
  return mutex;
}
```

### 休眠秒表

> 在多任务进来之后，会对任务进行「排序」，每个任务什么时候执行都做了规划。
>
> 需要通过睡眠秒表进行线程休眠。
>
> 为啥封装这一层呢。包含了对系统时间的读取，还有对线程打断异常的捕获处理。

```java
abstract static class SleepingStopwatch {
  /** Constructor for use by subclasses. */
  protected SleepingStopwatch() {}

  //读取当前系统时间的微秒数。
  protected abstract long readMicros();

  //进行不可打断的休眠
  protected abstract void sleepMicrosUninterruptibly(long micros);

  public static SleepingStopwatch createFromSystemTimer() {
    //创建匿名内部类进行子类实现
    return new SleepingStopwatch() {
      //开始计时，记录当前系统时间
      final Stopwatch stopwatch = Stopwatch.createStarted();

      @Override
      protected long readMicros() {
        //读取到现在为止的时间差
        return stopwatch.elapsed(MICROSECONDS);
      }

      @Override
      protected void sleepMicrosUninterruptibly(long micros) {
        if (micros > 0) {
          //对当前线程进行不可打断的休眠
          Uninterruptibles.sleepUninterruptibly(micros, MICROSECONDS);
        }
      }
    };
  }
}
```

#### Stopwatch

> 注意。这个Stopwatch不是spring.utils提供的那个秒表「多个任务记录，可以对运行时间进行比对。」
>
> 这个Stopwatch的操作有「开始计时、停、读取时间差」

```java
public final class Stopwatch {
  //时钟
  private final Ticker ticker;
  //是否在计时
  private boolean isRunning;
  //到stop为止的时间流逝差
  private long elapsedNanos;
  //开始时间
  private long startTick;

  //需要手动开始的构造方法
  public static Stopwatch createUnstarted() {
    return new Stopwatch();
  }

 	//创建就开始
  public static Stopwatch createStarted() {
    return new Stopwatch().start();
  }

  Stopwatch() {
    //使用Ticker获取系统时间
    this.ticker = Ticker.systemTicker();
  }
  
	//……重载的构造方法 略……

  //表是否在运行
  public boolean isRunning() {
    return isRunning;
  }

  //开始
  @CanIgnoreReturnValue
  public Stopwatch start() {
    checkState(!isRunning, "This stopwatch is already running.");
    isRunning = true;
    startTick = ticker.read();
    return this;
  }

  //停
  @CanIgnoreReturnValue
  public Stopwatch stop() {
    long tick = ticker.read();
    checkState(isRunning, "This stopwatch is already stopped.");
    isRunning = false;
    elapsedNanos += tick - startTick;
    return this;
  }

  //重置秒表
  @CanIgnoreReturnValue
  public Stopwatch reset() {
    elapsedNanos = 0;
    isRunning = false;
    return this;
  }

  private long elapsedNanos() {
    //如果stop了，就返回当时stop的时间
    //如果没有stop，就使用当前时间计算差值。
    return isRunning ? ticker.read() - startTick + elapsedNanos : elapsedNanos;
  }

  //读取时间流逝的差值
  public long elapsed(TimeUnit desiredUnit) {
    return desiredUnit.convert(elapsedNanos(), NANOSECONDS);
  }

  //……elapsed方法重载 略……

  //……toString()略……
}
```

##### 系统时钟

```java
public abstract class Ticker {
  protected Ticker() {}

  public abstract long read();

  //使用静态内部类实现单例
  public static Ticker systemTicker() {
    return SYSTEM_TICKER;
  }

  private static final Ticker SYSTEM_TICKER =
      new Ticker() {
        @Override
        public long read() {
          //System.nanoTime()读取系统当前的纳秒数
          return Platform.systemNanoTime();
        }
      };
}
```

#### Uninterruptibles.sleepUninterruptibly

> 发生了线程打断异常，依然等那么长时间「坑已经留好了」。
>
> 两个线程A,B，之前就设置好了A 100ms后执行，B 200ms后执行。如果此时发生A线程的InterruptedException，B仍然在200ms后执行，A线程也依旧要等到100ms时间到执行Thread.currentThread().interrupt()

```java
public static void sleepUninterruptibly(long sleepFor, TimeUnit unit) {
  boolean interrupted = false;
  try {
    long remainingNanos = unit.toNanos(sleepFor);
    long end = System.nanoTime() + remainingNanos;
    while (true) {
      try {
        // TimeUnit.sleep() treats negative timeouts just like zero.
        NANOSECONDS.sleep(remainingNanos);
        return;
      } catch (InterruptedException e) {
        //休眠期间被打断 进入下一轮循环继续等待
        interrupted = true;
        remainingNanos = end - System.nanoTime();
      }
    }
  } finally {
    //发生了线程中断异常
    if (interrupted) {
      //执行线程中断
      Thread.currentThread().interrupt();
    }
  }
}
```

## 构造方法

```java
//提供了一个供子类调用的指定秒表的单参构造器
RateLimiter(SleepingStopwatch stopwatch) {
  this.stopwatch = checkNotNull(stopwatch);
}
```

提供了统一的子类对象创建入口。

> 默认不指定预热时间的突发限流器

```java
public static RateLimiter create(double permitsPerSecond) {
  //默认设置1s限流速度
  return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
}

//不对外暴露
@VisibleForTesting
static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
  //默认创建一个突发限流器
  RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
  rateLimiter.setRate(permitsPerSecond);
  return rateLimiter;
}

```
> 指定了预热时间，就变成了预热限流器。

```java
public static RateLimiter create(double permitsPerSecond, Duration warmupPeriod) {
  //指定预热时间
  return create(permitsPerSecond, toNanosSaturated(warmupPeriod), TimeUnit.NANOSECONDS);
}


@SuppressWarnings("GoodTime") // should accept a java.time.Duration
public static RateLimiter create(double permitsPerSecond, long warmupPeriod, TimeUnit unit) {
  checkArgument(warmupPeriod >= 0, "warmupPeriod must not be negative: %s", warmupPeriod);
  return create(
    	//指定了一个默认的coldFactor 3.0
      permitsPerSecond, warmupPeriod, unit, 3.0, SleepingStopwatch.createFromSystemTimer());
}

//不对外暴露
@VisibleForTesting
static RateLimiter create(
    double permitsPerSecond,
    long warmupPeriod,
    TimeUnit unit,
    double coldFactor,
    SleepingStopwatch stopwatch) {
  //创建预热限流器
  RateLimiter rateLimiter = new SmoothWarmingUp(stopwatch, warmupPeriod, unit, coldFactor);
  rateLimiter.setRate(permitsPerSecond);
  return rateLimiter;
}
```

> 这里简单介绍一下「预热」和「突发」两种限流方案的区别。
>
> 首先，在**请求平稳到来的稳定期**，两种限流方案一样：
>
> - 都是把QPS平分到1秒中之内，排队等待请求被运行。
>
> 但是，在**突发流量时，或者系统刚刚启动，资源没有加载完，需要缓冲的时候**，两种限流方案在这里有区别：
>
> 『且。这里有一个前提：在最初令牌桶都是满的，这一部分叫做**storedPermits**』。
>
> - 「突发限流」这些storedPermits，不需要等待时间，>0就可以零等待执行。
>
>   ==突发限流器是令牌桶思想。==
>
> - 「预热限流」==是漏桶限流思想==「你别管我桶里有多少令牌，我只对获取令牌的速度做限制」。
>
>   最初的一个半桶的令牌获取是预热期「预热时间用户指定」，速度越来越快。
>
>   后一个半桶的令牌获取，按照突发限流的稳定期「也就是视storedPermits=0」，获取令牌总是按照QPS评分到1秒的时间等待。
>
> 暂介绍到这里，我们先看共同的抽象部分。
>
> ![image-20210720155106751](https://gitee.com/wangigor/typora-images/raw/master/image-20210720155106751.png)

## 主要方法

### 设置限流速度

> 设置速度我们看到在前面的构造方法中创建完后调用。
>
> 先看RateLimiter中的实现

```java
//RateLimiter.java

//设置速度
public final void setRate(double permitsPerSecond) {
  checkArgument(
      permitsPerSecond > 0.0 && !Double.isNaN(permitsPerSecond), "rate must be positive");
  //注意，这是一个统一的排它锁
  synchronized (mutex()) {
    doSetRate(permitsPerSecond, stopwatch.readMicros());
  }
}

abstract void doSetRate(double permitsPerSecond, long nowMicros);

//获取速率
public final double getRate() {
  //注意，这是一个统一的排它锁
  synchronized (mutex()) {
    return doGetRate();
  }
}

abstract double doGetRate();
```

子类实现

```java
//SmoothRateLimiter.java

/**
 * 设置速度
 * @param permitsPerSecond
 * @param nowMicros 从创建开始到调用的间隔时间
 */
@Override
final void doSetRate(double permitsPerSecond, long nowMicros) {
    //先设置从开始到此时的应该产生的令牌数和重置任务排期时间
    resync(nowMicros);
    //每个令牌插入的稳定时间间隔
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
    this.stableIntervalMicros = stableIntervalMicros;
  	//继续一个供子类实现的设置速度方法
    doSetRate(permitsPerSecond, stableIntervalMicros);
}

//这里先放一放，等到子类的时候再说
abstract void doSetRate(double permitsPerSecond, double stableIntervalMicros);

/**
 * 获取速度
 */
@Override
final double doGetRate() {
  	//时间间隔不允许设置，都是1s
    return SECONDS.toMicros(1L) / stableIntervalMicros;
}
```

> 这里注意，==限流周期都是1s==。



这里设置令牌，不是通过一个线程间隔固定时间的方式防止令牌，而是通过秒表的当前时间进行计算出来的可使用令牌数。有一个同步方法。

```java
/**
 * Updates {@code storedPermits} and {@code nextFreeTicketMicros} based on the current time.
 */
void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
        //计算从调用时间往前一直到上一个任务「执行完成」的这一段时间内，应该产生多少令牌
        double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
        //不越界的加到未消耗令牌里
        storedPermits = min(maxPermits, storedPermits + newPermits);
        //下一个请求，从此刻开始「排期」
        nextFreeTicketMicros = nowMicros;
    }
}
//供子类实现
abstract double coolDownIntervalMicros();
```

### 获取令牌

> 这里的获取令牌是不可中断。
>
> 要么当时就执行，要么休眠到指定时间执行。
>
> 如果需求是当时就执行，可以接受1s内执行，1秒内执行不了就进队列或者退回，需要用boolean tryAcquire方法

```java
//RateLimiter.java

@CanIgnoreReturnValue
public double acquire() {
  //获取令牌 默认1
  return acquire(1);
}

@CanIgnoreReturnValue
public double acquire(int permits) {
  //计算获取当前令牌需要等待的时间
  long microsToWait = reserve(permits);
  //线程等待
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  //用于向微秒整形
  //返回的是等待时间
  return 1.0 * microsToWait / SECONDS.toMicros(1L);
}

/**
 * Reserves the given number of permits from this {@code RateLimiter} for future use, returning
 * the number of microseconds until the reservation can be consumed.
 *
 * @return time in microseconds to wait until the resource can be acquired, never negative
 */
final long reserve(int permits) {
  //对获取令牌数量进行检查
  // >0就行
  checkPermits(permits);
  //使用了同一个排它锁。
  synchronized (mutex()) {
    //计算等待时间
    return reserveAndGetWaitLength(permits, stopwatch.readMicros());
  }
}
```

### tryAcquire

```java
//RateLimiter.java

//带超时时间的等待
public boolean tryAcquire(Duration timeout) {
  return tryAcquire(1, toNanosSaturated(timeout), TimeUnit.NANOSECONDS);
}

//重载方法
@SuppressWarnings("GoodTime") // should accept a java.time.Duration
public boolean tryAcquire(long timeout, TimeUnit unit) {
  return tryAcquire(1, timeout, unit);
}

//重载方法
public boolean tryAcquire(int permits) {
  return tryAcquire(permits, 0, MICROSECONDS);
}

//默认方法
//不等待 
public boolean tryAcquire() {
  return tryAcquire(1, 0, MICROSECONDS);
}
//重载方法
public boolean tryAcquire(int permits, Duration timeout) {
  return tryAcquire(permits, toNanosSaturated(timeout), TimeUnit.NANOSECONDS);
}

//带有超时退出的尝试获取
//通过计算等待时间，如果在可接受范围timeout范围之内就排队执行；如果超过timeout，立即返回false。
@SuppressWarnings("GoodTime") // should accept a java.time.Duration
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
  long timeoutMicros = max(unit.toMicros(timeout), 0);
  checkPermits(permits);
  long microsToWait;
  synchronized (mutex()) {
    long nowMicros = stopwatch.readMicros();
    //尝试获取 获取失败就返回
    if (!canAcquire(nowMicros, timeoutMicros)) {
      return false;
    } else {
      //可以等待，就计算延迟时间
      microsToWait = reserveAndGetWaitLength(permits, nowMicros);
    }
  }
  //线程休眠并执行
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return true;
}

private boolean canAcquire(long nowMicros, long timeoutMicros) {
  //已排序任务的结束时间-超时时间 在当前时间之前
  //也就是当前任务可以在超时时间之内完成等待。
  return queryEarliestAvailable(nowMicros) - timeoutMicros <= nowMicros;
}
```

注意这里的默认``` boolean tryAcquire()```方法，不等待，如果使用这里的返回值进行判断，10个请求同时进来，至多有一个能运行。

> 可能在本地测试的时候会有这样的问题。

### 计算等待时间

```java
//RateLimiter.java

final long reserveAndGetWaitLength(int permits, long nowMicros) {
  //获取等待时间
  long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
  //不可以分配在过去的时间执行，最起码也是now「0」
  return max(momentAvailable - nowMicros, 0);
}

//供子类实现的计算方法
abstract long reserveEarliestAvailable(int permits, long nowMicros);
```

```java
//SmoothRateLimiter.java

/**
 * 方法执行都被上级synchronized包裹。不需要考虑并发问题。
 */
@Override
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    //同步一下令牌数和开始排序的时间
    resync(nowMicros);
    //初始返回值，从上一个任务分配出去的结束时间开始
    long returnValue = nextFreeTicketMicros;
    //优先使用未使用的令牌
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    //已存储的令牌不够用了，在使用恒定的添加令牌
    double freshPermits = requiredPermits - storedPermitsToSpend;
    long waitMicros =
            //未使用的令牌使用速度，看子类实现
            storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
                    + (long) (freshPermits * stableIntervalMicros);

    //更新当前任务的下一个分配结束时间
    //采用封装过的防止溢出越界等的long型相加
    this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
    //扣减令牌存量
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
}
```

### 最早可执行时间

> 可能暂时没有排期，或者排期已经到了之后。
>
> 那么RateLimiter里有一个获取排期的方法。
>
> 注意：**这个时间不是上一个任务的放行时间，而是下一个任务的可放行时间阈值。**啥意思呢，初始时间0，假设1秒10QPS「100ms放行一个」，那放行第一个任务「在0ms执行」之后，这个时间变成100ms。

```java
//RateLimiter.java

abstract long queryEarliestAvailable(long nowMicros);
```

```java
//SmoothRateLimiter.java
@Override
final long queryEarliestAvailable(long nowMicros) {
    return nextFreeTicketMicros;
}
```



## 突发和预热

### 突发限流

> 令牌桶思想。
>
> 如果有空闲的未使用令牌，毫无保留的使用。

```java
static final class SmoothBursty extends SmoothRateLimiter {

    /**
     * The work (permits) of how many seconds can be saved up if this RateLimiter is unused?
     */
    //多少秒「默认是1，不对外开放指定/修改」
    final double maxBurstSeconds;

    SmoothBursty(SleepingStopwatch stopwatch, double maxBurstSeconds) {
        super(stopwatch);
        this.maxBurstSeconds = maxBurstSeconds;
    }

    @Override
    void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
        double oldMaxPermits = this.maxPermits;
        //修改桶令牌总量
        maxPermits = maxBurstSeconds * permitsPerSecond;

        if (oldMaxPermits == Double.POSITIVE_INFINITY) {
            // if we don't special-case this, we would get storedPermits == NaN, below
            storedPermits = maxPermits;
        } else {
            //已存未消耗的令牌 按照比例扣减
            storedPermits =
                    (oldMaxPermits == 0.0)
                            ? 0.0 // initial state
                            : storedPermits * maxPermits / oldMaxPermits;
        }
    }

    @Override
    long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
        //已存的令牌消耗-直接消耗，不做延迟。
        return 0L;
    }

    @Override
    double coolDownIntervalMicros() {
        return stableIntervalMicros;
    }
}
```

### 预热限流

> 漏桶思想。
>
> 先上代码，如果看不懂可以先看下面的解释。

```java
static final class SmoothWarmingUp extends SmoothRateLimiter {
    private final long warmupPeriodMicros;
    /**
     * The slope of the line from the stable interval (when permits == 0), to the cold interval
     * (when permits == maxPermits)
     */
    private double slope;

    private double thresholdPermits;
    private double coldFactor;

    SmoothWarmingUp(
            SleepingStopwatch stopwatch, long warmupPeriod, TimeUnit timeUnit, double coldFactor) {
        super(stopwatch);
        this.warmupPeriodMicros = timeUnit.toMicros(warmupPeriod);
        this.coldFactor = coldFactor;
    }

    /**
     * @param permitsPerSecond 总令牌数
     * @param stableIntervalMicros 稳定期stableIntervalMicros秒放入一个令牌。
     */
    @Override
    void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
        //原速率最大值。
        double oldMaxPermits = maxPermits;
        //预热期 token令牌放置时间要乘以coldFactor
        double coldIntervalMicros = stableIntervalMicros * coldFactor;
        //预热期的token令牌最小要达到 预热时间里能够产生的令牌数的一半
        thresholdPermits = 0.5 * warmupPeriodMicros / stableIntervalMicros;
        //最大令牌数 根据梯形面积=warmupPeriod计算
        maxPermits =
                thresholdPermits + 2.0 * warmupPeriodMicros / (stableIntervalMicros + coldIntervalMicros);
        //上图梯形斜边的斜率「后续用于计算右侧使用时间」
        slope = (coldIntervalMicros - stableIntervalMicros) / (maxPermits - thresholdPermits);
        if (oldMaxPermits == Double.POSITIVE_INFINITY) {
            // if we don't special-case this, we would get storedPermits == NaN, below
            storedPermits = 0.0;
        } else {
            storedPermits =
                    (oldMaxPermits == 0.0)
                            //初始化的时候，桶令牌是满的
                            ? maxPermits // initial state is cold
                            : storedPermits * maxPermits / oldMaxPermits;
        }
    }

    /**
     * 计算消耗permitsToTake个令牌需要多少时间
     *
     * 左右两个部分要分别计算
     *  - 左侧消耗的时间=左边消耗的令牌数*stableInterval
     *  - 右侧消耗的时间=从permitsToTake扣减到thresholdPermits的梯形面积
     *
     * @param storedPermits 已存令牌数量
     * @param permitsToTake 请求消耗令牌数量
     * @return
     */
    @Override
    long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {

        double availablePermitsAboveThreshold = storedPermits - thresholdPermits;
        long micros = 0;
        //右侧
        // measuring the integral on the right part of the function (the climbing line)
        if (availablePermitsAboveThreshold > 0.0) {
            double permitsAboveThresholdToTake = min(availablePermitsAboveThreshold, permitsToTake);
            // TODO(cpovirk): Figure out a good name for this variable.
            double length =
                    permitsToTime(availablePermitsAboveThreshold)
                            + permitsToTime(availablePermitsAboveThreshold - permitsAboveThresholdToTake);
            micros = (long) (permitsAboveThresholdToTake * length / 2.0);
            permitsToTake -= permitsAboveThresholdToTake;
        }
        //剩余的左侧面积
        // measuring the integral on the left part of the function (the horizontal line)
        micros += (long) (stableIntervalMicros * permitsToTake);
        return micros;
    }

    //计算当前permits到xxxInterval的梯形边的转换
    private double permitsToTime(double permits) {
        return stableIntervalMicros + permits * slope;
    }

    @Override
    double coolDownIntervalMicros() {
        return warmupPeriodMicros / maxPermits;
    }
}
```



假设建立一个「已存储令牌数量storedPermits」和「令牌获取时间间隔interval」的坐标。

- 稳定期的时间速度就是1s/QPS。在坐标轴上是一条水平的直线。

<img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210720163702971.png" alt="image-20210720163702971" style="zoom:50%;" />

中间夹的这个区域就是消耗掉所有storedPermits需要的时间「1s」。

预热要做的事情是，延缓一部分「最初的一部分」的令牌分配速度。也就是越靠近maxPermits，interval越大「焦点的纵坐标值就是coldInterval」。

> RateLimiter增加了一个前提，就是默认这个预热是前半个周期。

![image-20210720165623968](https://gitee.com/wangigor/typora-images/raw/master/image-20210720165623968.png)

- 预热限流指定了预热时间warmPeriod，也就是图中的粉红色区域。

- 默认指定了第一个令牌「产生时间」是coldInterval = **coldFactor** * stableInterval。**coldFactor默认3**.

- 这就固定了绿色部分的大小「stableInterval * thresholdPermits」是红色部分大小「4 * stableInterval * (maxPermits - thresholdPermits) / 2」的一半。

- 那么**thresholdPermits = 1/2 * warmPeriod / stableInterval**

- 粉色梯形面积warmPeriod= ( coldInteval + stableInterval ) * ( maxPermits - thresholdPermits ) / 2

  **maxPermits= 2 * warmPeriod / ( coldInterval - stableInterval )**

  其实就跟设置的permits一样。

- 假设需要获取从thresholdPermits到maxPermits中的一部分令牌怎么办呢？

  需要知道这是第几块令牌，而这块令牌的interval是多少。

  做一条平行于粉色提醒斜边的过0点的线计算斜率slope，就能帮助以后直接获取interval。

  **slope = (coldIntervalMicros - stableIntervalMicros) / (maxPermits - thresholdPermits)**

  后续获取**interval = stableIntervalMicros + permits * slope;**

