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
>
> ![image-20200908103416001](https://gitee.com/wangigor/typora-images/raw/master/forkjoinpool流程.png)
>
> - 每个线程对应两个队列
>   - 偶数队列的任务来源于：外部业务线程
>   - 奇数队列的任务来源于：偶数队列任务fork出的新子任务。
> - 外部新任务进来时，通过ThreadLocalRandom.getProbe()随机探针，进入线程的偶数队列，并尝试唤醒线程工作。
> - 工作线程创建完成之后，首先循环扫描所有队列中的任务，窃取执行，如果没有，当前线程闲置，放入ctl中记录「为了建立闲置线程栈」。
> - 如果任务，需要join休眠等待，先尝试窃取其他线程的任务，为尽可能加速等待任务进行完成，再尝试唤醒闲置线程或者创建新替补线程，执行本线程的任务。进入休眠。

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

最后都调用了**externalPush**，将任务添加到线程池的「随机队列」中。

> 注意。这里是外部任务，提交到**==偶数队列==**

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
          	// INACTIVE : 第32位为1 「1 << 31」也就是2*32 ; ~INACTIVE:后31位为1
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

> 工作线程是使用ForkJoinWorkerThreadFactory创建的线程。
>
> ```java
> //默认采用DefaultForkJoinWorkerThreadFactory
> defaultForkJoinWorkerThreadFactory =
>          new DefaultForkJoinWorkerThreadFactory();
> ```
> ```java
>     public static interface ForkJoinWorkerThreadFactory {
>         /**
>          * Returns a new worker thread operating in the given pool.
>          *
>          * @param pool the pool this thread works in
>          * @return the new worker thread
>          * @throws NullPointerException if the pool is null
>          */
>         public ForkJoinWorkerThread newThread(ForkJoinPool pool);
>     }
> ```

>
>```java
>static final class DefaultForkJoinWorkerThreadFactory
>    implements ForkJoinWorkerThreadFactory {
>    public final ForkJoinWorkerThread newThread(ForkJoinPool pool) {
>        return new ForkJoinWorkerThread(pool);
>    }
>}
>```
>
>```java
>    protected ForkJoinWorkerThread(ForkJoinPool pool) {
>        // Use a placeholder until a useful name can be set in registerWorker
>        super("aForkJoinWorkerThread");
>        this.pool = pool;
>        this.workQueue = pool.registerWorker(this);
>    }
>```
>
>ForkJoinWorkerThread继承自thread。创建完成后，注册到线程池。

#### pool.registerWorker(this)注册worker

```java
    final WorkQueue registerWorker(ForkJoinWorkerThread wt) {
      	//异常处理器 可使用java.util.concurrent.ForkJoinPool.common.exceptionHandler系统参数指定
      	//默认为空
        UncaughtExceptionHandler handler;
        wt.setDaemon(true);                           // 设置为守护线程。「程序该停止就停止，不需要等待这个线程跑完」
        if ((handler = ueh) != null)
            wt.setUncaughtExceptionHandler(handler);
      
      	//创建workQueue
        WorkQueue w = new WorkQueue(this, wt);
        int i = 0;                                    // assign a pool index
        int mode = config & MODE_MASK; //获取异步模式
        int rs = lockRunState(); 
        try {
            WorkQueue[] ws; int n;                    // skip if no array
            if ((ws = workQueues) != null && (n = ws.length) > 0) {
              	//SEED_INCREMENT是0x9e3779b9
              	//他是由黄金分割比例得到的一个32位整数
              	//phi = (1 + sqrt(5)) / 2 = 1,6180339887498948482045868343656
              	//2^32 / phi = 0x9e3779b9
              	//为了更离散。防止碰撞
                int s = indexSeed += SEED_INCREMENT;  // unlikely to collide
                int m = n - 1;
                i = ((s << 1) | 1) & m;               // odd-numbered indices 奇数下标
                if (ws[i] != null) {                  // collision 发生了碰撞的情况
                    int probes = 0;                   // step by approx half n 步长为n/2
                    int step = (n <= 4) ? 2 : ((n >>> 1) & EVENMASK) + 2;
                  	//递增一个step。再次判断槽位冲突
                    while (ws[i = (i + step) & m] != null) {
                      	//尝试n次。如果还冲突就扩容
                        if (++probes >= n) {
                            workQueues = ws = Arrays.copyOf(ws, n <<= 1);	//扩容
                            m = n - 1;
                            probes = 0;
                        }
                    }
                }
              	//找到一个空槽位，防止对应的workQueue
                w.hint = s;                           // use as random seed
                w.config = i | mode;									// 记录模式
                w.scanState = i;                      // publication fence 池中下标
                ws[i] = w;
            }
        } finally {
            unlockRunState(rs, rs & ~RSLOCK);
        }
      	//设置线程名称。前缀+下标的一半。「数组可以理解为双倍的线程数，并发数的进位二次幂的两倍」
        wt.setName(workerNamePrefix.concat(Integer.toString(i >>> 1)));
        return w;
    }
```

#### deregisterWorker(wt, ex)取消注册

> 创建线程过程中发生异常，就取消注册。

```java
    final void deregisterWorker(ForkJoinWorkerThread wt, Throwable ex) {
        WorkQueue w = null;
      	//清空队列组中的线程对应队列
        if (wt != null && (w = wt.workQueue) != null) {
            WorkQueue[] ws;                           // remove index from array
            int idx = w.config & SMASK;
            int rs = lockRunState();
            if ((ws = workQueues) != null && ws.length > idx && ws[idx] == w)
                ws[idx] = null;
            unlockRunState(rs, rs & ~RSLOCK);
        }
        long c;                                       // decrement counts
      	// ac 和 tc 减一
        do {} while (!U.compareAndSwapLong
                     (this, CTL, c = ctl, ((AC_MASK & (c - AC_UNIT)) |
                                           (TC_MASK & (c - TC_UNIT)) |
                                           (SP_MASK & c))));
      	// 清空队列 。取消等待任务
        if (w != null) {
            w.qlock = -1;                             // ensure set
            w.transferStealCount(this);
            w.cancelAll();                            // cancel remaining tasks
        }
        for (;;) {                                    // possibly replace
            WorkQueue[] ws; int m, sp;
          	//尝试终止线程池
						//如果终止不了，看线程池是否已经在进行停止操作          	
            if (tryTerminate(false, false) || w == null || w.array == null ||
                (runState & STOP) != 0 || (ws = workQueues) == null ||
                (m = ws.length - 1) < 0)              // already terminating
                break;
          	//如果有不活动的等待线程
          	//进行释放
            if ((sp = (int)(c = ctl)) != 0) {         // wake up replacement
                if (tryRelease(c, ws[sp & m], AC_UNIT))
                    break;
            }
          	//有异常
          	//没有等待线程
          	//线程总数未到达并行度 创建一个worker todo 为啥？
            else if (ex != null && (c & ADD_WORKER) != 0L) {
                tryAddWorker(c);                      // create replacement
                break;
            }
            else                                      // don't need replacement
                break;
        }
      	//异常处理
        if (ex == null)                               // help clean on way out
            ForkJoinTask.helpExpungeStaleExceptions();
        else                                          // rethrow
            ForkJoinTask.rethrow(ex);
    }
```

#### 启动 wt.start();

```java
    public synchronized void start() {

        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
            }
        }
    }

    private native void start0();

    public void run() {
        if (workQueue.array == null) { // only run once
            Throwable exception = null;
            try {
                onStart();
                pool.runWorker(workQueue);
            } catch (Throwable ex) {
                exception = ex;
            } finally {
                try {
                    onTermination(exception);
                } catch (Throwable ex) {
                    if (exception == null)
                        exception = ex;
                } finally {
                    pool.deregisterWorker(this, exception);
                }
            }
        }
    }
```

> 线程启动之后，就会调用 pool.runWorker(workQueue);方法。

#### 获取并执行任务 pool.runWorker(workQueue)

```java
    final void runWorker(WorkQueue w) {
      	//初始化任务队列
        w.growArray();                   // allocate queue
        int seed = w.hint;               // initially holds randomization hint 随机数
        int r = (seed == 0) ? 1 : seed;  // avoid 0 for xorShift 非零随机数
        for (ForkJoinTask<?> t;;) {
          	//获取任务
            if ((t = scan(w, r)) != null)
                w.runTask(t);	//执行任务
            else if (!awaitWork(w, r))
                break;
            r ^= r << 13; r ^= r >>> 17; r ^= r << 5; // xorshift
        }
    }
```

##### 获取任务scan(w, r)

> 按照WorkQueues长度，随机获取队列，进行窃取任务。

```java
    private ForkJoinTask<?> scan(WorkQueue w, int r) {
        WorkQueue[] ws; int m;
        if ((ws = workQueues) != null && (m = ws.length - 1) > 0 && w != null) {
          	//对应workQueues下标「对应线程标记」 初始非负
            int ss = w.scanState;                     // initially non-negative
          	//k和origin不超过workQueues的「随机下标」
            for (int origin = r & m, k = origin, oldSum = 0, checkSum = 0;;) {
                WorkQueue q; ForkJoinTask<?>[] a; ForkJoinTask<?> t;
                int b, n; long c;
              	//随机队列不为空
                if ((q = ws[k]) != null) {
                  	//队列中有任务
                    if ((n = (b = q.base) - q.top) < 0 &&
                        (a = q.array) != null) {      // non-empty
                      	//base元素对应的偏移量
                        long i = (((a.length - 1) & b) << ASHIFT) + ABASE;
                      	//获取任务
                        if ((t = ((ForkJoinTask<?>)
                                  U.getObjectVolatile(a, i))) != null &&
                            q.base == b) {
                          	//第一次总会进来这里的。
                            if (ss >= 0) {
                              	//cas 尝试「窃取」任务 ，将原base位置设为空，base右移一位。
                                if (U.compareAndSwapObject(a, i, t, null)) {
                                    q.base = b + 1;
                                  	//如果队列中还有剩余，唤醒其他线程处理。
                                    if (n < -1)       // signal others
                                        signalWork(ws, q);
                                    return t;
                                }
                            }
                          	//如果当前队列对应线程未活动，尝试激活ctl对应的id「非活动线程栈栈顶的线程」
                            else if (oldSum == 0 &&   // try to activate
                                     w.scanState < 0)
                                tryRelease(c = ctl, ws[m & (int)c], AC_UNIT);
                        }
                        if (ss < 0)                   // refresh
                            ss = w.scanState;
                      	//重新随机r 重置k/origin/oldsum/checksum
                        r ^= r << 1; r ^= r >>> 3; r ^= r << 10;
                        origin = k = r & m;           // move and rescan
                        oldSum = checkSum = 0;
                        continue;
                    }
                    checkSum += b; //checkSum每次都会都会加上当前队列的base值。
                }
              	
              	//每次循环都要进行k+1
              	//说明没有窃取到任务「窃取任务失败」
              	//如果进入这个逻辑，说明遍历了一圈都没有窃取到任务，需要将当前线程闲置
                if ((k = (k + 1) & m) == origin) {    // continue until stable
                    if ((ss >= 0 || (ss == (ss = w.scanState))) &&
                        oldSum == (oldSum = checkSum)) {//这里说明每个队列的base值都没有变过
                        if (ss < 0 || w.qlock < 0)    // already inactive
                            break;
                        int ns = ss | INACTIVE;       // try to inactivate 尝试「终止」当前队列
                        long nc = ((SP_MASK & ns) |		//ns的低32位
                                   (UC_MASK & ((c = ctl) - AC_UNIT)));//原ctl的高32位
                        w.stackPred = (int)c;         // hold prev stack top //设置当前队列的前置等待队列为ctl的栈顶线程
                        U.putInt(w, QSCANSTATE, ns); 	// 当前队列设为不活动。
                        if (U.compareAndSwapLong(this, CTL, c, nc))//更新ctl
                            ss = ns;
                        else
                            w.scanState = ss;         // back out
                    }
                    checkSum = 0;
                }
            }
        }
        return null; //返回null，说明当前线程和队列，需要闲置。
    }
```

##### 执行任务w.runTask(t)



```java
        final void runTask(ForkJoinTask<?> task) {
            if (task != null) {
                scanState &= ~SCANNING; // mark as busy 设置当前队列的状态为SCANNING 奇数
              	//执行窃取的任务
                (currentSteal = task).doExec();
                U.putOrderedObject(this, QCURRENTSTEAL, null); // release for GC
              	//执行工作线程对应的workQueue的任务
              	//这里其实就是根据mode模式，开始逐个取出task，进行doExec
                execLocalTasks();//注意。这里本地任务，是指线程存的奇数队列。
                ForkJoinWorkerThread thread = owner;
                if (++nsteals < 0)      // collect on overflow
                    transferStealCount(pool);
                scanState |= SCANNING;
                if (thread != null)
                    thread.afterTopLevelExec();
            }
        }
```

> 那doExec()是怎么跟业务代码执行到一起的呢？

```java
    //状态字段将运行控制状态位打包为单个int，以最小化占用空间并确保原子性（通过CAS）。
		//状态最初为零，并采用非负值，直到完成为止，然后状态（与DONE_MASK一起）保持值为NORMAL，CANCELLED或EXCEPTIONAL。
		//其他线程正在等待阻塞的任务将SIGNAL位置1。
		//设置了SIGNAL的被盗任务完成后，将通过notifyAll唤醒任何侍者。
		//即使出于某些目的不是最优的，我们还是使用基本的内置等待/通知来利用JVM中的“监视器膨胀”，否则我们将需要模拟JVM以避免增加每个任务的簿记开销。我们希望这些监视器是“胖”的，即，不使用偏置或细锁技术，因此使用一些倾向于避免它们的奇数编码习惯，主要是通过安排每个同步块执行一个wait，notifyAll或两者。
 		//这些控制位仅占用状态字段的上半部分（16位）中的一部分。低位用于用户定义的标签。
		//任务状态
		volatile int status; // accessed directly by pool and workers
    static final int DONE_MASK   = 0xf0000000;  // mask out non-completion bits 为了屏蔽掉非完成状态使用的mask
    static final int NORMAL      = 0xf0000000;  // must be negative 完成状态 负数
    static final int CANCELLED   = 0xc0000000;  // must be < NORMAL 取消
    static final int EXCEPTIONAL = 0x80000000;  // must be < CANCELLED 异常
    static final int SIGNAL      = 0x00010000;  // must be >= 1 << 16 
    static final int SMASK       = 0x0000ffff;  // short bits for tags 为了获取后16位

		final int doExec() {
        int s; boolean completed;
      	//非完成状态
        if ((s = status) >= 0) {
            try {
              	//执行
                completed = exec();
            } catch (Throwable rex) {
                return setExceptionalCompletion(rex);
            }
            if (completed)
                s = setCompletion(NORMAL);
        }
        return s;
    }
```

```java
//一个需要子类重写的执行方法
protected abstract boolean exec();
```

> 以自定义fork-join测试的例子，继承了RecursiveTask。
>
> ```java
>     //自定义fork-join的方法
> 		protected abstract V compute();
> 		
> 		//执行
>     protected final boolean exec() {
>         result = compute();
>         return true;
>     }
> ```



##### 等到任务awaitWork(w, r)



### 线程池终止与释放

#### 终止

> tryTerminate(false, false)
>
> 第一个参数now:
>
> - true：无条件终止
> - false：要工作到没有工作线程和任务的时候，再终止
>
> 第二个参数enable：是否开启SHUTDOWN状态。开启了可在下一次进行关闭

```java
    /**
     * Possibly initiates and/or completes termination.
     *	
     * @param now if true, unconditionally terminate, else only
     * if no work and no active workers
     * @param enable if true, enable shutdown when next possible
     * @return true if now terminating or terminated
     */
		private boolean tryTerminate(boolean now, boolean enable) {
        int rs;
      	//如果是公共线程池common实例，不终止。
      	//这个common实在ForkJoinPool的静态代码块中生成的，为了防止单个jvm会产生多个池实例。也供lambda的parallel流使用。
        if (this == common)                       // cannot shut down
            return false;
      	//线程池状态变更为SHUTDOWN
      	//enable 为false，不能终止。
        if ((rs = runState) >= 0) {
            if (!enable)
                return false;
            rs = lockRunState();                  // enter SHUTDOWN phase
            unlockRunState(rs, (rs & ~RSLOCK) | SHUTDOWN);
        }
				//查看线程池状态是否为STOP
        if ((rs & STOP) == 0) {
          	//如果不是立即关闭，进行下面逻辑
            if (!now) {                           // check quiescence
                for (long oldSum = 0L;;) {        // repeat until stable
                    WorkQueue[] ws; WorkQueue w; int m, b; long c;
                    long checkSum = ctl;
                  	//如果ac+并行度大于0
                  	//说明有工作线程。不终止
                    if ((int)(checkSum >> AC_SHIFT) + (config & SMASK) > 0)
                        return false;             // still active workers
                  	//检查队列组。为空。跳出循环。
                    if ((ws = workQueues) == null || (m = ws.length - 1) <= 0)
                        break;                    // check queues
                  	//队列组不为空。就遍历
                    for (int i = 0; i <= m; ++i) {
                      	//获取工作队列。为空就不管他了。
                        if ((w = ws[i]) != null) {
                          	//如果队列中有任务。或者工作队列状态不是未活动「1<<31小于零」。或者正在窃取任务。
                            if ((b = w.base) != w.top || w.scanState >= 0 ||
                                w.currentSteal != null) {
                              	//释放逻辑。
                                tryRelease(c = ctl, ws[m & (int)c], AC_UNIT);
                                return false;     // arrange for recheck 当前不关闭。
                            }
                          	//队列中已经没有任务了。
                          	//这里是获取每一个队列的base索引相加。为了做变动测试。
                            checkSum += b;
                            if ((i & 1) == 0) //todo qlock不知道干啥的。
                                w.qlock = -1;     // try to disable external
                        }
                    }
                  	//说明跟上一次所有队列的base相加，没有变动。跳出循环
                    if (oldSum == (oldSum = checkSum))
                        break;
                }
            }
          	//将线程池状态改为STOP
            if ((runState & STOP) == 0) {
                rs = lockRunState();              // enter STOP phase
                unlockRunState(rs, (rs & ~RSLOCK) | STOP);
            }
        }

      
      	//完成终止操作 。「三次确认。」
        int pass = 0;                             // 3 passes to help terminate
        for (long oldSum = 0L;;) {                // or until done or stable
            WorkQueue[] ws; WorkQueue w; ForkJoinWorkerThread wt; int m;
            long checkSum = ctl;
          	//已经没有工作者了 或者工作队列组为空 
            if ((short)(checkSum >>> TC_SHIFT) + (config & SMASK) <= 0 ||
                (ws = workQueues) == null || (m = ws.length - 1) <= 0) {
              	//就把线程池状态改为TERMINATED
                if ((runState & TERMINATED) == 0) {
                    rs = lockRunState();          // done
                    unlockRunState(rs, (rs & ~RSLOCK) | TERMINATED);
                    synchronized (this) { notifyAll(); } // for awaitTermination
                }
                break; //这里终止循环就行了。不需要再继续操作了。线程池已经停止了。
            }
          	//遍历工作队列组
            for (int i = 0; i <= m; ++i) {
                if ((w = ws[i]) != null) {
                  	//这里还是为了校验和的变动测试
                    checkSum += w.base;
                    w.qlock = -1;                 // try to disable todo 不懂。
                    if (pass > 0) {
                        w.cancelAll();            // clear queue 清空队列
                        if (pass > 1 && (wt = w.owner) != null) {
                            if (!wt.isInterrupted()) {
                                try {             // unblock join
                                    wt.interrupt(); //中断线程
                                } catch (Throwable ignore) {
                                }
                            }
                            if (w.scanState < 0)
                                U.unpark(wt);     // wake up 唤醒当前队列对应线程
                        }
                    }
                }
            }
          	//说明还在清理中。下一次循环再试。
            if (checkSum != oldSum) {             // unstable
                oldSum = checkSum;
                pass = 0; //创智pass
            }
            else if (pass > 3 && pass > m)        // can't further help
                break;
            else if (++pass > 1) {                // try to dequeue
                long c; int j = 0, sp;            // bound attempts
              	//遍历释放每一个队列。
                while (j++ <= m && (sp = (int)(c = ctl)) != 0)
                    tryRelease(c, ws[sp & m], AC_UNIT);
            }
        }
        return true;
    }
```

#### 释放

> 要等等。看不懂。todo。

```java
    /**
     * Signals and releases worker v if it is top of idle worker
     * stack.  This performs a one-shot version of signalWork only if
     * there is (apparently) at least one idle worker.
     *
     * @param c incoming ctl value 最新的ctl数据
     * @param v if non-null, a worker 
     * @param inc the increment to active count (zero when compensating)
     * @return true if successful
     */
		private boolean tryRelease(long c, WorkQueue v, long inc) {
        int sp = (int)c, vs = (sp + SS_SEQ) & ~INACTIVE; Thread p;
        if (v != null && v.scanState == sp) {          // v is at top of stack
            long nc = (UC_MASK & (c + inc)) | (SP_MASK & v.stackPred);
            if (U.compareAndSwapLong(this, CTL, c, nc)) {
                v.scanState = vs;
                if ((p = v.parker) != null)
                    U.unpark(p);
                return true;
            }
        }
        return false;
    }
```



### 工作队列WorkQueue

```java
        //构造方法
				WorkQueue(ForkJoinPool pool, ForkJoinWorkerThread owner) {
            this.pool = pool;
            this.owner = owner;
          	//将base和top的初始位置都放置在队列中心
            // Place indices in the center of array (that is not yet allocated)
            base = top = INITIAL_QUEUE_CAPACITY >>> 1;
        }
```

#### 属性

```java
				static final int INITIAL_QUEUE_CAPACITY = 1 << 13;//初始容量

        static final int MAXIMUM_QUEUE_CAPACITY = 1 << 26; // 64M 最大容量

        volatile int scanState;    // versioned, <0: inactive; odd:scanning 扫描状态 <0未激活；奇数在扫描中，且对应WorkQueue[]下标
        int stackPred;             // pool stack (ctl) predecessor 指向了前一个等待线程的索引 ctl的低16位指向了第一个等待线程的索引，这就形成了一个等待线程「链」
        int nsteals;               // number of steals 窃取数量
        int hint;                  // randomization and stealer index hint 随机和窃取下标提示
        int config;                // pool index and mode 高16位为mode。低16位是WorkQueue[]下标
        volatile int qlock;        // 1: locked, < 0: terminate; else 0 锁
        volatile int base;         // index of next slot for poll base初始在队列中心，供其他线程窃取，所有volatile。
        int top;                   // index of next slot for push top初始跟base一样，供本线程push用
        ForkJoinTask<?>[] array;   // the elements (initially unallocated) 任务队列。
        final ForkJoinPool pool;   // the containing pool (may be null) 线程池实例。
        final ForkJoinWorkerThread owner; // owning thread or null if shared 所有的worker线程
        volatile Thread parker;    // == owner during call to park; else null 
        volatile ForkJoinTask<?> currentJoin;  // task being joined in awaitJoin 当前正在join等待结果的任务。
        volatile ForkJoinTask<?> currentSteal; // mainly used by helpStealer 当前执行的任务是steal来的任务，做记录。
```

#### 取出任务

```java
        //取出指定任务 「当前top任务」
				final boolean tryUnpush(ForkJoinTask<?> t) {
            ForkJoinTask<?>[] a; int s;
          	//通过cas，将top-1的位置的t任务置为null，并更新top位置
            if ((a = array) != null && (s = top) != base &&
                U.compareAndSwapObject
                (a, (((a.length - 1) & --s) << ASHIFT) + ABASE, t, null)) {
                U.putOrderedInt(this, QTOP, s);
                return true;
            }
            return false;
        }
```

```java
        //仅由awaitJoin调用
				//从队列中取出给定任务，执行。
        final boolean tryRemoveAndExec(ForkJoinTask<?> task) {
            ForkJoinTask<?>[] a; int m, s, b, n;
            if ((a = array) != null && (m = a.length - 1) >= 0 &&
                task != null) {
                while ((n = (s = top) - (b = base)) > 0) {
                    for (ForkJoinTask<?> t;;) {      // traverse from s to b
                      	//从top开始一个一个往下找
                        long j = ((--s & m) << ASHIFT) + ABASE;
                      	//获取的任务为null 说明当前任务已经被其他线程并发窃取走了。
                      	//而且，因为窃取是从base开始，当前队列的执行是从top开始，说明当前已经执行到了base节点。
                      	//如果是栈顶元素，就返回true，不是栈顶元素，返回false。
                      	//这里的返回是要进行helpStealer操作，如果在栈顶位置被窃取，说明当前队列，也没有可执行任务了，可以去帮助窃取线程执行它的任务
                      	//如果不是栈顶，说明自己的「排在前面的待执行任务」可能还没有执行完，不需要去窃取其他任务。
                        if ((t = (ForkJoinTask<?>)U.getObject(a, j)) == null)
                            return s + 1 == top;     // shorter than expected
                      	//获取到指定任务任务
                        else if (t == task) {
                            boolean removed = false;
                          	//如果是在top节点，top节点置空。
                          	//如果cas失败，说明当前要执行的节点已经被窃取，当前队列已经空了。
                            if (s + 1 == top) {      // pop
                                if (U.compareAndSwapObject(a, j, task, null)) {
                                    U.putOrderedInt(this, QTOP, s);
                                    removed = true;
                                }
                            }
                          	//task不在栈顶，且base没有发生变化，将当前任务位置替换成EmptyTask「初始状态就是NORMAL完成」
                          	//替换成EmptyTask而不是null，是因为队列要保证非空连续，在前面的判断中，null会被判断为空队列。
                            else if (base == b)      // replace with proxy
                              	//如果base发生改变不进该逻辑，因为有其他空闲线程在进行任务窃取，就下一个循环再等等看。
                              	//不行的话再交由等待线程处理。
                                removed = U.compareAndSwapObject(
                                    a, j, task, new EmptyTask());
                          	//删除之后，当前「join」线程执行任务。跳出循环返回。
                            if (removed)
                                task.doExec();
                            break;
                        }
                      	//其他任务检查，已完成的清除掉。
                      	//其实任务要先弹栈置空才执行，这里主要是针对因为join而置换进来的EmptyTask
                        else if (t.status < 0 && s + 1 == top) {
                            if (U.compareAndSwapObject(a, j, t, null))
                                U.putOrderedInt(this, QTOP, s);
                            break;                  // was cancelled
                        }
                      	//找到base节点了，都没有return，就返回失败
                      	//如果这里要是返回true，需要的条件是空队列。而其他线程「可能排在前面的线程」不归在这里判断。
                      	//还不能判断是空队列，需要去窃取任务，所以是false。
                        if (--n == 0)
                            return false;
                    }
                  	//任务已完成状态，返回执行失败。
                    if (task.status < 0)
                        return false;
                }
            }
            return true;
        }
```





### 任务执行结果

> ForkJoinTask是Future的实现类，可以通过get方法获取结果。
>
> 但是ForkJoin框架的任务有自己特有的方法，fork()「提交」和join()「获取结果」。

#### fork



```java
    public final ForkJoinTask<V> fork() {
        Thread t;
      	//如果当前线程是ForkJoinWorkerThread，则放入对应的偶数队列workQueue中。
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
            ((ForkJoinWorkerThread)t).workQueue.push(this);
      	//否则，使用将任务作为外部提交放入线程池中。
        else
            ForkJoinPool.common.externalPush(this);
        return this;
    }
```

> fork也有两种情况
>
> - 「**偶数下标队列**」外部「业务」线程提交任务。externalPush前面讲过
> - 「**奇数下标队列**」线程池线程ForkJoinWorkerThread的任务fork出的新子任务。

ForkJoinWorkerThread的任务fork出的新子任务调用

```java
((ForkJoinWorkerThread)t).workQueue.push(this);
```

```java
//仅由非共享队列的所有者调用，也就是ForkJoinWorkerThread对应的奇数队列，只能由ForkJoinWorkerThread放入元素
//这个队列的任务，是其他线程的窃取对象。
final void push(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; ForkJoinPool p;
    int b = base, s = top, n;
    if ((a = array) != null) {    // ignore if queue removed
        int m = a.length - 1;     // fenced write for task visibility
        U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
        U.putOrderedInt(this, QTOP, s + 1);
        if ((n = s - b) <= 1) {
            if ((p = pool) != null)
                p.signalWork(p.workQueues, this);
        }
        else if (n >= m)
            growArray();
    }
}
```

#### join

> 等待执行结果

```java
    public final V join() {
        int s;
      	//调用doJoin执行任务
      	//非正常结束，调用reportException报告异常。
      	//正常结束，使用getRawResult获取结果返回。
        if ((s = doJoin() & DONE_MASK) != NORMAL)
            reportException(s);
        return getRawResult();
    }
```

##### doJoin()

> 只有当前任务是栈顶任务，会被立即执行。
>
> 否则进行awaitJoin。

```java
    private int doJoin() {
        int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
      	//如果任务状态已完成，返回完成状态
      	//如果当前线程是ForkJoinWorkerThread：
      	//	尝试从对应队列中取出并执行。执行完成，返回完成状态，否则「不是栈顶任务」awaitJoin。
      	//否则调用externalAwaitDone()等待任务执行完成。
        return (s = status) < 0 ? s :
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
            (w = (wt = (ForkJoinWorkerThread)t).workQueue).
            tryUnpush(this) && (s = doExec()) < 0 ? s :
            wt.pool.awaitJoin(w, this, 0L) :
            externalAwaitDone();
    }
```

awaitJoin方法

```java
    //假设有三个任务「A,B,C」顺序进行fork，任务栈为base[A,B,C]top.
		//进行顺序join「而不是fork的逆序」，在A,B的join时，就会awaitJoin，因为任务不在栈顶。
		final int awaitJoin(WorkQueue w, ForkJoinTask<?> task, long deadline) {
        int s = 0;
        if (task != null && w != null) {
          	//先获取一下之前的等待任务。
            ForkJoinTask<?> prevJoin = w.currentJoin;
          	//设置当前队列的等待任务currentJoin
            U.putOrderedObject(w, QCURRENTJOIN, task);
          	//对于CountedCompleter「ForkJoinTask的子类，如果没有后续等待线程，就执行任务完成“通知”」的特殊处理
          	//这个类，暂时还没研究过
            CountedCompleter<?> cc = (task instanceof CountedCompleter) ?
                (CountedCompleter<?>)task : null;
          	//「轮询等待」
            for (;;) {
              	//任务执行完成跳出
                if ((s = task.status) < 0)
                    break;
                if (cc != null)//帮助完成，用于CountedCompleter类，暂不研究。
                    helpComplete(w, cc, 0);
              	//当前队列空了，获取自己队列中没找到该任务，说明这个任务已经被其他线程窃取
              	//执行helpStealer。寻找到窃取它的线程，帮它「窃取线程」执行任务。
              	//因为可能当前任务需要等待其他任务执行完成，而其他任务可能在队列中，这里帮一帮，窃取一下，「尽可能」加速处理。使得join返回。
                else if (w.base == w.top || w.tryRemoveAndExec(task))
                    helpStealer(w, task);
              	//任务执行完毕
                if ((s = task.status) < 0)
                    break;
              	//计算deadline时间
                long ms, ns;
                if (deadline == 0L)
                    ms = 0L;
              	//超时就不等了
                else if ((ns = deadline - System.nanoTime()) <= 0L)
                    break;
              	//小于1ms按1ms算
                else if ((ms = TimeUnit.NANOSECONDS.toMillis(ns)) <= 0L)
                    ms = 1L;
              	//补偿措施
                if (tryCompensate(w)) {
                  	//线程休眠，等待任务运行完成
                  	//这里是用wait阻塞的
                    task.internalWait(ms);
                  	//线程被唤醒后，增加ctl的活跃线程数。
                    U.getAndAddLong(this, CTL, AC_UNIT);
                }
            }
          	//恢复为前一个等待任务。
            U.putOrderedObject(w, QCURRENTJOIN, prevJoin);
        }
        return s;
    }
```

helpStealer

> 为了加速要等待task的join返回，窃取其他队列的任务执行。

```java
    private void helpStealer(WorkQueue w, ForkJoinTask<?> task) {
        WorkQueue[] ws = workQueues;
        int oldSum = 0, checkSum, m;
      	//非空容错
        if (ws != null && (m = ws.length - 1) >= 0 && w != null &&
            task != null) {
          	//如果等待的task的任务没有执行完成，就继续循环窃取。
          	//如果多次计算所有队列的base的计算和都一样，说明也没有窃取的必要了。
            do {                                       // restart point
                checkSum = 0;                          // for stability check
                ForkJoinTask<?> subtask;
                WorkQueue j = w, v;                    // v is subtask stealer
              	//外层的descent循环，查看任务是否已经完成。
                descent: for (subtask = task; subtask.status >= 0; ) {
                  	//h: 当前线程的hint，初始为队列组下标；如果发生窃取，hint为窃取队列的下标
                  	//|1 ：将值变为奇数，步长为2.
                    for (int h = j.hint | 1, k = 0, i; ; k += 2) {
                      	//索引下标已经越界，没有找到对应的窃取队列，就退出。
                      	//任务可能已经完成，进入下一次循环检测。
                        if (k > m)                     // can't find stealer
                            break descent;
                      	//逐步查找对应任务所在的队列，找到窃取线程队列。
                        if ((v = ws[i = (h + k) & m]) != null) {
                          	//判断是否是窃取线程
                            if (v.currentSteal == subtask) {
                              	//如果是，记录窃取线程索引
                                j.hint = i;
                                break;
                            }
                          	//计算base校验和。
                            checkSum += v.base;
                        }
                    }
                    for (;;) {                         // help v or descend
                        ForkJoinTask<?>[] a; int b;
                      	//找到了窃取队列，也计算一下窃取队列的base校验和。
                      	//因为找到窃取队列的时候提前break了。
                        checkSum += (b = v.base);
                      	//获取窃取线程正在join的任务
                        ForkJoinTask<?> next = v.currentJoin;
                      	//subtask.status < 0：如果当前窃取线程要join的任务已经完成。就退出
                      	//j.currentJoin != subtask：如果当前「被窃取线程」的等待任务不是我们需要等待的任务。就退出「可能在之前的步骤中，任务已经执行完成，进入下一个循环查看目标任务状态」
                        //v.currentSteal != subtask：如果当前窃取线程的窃取任务发生变化。就退出「说明目标等待任务可能已经完成」
                      	if (subtask.status < 0 || j.currentJoin != subtask ||
                            v.currentSteal != subtask) // stale
                            break descent;
                      	//如果窃取任务队列没有等待任务
                        if (b - v.top >= 0 || (a = v.array) == null) {
                          	//而且窃取线程没有join等待完成的任务，退出重新判断任务是否已完成
                            if ((subtask = next) == null)
                                break descent;
                          	//否则帮窃取线程A的窃取线程B去执行任务
                            j = v;
                            break;
                        }
                      	//窃取窃取线程的任务，从base开始
                        int i = (((a.length - 1) & b) << ASHIFT) + ABASE;
                        ForkJoinTask<?> t = ((ForkJoinTask<?>)
                                             U.getObjectVolatile(a, i));
                      	//相当于双端检测，base有没有变
                      	//避免进行无效的并发窃取。
                        if (v.base == b) {
                          	//说明可能其他线程已经窃取了这里的base，还没来得及更新base的下标
                            if (t == null)             // stale
                                break descent;
                          	//cas弹出任务，置空base
                            if (U.compareAndSwapObject(a, i, t, null)) {
                              	//base右移
                                v.base = b + 1;
                              	//记录被窃取队列w的窃取任务
                                ForkJoinTask<?> ps = w.currentSteal;
                                int top = w.top;
                                do {
                                  	//将窃取来的任务，放置在currentSteal中记录
                                    U.putOrderedObject(w, QCURRENTSTEAL, t);
                                  	//执行任务
                                    t.doExec();        // clear local tasks too
                                  //要等待的任务没有执行完，并且，刚才执行的任务发生了fork新任务，那么继续执行新任务。
                                } while (task.status >= 0 &&
                                         w.top != top &&
                                         (t = w.pop()) != null);
                              	//恢复上一个窃取任务。
                                U.putOrderedObject(w, QCURRENTSTEAL, ps);
                              	//如果w有自己的local任务需要执行，就先不窃取了。
                                if (w.base != w.top)
                                    return;            // can't further help
                            }
                        }
                    }
                }
            } while (task.status >= 0 && oldSum != (oldSum = checkSum));
        }
    }
```



 tryCompensate

> 阻塞前，尝试创建一个补偿线程去帮忙处理下任务。
>
> 如果不能阻塞，就不阻塞，返回false。

```java
private boolean tryCompensate(WorkQueue w) {
    boolean canBlock; //是否可以阻塞标记
    WorkQueue[] ws; long c; int m, pc, sp;
  	//当前队列被锁，为空，都不可以阻塞
    if (w == null || w.qlock < 0 ||           // caller terminating
        (ws = workQueues) == null || (m = ws.length - 1) <= 0 ||
        (pc = config & SMASK) == 0)           // parallelism disabled
        canBlock = false;
  	//存在非活动线程，尝试释放它，加快任务进行。
  	//如果非活动线程唤醒成功，当前线程阻塞。返回唤醒结果
    else if ((sp = (int)(c = ctl)) != 0)      // release idle worker
        canBlock = tryRelease(c, ws[sp & m], 0L);
    else {
      	//获取ac和tc
        int ac = (int)(c >> AC_SHIFT) + pc;
        int tc = (short)(c >> TC_SHIFT) + pc;
        int nbusy = 0;                        // validate saturation
      	//饱和度校验
        for (int i = 0; i <= m; ++i) {        // two passes of odd indices 
            WorkQueue v;
          	//获取所有的奇数节点，如果活动，nbusy+1；
          	//注意！这里计算了两次，因为++i，且&m.
          	//为啥需要两次：可能是看为了减小误差
            if ((v = ws[((i << 1) | 1) & m]) != null) {
                if ((v.scanState & SCANNING) != 0)
                    break;
                ++nbusy;
            }
        }
      	//看工作个数是否跟tc相等，不相等说明有空闲线程
      	//先不等待，看看任务执行完了没有
        if (nbusy != (tc << 1) || ctl != c)
            canBlock = false;                 // unstable or stale
      	//tc >= pc :如果当前线程总数已经达到并行度。
      	//且 ac > 1 :有除自己以外的活动线程。
      	// 且 w.isEmpty() : 么有其他任务要执行了
      	//就不需要创建其他替补线程来处理任务了。
        else if (tc >= pc && ac > 1 && w.isEmpty()) {
          	//否则，自己需要休眠、
          	//减少活跃线程数
            long nc = ((AC_MASK & (c - AC_UNIT)) |
                       (~AC_MASK & c));       // uncompensated
            canBlock = U.compareAndSwapLong(this, CTL, c, nc);
        }
      	//如果超过了最大线程数
      	//或者对于common池，超过了指定并发度+common池默认的最大线程数
      	//抛出RejectedExecutionException异常
        else if (tc >= MAX_CAP ||
                 (this == common && tc >= pc + commonMaxSpares))
            throw new RejectedExecutionException(
                "Thread limit exceeded replacing blocked worker");
        else {                                // similar to tryAddWorker
            boolean add = false; int rs;      // CAS within lock
          	//增加线程总数tc.因为ac一加一减没变化。
            long nc = ((AC_MASK & c) |
                       (TC_MASK & (c + TC_UNIT)));
            if (((rs = lockRunState()) & STOP) == 0)
                add = U.compareAndSwapLong(this, CTL, c, nc);
            unlockRunState(rs, rs & ~RSLOCK);
          	//添加成功，就返回可以阻塞
            canBlock = add && createWorker(); // throws on exception
        }
    }
    return canBlock;
}
```

那么回到doJoin，如果当前线程不是ForkJoinWorkerThread，是外部线程「业务线程」。

需要externalAwaitDone()额外等待完成。

```java
private int externalAwaitDone() {
  	//对CountedCompleter的处理
    int s = ((this instanceof CountedCompleter) ? // try helping
             ForkJoinPool.common.externalHelpComplete(
                 (CountedCompleter<?>)this, 0) :
             //非CountedCompleter：尝试从栈顶获取任务，并执行。
             ForkJoinPool.common.tryExternalUnpush(this) ? doExec() : 0);
  	//如果任务不在栈顶，就阻塞等待。
    if (s >= 0 && (s = status) >= 0) {
        boolean interrupted = false;
      	//就是持续阻塞。
        do {
            if (U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                synchronized (this) {
                    if (status >= 0) {
                        try {
                            wait(0L);
                        } catch (InterruptedException ie) {
                            interrupted = true;
                        }
                    }
                    else
                        notifyAll();
                }
            }
        } while ((s = status) >= 0);
        if (interrupted)
            Thread.currentThread().interrupt();
    }
    return s;
}
```

