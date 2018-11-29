# JMH概述   
>JMH 是一个由 OpenJDK/Oracle 里面那群开发了 Java 编译器的大牛们所开发的 Micro Benchmark Framework 。何谓 Micro Benchmark 呢？简单地说就是在 method 层面上的 benchmark，精度可以精确到微秒级。可以看出 JMH 主要使用在当你已经找出了热点函数，而需要对热点函数进行进一步的优化时，就可以使用 JMH 对优化的效果进行定量的分析。

# 典型的使用场景
1. 想定量地知道某个函数需要执行多长时间，以及执行时间和输入 n 的相关性   
2. 一个函数有两种不同实现(例如实现 A 使用了 FixedThreadPool，实现 B 使用了 ForkJoinPool)，不知道哪种实现性能更好

# 典型用法和部分常用选项
```java
package normaltest;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Thread)
public class JMHFirstBenchmark {
    @Benchmark//对要被测试性能的代码添加注解，说明该方法是要被测试性能的
    public int sleepAWhile() {
        try {
            Thread.sleep(50);
        } catch (InterruptedException e) {
            // ignore
        }
        return 0;
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(JMHFirstBenchmark.class.getSimpleName())
                .forks(1)
                .warmupIterations(3)
                .measurementIterations(3)
                .build();

        new Runner(opt).run();
    }

}
```
```
# JMH version: 1.19
# VM version: JDK 1.8.0_152, VM 25.152-b16
# VM invoker: C:\Program Files\Java\jdk1.8.0_152\jre\bin\java.exeIDEA 2017.3.5\bin -Dfile.encoding=UTF-8
# Warmup: 3 iterations, 1 s each
# Measurement: 3 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: cn.fexo.JMHFirstBenchmark.sleepAWhile

# Run progress: 0.00% complete, ETA 00:00:06
# Fork: 1 of 1
# Warmup Iteration   1: 50924.558 us/op
# Warmup Iteration   2: 50960.144 us/op
# Warmup Iteration   3: 50927.677 us/op
Iteration   1: 50961.230 us/op
Iteration   2: 50963.006 us/op
Iteration   3: 50963.282 us/op
```

对 sleepAWhile() 的测试结果显示执行时间平均约为50毫秒。因为我们的测试对象 sleepAWhile() 正好就是睡眠50毫秒，所以 JMH 显示的结果可以说很符合我们的预期。

基本概念：
Mode 
Mode 表示 JMH 进行 Benchmark 时所使用的模式。通常是测量的维度不同，或是测量的方式不同。目前 JMH 共有四种模式：
(1).Throughput: 整体吞吐量，例如“1秒内可以执行多少次调用”。
(2).AverageTime: 调用的平均时间，例如“每次调用平均耗时xxx毫秒”。
(3).SampleTime: 随机取样，最后输出取样结果的分布，例如“99%的调用在xxx毫秒以内，99.99%的调用在xxx毫秒以内”
(4).SingleShotTime: 以上模式都是默认一次 iteration 是 1s，唯有 SingleShotTime 是只运行一次。往往同时把 warmup 次数设为0，用于测试冷启动时的性能。
Interation 
Iteration是JMH进行测试的最小单位。大部分模式下，iteration代表的是一秒，JMH会在这一秒内不断调用需要benchmark的方法，然后根据模式对其采样，计算吞吐量，计算平均执行时间等。
Warmup 
Warmup是指在实际进行Benchmark前先进行预热的行为。因为JVM的JIT机制的存在，如果某个函数被调用多次以后，JVM会尝试将其编译成为机器码从而提高执行速度。所以为了让benchmark的结果更加接近真实情况就需要进行预热。

注解
现在来解释一下上面例子中使用到的注解，其实很多注解的意义完全可以望文生义 :)
@Benchmark 
表示该方法是需要进行 benchmark 的对象，用法和 JUnit 的 @Test 类似。
@Mode
Mode 如之前所说，表示 JMH 进行 Benchmark 时所使用的模式。
@State
State 用于声明某个类是一个“状态”，然后接受一个 Scope 参数用来表示该状态的共享范围。因为很多 benchmark 会需要一些表示状态的类，JMH 允许你把这些类以依赖注入的方式注入到 benchmark 函数里。Scope 主要分为两种。

(1).Thread: 该状态为每个线程独享。
(2).Benchmark: 该状态在所有线程间共享。
关于State的用法，官方的 code sample 里有比较好的例子。

@OutputTimeUnit
benchmark 结果所使用的时间单位。

启动选项

解释完了注解，再来看看 JMH 在启动前设置的参数。

Options opt = new OptionsBuilder()
        .include(FirstBenchmark.class.getSimpleName())
        .forks(1)
        .warmupIterations(5)
        .measurementIterations(5)
        .build();
new Runner(opt).run();
benchmark 所在的类的名字，注意这里是使用正则表达式对所有类进行匹配的。

fork
进行 fork 的次数。如果 fork 数是2的话，则 JMH 会 fork 出两个进程来进行测试。

warmupIterations
预热的迭代次数。

measurementIterations
实际测量的迭代次数。
