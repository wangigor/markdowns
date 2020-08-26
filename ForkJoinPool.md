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



## 源码





