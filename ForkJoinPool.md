# ForkJoinPool

## 背景知识

### fork-join「分支与合并」

> ![img](https://gitee.com/wangigor/typora-images/raw/master/fork-join.png)
>
> 案例：计算10亿个数字相加。四种方式：单线程、线程池、forkjoin、ForkJoinPool「java8」.

```java
/**
 * @author wangke
 */
@Slf4j
public class ForkJoinTest {


    @Test
    public void TimeTaskTest() throws ExecutionException, InterruptedException {

        StopWatch watch = new StopWatch();

        //初始化数据
        long start = 0L;
        long end = 1000000000L;

        singleThreadSum(start, end, watch);
        multiThreadGroupSum(start, end, watch);
        forkJoinSum(start, end, watch);
        java8Lambda(start, end, watch);

        log.info(watch.prettyPrint());
    }

    /**
     * 单线程求和
     *
     * @param start
     * @param end
     * @param watch
     * @return
     */
    private long singleThreadSum(long start, long end, StopWatch watch) {
        watch.start("singleThreadSum");

        long sum = 0L;
        for (long i = start; i <= end; i++) {
            sum += activeDelay(i);
        }
        watch.stop();
        log.info("singleThreadSum result:{}", sum);
        return sum;
    }

    /**
     * 多线程分组求和
     *
     * @param start
     * @param end
     * @param watch
     * @return
     */
    private long multiThreadGroupSum(long start, long end, StopWatch watch) throws ExecutionException, InterruptedException {
        watch.start("multiThreadGroupSum");

        int groupNum = 100;

        ExecutorService poolExecutor = new ThreadPoolExecutor(
                3, 3,
                5, TimeUnit.MINUTES,
                new ArrayBlockingQueue<>(50),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.CallerRunsPolicy());

        long groupBy = (end - start) / groupNum;

        List<Future> futures = new ArrayList<>(groupNum);
        //fork
        for (int group = 1; group <= groupNum; group++) {
            final int group_final = group;
            Future<Long> future = poolExecutor.submit(() -> {
                long start_tmp = (group_final - 1) * groupBy + 1;
                long end_temp = group_final * groupBy;

                long sum_group = 0L;

                for (long i = start_tmp; i <= end_temp; i++) {
                    sum_group += activeDelay(i);
                }
                return sum_group;
            });
            futures.add(future);
        }

        long sum = 0L;

        //join
        for (Future<Long> future : futures) {
            sum += future.get();
        }
        poolExecutor.shutdown();
        watch.stop();
        log.info("multiThreadGroupSum result:{}", sum);

        return sum;
    }

    /**
     * java LongStream 底层是ForkJoinPool
     *
     * @param start
     * @param end
     * @param watch
     * @return
     */
    private long java8Lambda(long start, long end, StopWatch watch) {
        watch.start("java8Lambda");

        long sum = LongStream.rangeClosed(start, end)
                .parallel()
                .reduce((sum_temp, item) -> {
                    sum_temp += activeDelay(item);
                    return sum_temp;
                }).getAsLong();

        watch.stop();
        log.info("java8Lambda result:{}", sum);
        return sum;
    }

    private long forkJoinSum(long start, long end, StopWatch watch) throws ExecutionException, InterruptedException {
        watch.start("forkJoinSum");

        ForkJoinPool forkJoinPool = ForkJoinPool.commonPool();

        ForkJoinTask<Long> joinTask = forkJoinPool.submit(new SumTask(start, end));
        Long sum = joinTask.get();

        forkJoinPool.shutdown();

        watch.stop();
        log.info("forkJoinSum result:{}", sum);
        return sum;
    }

    @Data
    @RequiredArgsConstructor
    class SumTask extends RecursiveTask<Long> {

        /**
         * 拆分阈值
         */
        static final int threshold = 100;

        private @NonNull long from;
        private @NonNull long to;

        @Override
        protected Long compute() {


            //小于阈值，直接当前线程计算。
            if (to - from <= threshold) {
                long sum = 0;
                for (long i = from + 1; i <= to; i++) {
                    sum += activeDelay(i);
                }
                return sum;
            }

            //拆分成「两个任务」

            long middle = (from + to) / 2;
            SumTask task0 = new SumTask(from, middle);
            SumTask task1 = new SumTask(middle, to);

            task0.fork();//提交新任务
            Long result1 = task1.compute();//使用当前线程执行，防止浪费。

            //join
            Long result0 = task0.join();

            return result1 + result0;
        }
    }


    /**
     * 手动延迟 尽量减少「forkjoin的拆分耗时」 对「执行耗时」的影响
     * 因为在生产环境中，这样的「累加操作」不是常规业务。
     *
     * @param a
     * @return
     */
    private long activeDelay(long a) {
        return a * 7 / 7 * 7 / 7 * 7 / 7 * 7 / 7 * 7 / 7 * 7 / 7 * 7 / 7 * 7 / 7 * 7 / 7 * 7 / 7;
    }
}
```

执行结果

```log
/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/bin/java 

[main] INFO com.test.forkjoin.ForkJoinTest - singleThreadSum result:500000000500000000
[main] INFO com.test.forkjoin.ForkJoinTest - multiThreadGroupSum result:500000000500000000
[main] INFO com.test.forkjoin.ForkJoinTest - forkJoinSum result:500000000500000000
[main] INFO com.test.forkjoin.ForkJoinTest - java8Lambda result:500000000500000000
[main] INFO com.test.forkjoin.ForkJoinTest - StopWatch '': running time = 19429873306 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
12191121840  063%  singleThreadSum
3648896129  019%  multiThreadGroupSum
1904817962  010%  forkJoinSum
1685037375  009%  java8Lambda

Process finished with exit code 0
```



### work-stealing「工作窃取」

![img](https://gitee.com/wangigor/typora-images/raw/master/work-stealing.jpg)

> 可以理解为：线程池中的每个线程都对应一个任务双端队列。自己工作做完了，去别人那里取。

### Treiber stack 非阻塞并发栈

> 对栈的并发操作，最简单的方式是加锁。但是**加锁会导致性能低下，且会阻塞其他线程。**
>
> - 使用CAS保证操作原子性。
> - 允许线程饥饿

```java
/**
 * Treiber stack 非阻塞并发栈
 *
 * @author wangke
 */
@Slf4j
public class ConcurrentStack<E> {

    /**
     * 头 「栈顶」
     */
    private volatile Node<E> head;

    private static Unsafe UNSAFE;
    private static long HEAD;


    static {
        try {
            Field getUnsafe = sun.misc.Unsafe.class.getDeclaredField("theUnsafe");
            getUnsafe.setAccessible(true);
            UNSAFE = (Unsafe) getUnsafe.get(null);

            HEAD = UNSAFE.objectFieldOffset(ConcurrentStack.class.getDeclaredField("head"));

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 入栈
     *
     * @param data
     */
    public void push(E data) {
        Node oldHead;
        Node<E> newHead = new Node<>(data);
        int count = 0;

        do {
            oldHead = head;
            count++;
        } while (!UNSAFE.compareAndSwapObject(this, HEAD, oldHead, newHead));

        newHead.setNext(oldHead);
        log.info("{}:{}对象入栈重试了{}次...", System.currentTimeMillis(), data, count);
    }

    /**
     * 出栈
     *
     * @return
     */
    public E poll() {

        Node<E> oldHead;
        Node<E> newHead;

        int count = 0;
        do {

            oldHead = head;
            newHead = oldHead.getNext();
            count++;

        } while (!UNSAFE.compareAndSwapObject(this, HEAD, oldHead, newHead));

        oldHead.setNext(null);
        log.info("{}:出栈重试了{}次|{}", System.currentTimeMillis(), count, oldHead.getData());
        return oldHead.getData();
    }

    /**
     * 测试
     * @param args
     * @throws InterruptedException
     */
    public static void main(String[] args) throws InterruptedException {
        ConcurrentStack<Integer> stack = new ConcurrentStack<>();


        IntStream.range(0, 100).parallel().forEach(
                item -> stack.push(new Random().nextInt(Integer.MAX_VALUE))
        );

        IntStream.range(0, 100).parallel().forEach(
                item -> log.info(stack.poll() + "")
        );

        new CountDownLatch(1).await();

    }
}

/**
 * 节点
 *
 * @param <E>
 */
@Data
class Node<E> {
    @NonNull
    private E data;
    private Node<E> next;
}
```



## 源码

> ForkJoinPool源码的位运算是真！的！多！

### 初始化

> 先看一下这四个重载的构造方法。

```java
    //无参
		public ForkJoinPool() {
        this(Math.min(MAX_CAP, Runtime.getRuntime().availableProcessors()),
             defaultForkJoinWorkerThreadFactory, null, false);
    }

		/**
		 *单参构造
		 * parallelism 并行数
		 */
    public ForkJoinPool(int parallelism) {
        this(parallelism, defaultForkJoinWorkerThreadFactory, null, false);
    }

		/**
		 *单参构造
		 * parallelism 并行数
		 * factory： 创建工作线程的工厂
     * handler： 工作线程异常回调处理
     * asyncMode： 异步模式：true表示采用FIFO模式；false表示采用LIFO模式；
		 */
    public ForkJoinPool(int parallelism,
                        ForkJoinWorkerThreadFactory factory,
                        UncaughtExceptionHandler handler,
                        boolean asyncMode) {
        this(checkParallelism(parallelism),
             checkFactory(factory),
             handler,
             asyncMode ? FIFO_QUEUE : LIFO_QUEUE,
             "ForkJoinPool-" + nextPoolId() + "-worker-");
        checkPermission();
    }
		
    private ForkJoinPool(int parallelism,
                         ForkJoinWorkerThreadFactory factory,
                         UncaughtExceptionHandler handler,
                         int mode,
                         String workerNamePrefix) {
        this.workerNamePrefix = workerNamePrefix;
        this.factory = factory;
        this.ueh = handler;
        this.config = (parallelism & SMASK) | mode;
        long np = (long)(-parallelism); // offset ctl counts
        this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);
    }
```

下面详细讲明config、np、ctl三个参数。

#### config

> ```java
> static final int SMASK        = 0xffff;        // 65535 
> //config是记录了并发量parallelism和异步模式的int
> // & SMASK 是为了防止并发量超过65535 其实构造方法里有对parallelism的校验
> //private static int checkParallelism(int parallelism) {
> //    if (parallelism <= 0 || parallelism > MAX_CAP) 不大于32767 也就保证只占int的低15位
> //        throw new IllegalArgumentException();
> //    return parallelism;
> //}
> this.config = (parallelism & SMASK) | mode;
> 
> static final int MODE_MASK    = 0xffff << 16;  //用于获取 config中的mode 11111111 11111111 00000000 00000000
> static final int LIFO_QUEUE   = 0;     
> static final int FIFO_QUEUE   = 1 << 16; //第17位是1 00000000 00000001 00000000 00000000
> static final int SHARED_QUEUE = 1 << 31;// 最高位是1 10000000 00000000 00000000 00000000
> ```
>
> ```log
> +-----------------------------+
> | 高16位：模式 | | 低15位：并发量 |
> +-----------------------------+
> ```

#### np

> ```java
> long np = (long)(-parallelism);
> //np是并行度parallelism的 负数的补码存储。转为64位long。
> //并行度为1时：
> //np = (long)-1 也就是 1111111111111111 1111111111111111 1111111111111111 1111111111111111
> //并行度为MAX_CAP时：
> //np = (long)-0x7fff  1111111111111111 1111111111111111 1111111111111111 1000000000000001
> ```

#### ctl

> 源码解释
> Bits and masks for field ctl, packed with 4 16 bit subfields:     // ctl字段的位和掩码，包含了4个16位子段
> AC: Number of active running workers minus target parallelism    // AC : 活跃的运行工作线程数 减去 目标并行度
> TC: Number of total workers minus target parallelism	// TC : 总工作线程数 减去 目标并行度
> SS: version count and status of top waiting thread	// SS : 版本计数 和 等待线程的状态
> ID: poolIndex of top of Treiber stack of waiters	// ID : 空闲线程等待队列「Treiber stack of waiters」的栈顶等待线程在线程池中的下标。
>
> When convenient, we can extract the lower 32 stack top bits (including version bits) as sp=(int)ctl.  
> The offsets of countsby the target parallelism and the positionings of fields makes it possible to perform the most common checks via sign tests of fields: 
> When ac is negative, there are not enough active workers // 当ac为负数时：没有足够的活跃worker
> when tc is negative, there are not enough total workers // 当tc为负数时：没有足够的worker
> When sp is non-zero, there are waiting workers //当sp非零，说明有空闲worker
> To deal with possibly negative fields, we use casts in and out of "short" and/or signed shifts to maintain signedness.
> Because it occupies uppermost bits, we can add one active count using getAndAddLong of AC_UNIT, rather than CAS, when returning from a blocked join. 
> Other updates entail multiple subfields and masking, requiring CAS.

> ```java
> this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);
> ```
> ```java
> // Active counts 活跃线程数
> private static final int  AC_SHIFT   = 48; //左移次数 也就是long的高16位 49-64
> private static final long AC_UNIT    = 0x0001L << AC_SHIFT; // 0000000000000001 0000000000000000 0000000000000000 0000000000000000
> private static final long AC_MASK    = 0xffffL << AC_SHIFT; // 1111111111111111 0000000000000000 0000000000000000 0000000000000000
> 
> // Total counts 总线程数
> private static final int  TC_SHIFT   = 32; //左移次数 也就是long的次16位 33-48
> private static final long TC_UNIT    = 0x0001L << TC_SHIFT; // 0000000000000000 0000000000000001 0000000000000000 0000000000000000
> private static final long TC_MASK    = 0xffffL << TC_SHIFT; // 0000000000000000 1111111111111111 0000000000000000 0000000000000000
> private static final long ADD_WORKER = 0x0001L << (TC_SHIFT + 15); // sign 0000000000000000 1000000000000000 0000000000000000 0000000000000000
> ```
>
> 目前：
>
> ctl的高32位，存了两份np
>
> parallelism=1时，ctl：1111111111111111 1111111111111111 0000000000000000 0000000000000000
>
> parallelism=MAX_CAP时，ctl：1000000000000001 1000000000000001 0000000000000000 0000000000000000

### 提交任务

> 有三类提交方法：invoke、execute、submit。
>
> ```java
>     public <T> T invoke(ForkJoinTask<T> task) {
>         if (task == null)
>             throw new NullPointerException();
>         externalPush(task);
>         return task.join();
>     }
> 
>     public void execute(ForkJoinTask<?> task) {
>         if (task == null)
>             throw new NullPointerException();
>         externalPush(task);
>     }
> 
>     public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {
>         if (task == null)
>             throw new NullPointerException();
>         externalPush(task);
>         return task;
>     }
> ```
>
> 差别：
>
> - invoke。要等到task执行完成，返回执行结果。
> - execute。只是提交任务，任务后续返回等待由调用线程处理。
> - submit。跟execute类似，提交任务，返回task。

最后都调用了**externalPush**，将任务添加到线程池的「当前线程等待队列」中。

```java
		//这里「不负责真实的添加任务」。主要还是线程池的初始化。
    final void externalPush(ForkJoinTask<?> task) {
      	//WorkQueue[] ws: 等待队列组
      	//WorkQueue q:当前线程的等待队列
      	//int m: ws的最大下标
        WorkQueue[] ws; WorkQueue q; int m;
        int r = ThreadLocalRandom.getProbe(); //线程探针的「标志」。
        int rs = runState; //线程池的运行状态 初始为0
      
      	//尝试任务添加
      	//①等待队列组 ws 不为空
      	//②等待队列组 ws 长度大于0
      	//③当前线程的等待队列已经初始化：「m & r & SQMASK」：计算当前线程对应的「槽位」：「m & r」不超过队列总长，且『& SQMASK （0x007e）0111 1110』小于126的偶数槽位。
      	//④当前线程探针已经初始化
      	//⑤线程池已经被初始化过
      	//⑥获取锁成功:对当前队列的QLOCK的cas
        if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&
            (q = ws[m & r & SQMASK]) != null && r != 0 && rs > 0 &&
            U.compareAndSwapInt(q, QLOCK, 0, 1)) {
            ForkJoinTask<?>[] a; int am, n, s;
          	//队列不为空且，队列没满
            if ((a = q.array) != null &&
                (am = a.length - 1) > (n = (s = q.top) - q.base)) {
              	//获取新任务在ForkJoinTask<?>[]中的偏移量
                int j = ((am & s) << ASHIFT) + ABASE;
                U.putOrderedObject(a, j, task);//放置任务
                U.putOrderedInt(q, QTOP, s + 1);// top+1
                U.putIntVolatile(q, QLOCK, 0);//释放锁
                if (n <= 1)
                    signalWork(ws, q);//如果之前的线程没有任务。就手动触发一下。
                return;
            }
            U.compareAndSwapInt(q, QLOCK, 1, 0);//释放锁
        }
      	
      	//未初始化完成或者容量满，走这里初始化和扩容。
        externalSubmit(task);
    }
```

#### ThreadLocalRandom.getProbe()线程探针标记

> 获取Thread的threadLocalRandomProbe参数。是为了对线程进行「标记」，用来hash对应WorkQueue[]的等待队列。

测试：

```java
    @org.junit.Test
    public void testThreadProbe() throws NoSuchFieldException, IllegalAccessException, InterruptedException {

				//获取unsafe
        Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
        theUnsafe.setAccessible(true);
        Unsafe unsafe = (Unsafe) theUnsafe.get(null);

				//获取Thread的类的threadLocalRandomProbe属性偏移量
        long probe = unsafe.objectFieldOffset(
                Thread.class.getDeclaredField("threadLocalRandomProbe")
        );

      	//新建俩线程--------------------------------------------------------
        Thread test = new Thread(() -> {
            ThreadLocalRandom.current();//初始化threadLocalRandomProbe
          	//打印输出
            log.info(
                    "{}|{}",
                    Thread.currentThread().getName(),
                    unsafe.getInt(Thread.currentThread(), probe) + ""
            );
        }, "test");

        Thread test1 = new Thread(() -> {
            ThreadLocalRandom.current();
            log.info(
                    "{}|{}",
                    Thread.currentThread().getName(),
                    unsafe.getInt(Thread.currentThread(), probe) + ""
            );
        }, "test1");
				//----------------------------------------------------------------
      
				//在线程创建完成，且没有初始化threadLocalRandomProbe字段的时候，打印探针值。
        int anInt = unsafe.getInt(test, probe);
        int anInt1 = unsafe.getInt(test1, probe);
        log.info(anInt + "|" + anInt1);
				
      	//启动线程
        test.start();
        test1.start();
				
      	//阻塞等待
        new CountDownLatch(1).await();

    }

```

执行结果：

```log
[main] INFO com.test.forkjoin.Test - 0|0
[test] INFO com.test.forkjoin.Test - test|-1640531527
[test1] INFO com.test.forkjoin.Test - test1|1013904242
```

```java
    //Thread.java
		//线程探针的hash值，初始化都是0.需要手动初始化
		/** Probe hash value; nonzero if threadLocalRandomSeed initialized */
    @sun.misc.Contended("tlr")
    int threadLocalRandomProbe;
```

```java
    //使用unsafe获取当前线程的probe
		//ThreadLocalRandom.getProbe()
		static final int getProbe() {
        return UNSAFE.getInt(Thread.currentThread(), PROBE);
    }
		//初始化probe
		//ThreadLocalRandom.localInit()
		static final void localInit() {
        int p = probeGenerator.addAndGet(PROBE_INCREMENT);
        int probe = (p == 0) ? 1 : p; // skip 0
        long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
        Thread t = Thread.currentThread();
        UNSAFE.putLong(t, SEED, seed);
        UNSAFE.putInt(t, PROBE, probe);
    }
		//封装的初始化方法current
		//ThreadLocalRandom.current()
    public static ThreadLocalRandom current() {
        if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
            localInit();
        return instance;
    }
```

就到这里，把他理解为线程hash标记就好。

***



#### 「线程池初始化」、队列初始化及扩容。

```java
    private void externalSubmit(ForkJoinTask<?> task) {
        int r;//thread probe                       
      	//初始化调用线程的probe探针
        if ((r = ThreadLocalRandom.getProbe()) == 0) {
            ThreadLocalRandom.localInit();
            r = ThreadLocalRandom.getProbe();
        }
        for (;;) {
            WorkQueue[] ws; WorkQueue q; int rs, m, k;
            boolean move = false;
          	//线程池 终止状态
            if ((rs = runState) < 0) {
                tryTerminate(false, false);     // help terminate
                throw new RejectedExecutionException();
            }
          	//线程池初始状态
          	//WorkQueue[] ws为空 或 长度为0
            else if ((rs & STARTED) == 0 ||     // initialize
                     ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {
                int ns = 0;
                rs = lockRunState();//锁线程池
                try {
                  	//相当于双端检测，再判断一次是否为初始化状态。
                  	//int STARTED = 1 << 2 00000000 00000000 00000000 00000100
                    if ((rs & STARTED) == 0) {
                      	//初始化线程池属性「任务窃取计数器」 steal counter
                        U.compareAndSwapObject(this, STEALCOUNTER, null,
                                               new AtomicLong());
                        // create workQueues array with size a power of two
                      	// 根据并发量初始化workQueues数组大小为2的幂
                      	// config 低15位为并发量
                      	// SMASK 0xffff
                        int p = config & SMASK; // ensure at least 2 slots 至少两个槽位。
                      
                      	//先减一个。
                      	//通过五次位运算 |= n >>> 1/2/4/8/16，使低位全部为1
                      	//再加一个，清零低位，<< 1翻倍扩容。
                      	//结果：并行的的进位2次幂的两倍。
                      	//e,g. p=1 进位2次幂为2 ，n最终为4
                      	//e,g. p=MAX_CAP 0x7fff 进位二次幂为0x8000，n最终为65536 2^16
                      	//e,g. p=16或15 进位二次幂都是16，n最终为32
                        int n = (p > 1) ? p - 1 : 1;
                        n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
                        n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1;
                      
                      	//创建WorkQueue[] 等待队列组
                        workQueues = new WorkQueue[n];
                        ns = STARTED; //1 << 2
                    }
                } finally {
                    unlockRunState(rs, (rs & ~RSLOCK) | ns);//解锁
                }
            }//下一次循环走到这里
          	// SQMASK: 0x007e 00000000 00000000 00000000 01111110
          	// r: thread probe
          	// m: ws.length()-1
            else if ((q = ws[k = r & m & SQMASK]) != null) {
              	//锁队列
                if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) {
                    ForkJoinTask<?>[] a = q.array;//任务列表
                    int s = q.top;//index of next slot for push 下一个任务存放位置
                    boolean submitted = false; // initial submission or resizing 是否提交成功标记
                    try {                      // locked version of push
                      	//当前队列不为空，且至少能再存下一个任务
                      	//否则，先扩容
                        if ((a != null && a.length > s + 1 - q.base) ||
                            (a = q.growArray()) != null) {
                          	//在队列中存放新任务
                            int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                            U.putOrderedObject(a, j, task);
                            U.putOrderedInt(q, QTOP, s + 1);
                            submitted = true;//标记为已提交
                        }
                    } finally {
                        U.compareAndSwapInt(q, QLOCK, 1, 0);//解锁
                    }
                    if (submitted) {
                        signalWork(ws, q);//手动触发执行。
                        return;
                    }
                }
                move = true;                   // 任务未提交成功，需要重新提交。
            }
          	//线程池rs未锁定
          	//RSLOCK：1
          	//能进到这个判断，说明需要初始化当前队列。
            else if (((rs = runState) & RSLOCK) == 0) { // create new queue
                q = new WorkQueue(this, null);//初始化队列
                q.hint = r; //thread probe
              	//k: 当前的坐标位置 「thread probe & ws长度 & SQMASK」
              	//SHARED_QUEUE: 1 << 31
                q.config = k | SHARED_QUEUE; //将队列所在「池」的槽位置 存放到队列config中
                q.scanState = INACTIVE;//1 << 31 工作队列状态为未活动，小于0
                rs = lockRunState();           // publish index
                if (rs > 0 &&  (ws = workQueues) != null &&//锁
                    k < ws.length && ws[k] == null)
                    ws[k] = q;                 // else terminated 存放到池中
                unlockRunState(rs, rs & ~RSLOCK); //解锁
            }
            else
                move = true;                   // 需要重复提交
            if (move)	//如果上面都没有提交成功：可能存在多个thread，对应到了一个队列。
                r = ThreadLocalRandom.advanceProbe(r);//重新计算thread probe。
        }
    }
```

##### 位置计算

> ```java
> //a: q.array
> //s: q.top
> int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
> ```
>
> ```java
> Class<?> ak = ForkJoinTask[].class;
> ABASE = U.arrayBaseOffset(ak);//获取ForkJoinTask[]的数组头的「位置」
> int scale = U.arrayIndexScale(ak);//ForkJoinTask的引用元素占用大小
> if ((scale & (scale - 1)) != 0)//scale必须为2的幂
> 	throw new Error("data type scale not a power of two");
> //Integer.numberOfLeadingZeros(scale)高位0的个数
> //ASHIFT标识低位0的个数.也就是scale是2的ASHIFT次幂。
> //你细品。
> ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
> ```
>
> so.
>
> (a.length-1 ) & s 是不超过数组长度的下一个元素存放位置。也就是**top**。
>
> **top << ASHIFT等价于 scale*top**。
>
> ```java
> scale * top = 2 ^ ASHIFT * top = top << ASHIFT
> ```
>
> 也就计算出了存放位置的偏移量。

#### 线程池rs 锁/解锁

##### 加锁lockRunState()

```java
    private int lockRunState() {
        int rs;
      	//线程池没有被锁定
      	//RUNSTATE是runState的偏移量 RUNSTATE = U.objectFieldOffset(k.getDeclaredField("runState"));
        return ((((rs = runState) & RSLOCK) != 0 ||
                 //这里的锁是使用runState的cas实现的
                 !U.compareAndSwapInt(this, RUNSTATE, rs, rs |= RSLOCK)) ?
                //锁不成功就阻塞等待锁成功。
                awaitRunStateLock() : rs);
    }

		//阻塞-锁操作
    private int awaitRunStateLock() {
        Object lock;
        boolean wasInterrupted = false; //线程中断标记
      	//todo SPINS是干啥的？
        for (int spins = SPINS, r = 0, rs, ns;;) {
          	//未加锁
            if (((rs = runState) & RSLOCK) == 0) {
              	//尝试加锁 cas。外层自旋
                if (U.compareAndSwapInt(this, RUNSTATE, rs, ns = rs | RSLOCK)) {
                    if (wasInterrupted) {
                        try {//线程中断
                            Thread.currentThread().interrupt();
                        } catch (SecurityException ignore) {
                        }
                    }
                  	//自旋锁成功，跳出。返回rs | RSLOCK。
                    return ns;
                }
            }
          	//当前锁被其他线程占用。
          	//且threadLocalRandomSecondarySeed未初始化。就初始化。
          	//todo seed是干啥的？
            else if (r == 0)
                r = ThreadLocalRandom.nextSecondarySeed(); 
          	//todo 这个spins 没看到加上去的。初始化是0；
            else if (spins > 0) {
                r ^= r << 6; r ^= r >>> 21; r ^= r << 7; // xorshift
                if (r >= 0)
                    --spins;
            }
          	//当前线程不是线程池初始化的主线程，且切入了初始化的过程中，就让一让。交出cpu。
            else if ((rs & STARTED) == 0 || (lock = stealCounter) == null)
                Thread.yield();   // initialization race
          	// RSIGNAL: 1 << 1
          	// 线程池初始化完成。锁被其他线程占用。将rs的第二位标为1.
          	// 这里条件是 其他线程持有锁，且线程池已经完成初始化。
          	// 需要阻塞当前线程并等其他线程释放锁之后，再获取锁。
            else if (U.compareAndSwapInt(this, RUNSTATE, rs, rs | RSIGNAL)) {
              	//使用stealCounter计数器作为锁的通知对象。wait/notifyall
                synchronized (lock) {
                  	//双端检测。再判断RSIGNAL位是否为0.
                  	//如果是0：说明加锁前，已有线程进行了唤醒notifyall
                  	//如果是1：进行阻塞等待 wait
                    if ((runState & RSIGNAL) != 0) {
                        try {
                            lock.wait();
                        } catch (InterruptedException ie) {
                            if (!(Thread.currentThread() instanceof
                                  ForkJoinWorkerThread))
                                wasInterrupted = true;
                        }
                    }
                    else
                        lock.notifyAll();
                }
            }
        }
    }
```



##### 解锁unlockRunState(rs, (rs & ~RSLOCK) | ns)

> 解锁比较简单，就是使用cas把老状态替换为新状态。
>
>  ns为线程池运行状态 ns = STARTED; //1 << 2
>
> (rs & ~RSLOCK) 取消锁状态

```java
private void unlockRunState(int oldRunState, int newRunState) {
    if (!U.compareAndSwapInt(this, RUNSTATE, oldRunState, newRunState)) {
        Object lock = stealCounter;
        runState = newRunState;              // clears RSIGNAL bit
        if (lock != null)
            synchronized (lock) { lock.notifyAll(); }//唤醒其他等待线程
    }
}
```

> 注意：
>
> **是cas失败的时候才执行手动赋值/唤醒逻辑。**
>
> cas成功就解锁。
>
> **cas失败，说明运行状态有其他线程进行了RSIGNAL的等待而wait**，所以cas会失败，所以要唤醒。

#### 手动触发signalWork

> 之前的代码可以看到，任务加入队列之后，都进行了signalWork的手动触发操作。

```java
    // Masks and units for WorkQueue.scanState and ctl sp subfield
    static final int SCANNING     = 1;             // false when running tasks
    static final int INACTIVE     = 1 << 31;       // must be negative
    static final int SS_SEQ       = 1 << 16;       // version count
```



```java
    final void signalWork(WorkQueue[] ws, WorkQueue q) {
        long c; int sp, i; WorkQueue v; Thread p;
      	//ctl 高16位初始是并行度的负数，次16位是并行度的负数
      	//当工作线程数添加到并行度时：高16位和次16位为0，ctl为0及0+
        while ((c = ctl) < 0L) {                       // 还没有达到并行度
            if ((sp = (int)c) == 0) {                  // sp: ctl的低32位转为int，初始为0 「sp为等待线程worker数」
              
              	//ADD_WORKER = 0x0001L << (TC_SHIFT + 15);  0000000000000000 1000000000000000 0000000000000000 0000000000000000
              	//次16位的高位不为0.也就是总线程数TC还有剩余
                if ((c & ADD_WORKER) != 0L)            // 新增工作线程worker
                    tryAddWorker(c);
                break;
            }
          
          	//===========================todo==========================
          
            if (ws == null)                            // unstarted/terminated
                break;
          	//ws长度小于sp的低16位
            if (ws.length <= (i = sp & SMASK))         // terminated
                break;
          	//sp对应的WorkQueue为空 初始sq为0 todo
            if ((v = ws[i]) == null)                   // terminating
                break;
          	// SS_SEQ : 第17位为1 「1 << 16」也就是2^16
          	// INACTIVE : 第32位为1 「1 << 31」也就是2*32 ; ~INACTIVE:第31位为1
          	// vs : sp+2^16 todo
            int vs = (sp + SS_SEQ) & ~INACTIVE;        // next scanState scanState初始化：1 << 31 「<0: inactive; odd:scanning」
            int d = sp - v.scanState;                  // screen CAS
            long nc = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & v.stackPred);
            if (d == 0 && U.compareAndSwapLong(this, CTL, c, nc)) {
                v.scanState = vs;                      // activate v
                if ((p = v.parker) != null)
                    U.unpark(p);
                break;
            }
            if (q != null && q.base == q.top)          // no more work
                break;
        }
    }
```

##### tryAddWorker添加工作线程

```java
    //c = ctl
		private void tryAddWorker(long c) {
        boolean add = false;
        do {
          	//nc就是new c 。ac和tc都+1
            long nc = ((AC_MASK & (c + AC_UNIT)) |
                       (TC_MASK & (c + TC_UNIT)));
          	//查看这里的ctl有没有被其他线程修改过。
            if (ctl == c) {
                int rs, stop;                 // check if terminating
                if ((stop = (rs = lockRunState()) & STOP) == 0)
                    add = U.compareAndSwapLong(this, CTL, c, nc);//先更新ctl
                unlockRunState(rs, rs & ~RSLOCK);
                if (stop != 0)
                    break;
                if (add) {	//更新成功才添加worker
                    createWorker();
                    break;
                }
            }
        } while (((c = ctl) & ADD_WORKER) != 0L && (int)c == 0);
      	//每次刷新c=ctl
      	//tc仍有冗余，且sp「（int）c」==0 「没有等待线程」
      	//就一直创建新worker。
    }
```

```java
    private boolean createWorker() {
      	//线程工厂默认使用DefaultForkJoinWorkerThreadFactory
        ForkJoinWorkerThreadFactory fac = factory;
        Throwable ex = null;
        ForkJoinWorkerThread wt = null;// 新工作线程
        try {
          	//创建线程、并启动。
            if (fac != null && (wt = fac.newThread(this)) != null) {
                wt.start();
                return true;
            }
        } catch (Throwable rex) {
            ex = rex;
        }
      	//创建失败，worker取消注册。
        deregisterWorker(wt, ex);
        return false;
    }
```

### 工作线程





