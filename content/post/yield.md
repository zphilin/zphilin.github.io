---
title: "yield生成器"
date: 2017-06-30T01:37:56+08:00
#lastmod: 2017-08-30T01:37:56+08:00
draft: false
tags: ["PHP","协程"]
categories: ["后台"]
author: "zphilin"

---
> yield is a Generator class is implementing Iterator

yield 是一个实现了迭代器的`生成器`，生成器多了一个send()方法，此方法和next() 类似也是恢复执行，但能从外部向生成器传入一个值，恢复执行直到下一个 yield。

# 程序执行流程

当一个函数中出现了yield关键字时，此时这个函数就变成了一个 `生成器` 
需要调用next或send迭代方法使其执行，且碰到yield就会中断让出执行权，并返回一个yield后的值，直到再次迭代到。

send方法传入的值将会被作为生成器当前所在的 [yield](http://php.net/manual/zh/language.generators.syntax.php#control-structures.yield) 的返回值；并自动恢复执行到下一个yield。

```php
<?php
 function gen() {
     $ret = (yield 1);
     var_dump($ret);
     $ret = (yield 2);
     var_dump($ret);
 }
 
 $gen = gen();
 echo PHP_EOL;
 var_dump($gen->current());    // int(1)
 echo PHP_EOL;
 var_dump($gen->send('senda')); 
 //   string(5) "senda" 生成器中第一个dump的输出
 //   int(2) 执行结果的输出，自动恢复执行到第二个yield
 echo PHP_EOL;
 var_dump($gen->send('sendb'));-
 //   string(5) "sendb"
 //   NULL 

```

# foreach 自动迭代

故所有生成器都可以用foreach来迭代，会自动调用迭代方法。

至于为什么叫生成器，可能就是因为在迭代的过程中能一直 `生成返回值` ，故每一次迭代总可以通过一定规则自动算出下一个值。

```php
<?php
function xrange($start, $end, $step = 1) {
    for ($i = $start; $i <= $end; $i += $step) {
        yield $i;
    }
}
 
foreach (xrange(1, 1000000) as $num) {
    echo $num, "\n";
}
```

# 协程

与进程对比，操作系统的进程调度，多个进程在一个 CPU 核心上执行，在系统调度下每一个进程执行一段指令就被暂停，切换到下一个进程，这样外部用户看起来就像是同时在执行多个任务。

协程可以理解为纯用户态的线程，通过协作而不是抢占来进行任务切换。相对于进程或者线程，协程所有的操作都可以在用户态而非操作系统内核态完成，创建和切换的消耗非常低。

操作系统在一个进程或线程发生IO时阻塞就会切换到另一个进程，而这里相当于程序自己发生阻塞时自己切换到其他任务，过一会再来，如此往返重复执行。

故需要我们把大任务拆分成多个小任务轮流执行，如果有某个小任务在等待系统 IO，就跳过它，执行下一个小任务，这样往复调度，实现了 IO 操作和 CPU 计算的并行执行，总体上就提升了任务的执行效率。

Go 协程意味着并行（或者可以以并行的方式部署，可以用 runtime.GOMAXPROCS() 指定可同时使用的 CPU 个数），协程一般来说只是并发。

```php
<?php
class Task
{
    protected $taskId;
    protected $coroutine;
    protected $beforeFirstYield = true;
    protected $sendValue;

    /**
     * Task constructor.
     * @param $taskId
     * @param Generator $coroutine
     */
    public function __construct($taskId, Generator $coroutine)
    {
        $this->taskId = $taskId;
        $this->coroutine = $coroutine;
    }

    /**
     * 获取当前的Task的ID
     * 
     * @return mixed
     */
    public function getTaskId()
    {
        return $this->taskId;
    }

    /**
     * 判断Task执行完毕了没有
     * 
     * @return bool
     */
    public function isFinished()
    {
        return !$this->coroutine->valid();
    }

    /**
     * 设置下次要传给协程的值，比如 $id = (yield $xxxx)，这个值就给了$id了
     * 
     * @param $value
     */
    public function setSendValue($value)
    {
        $this->sendValue = $value;
    }

    /**
     * 运行任务
     * 
     * @return mixed
     */
    public function run()
    {
        // 这里要注意，生成器的开始会reset，所以第一个值要用current获取
        if ($this->beforeFirstYield) {
            $this->beforeFirstYield = false;
            return $this->coroutine->current();
        } else {
            // 我们说过了，用send去调用一个生成器
            $retval = $this->coroutine->send($this->sendValue);
            $this->sendValue = null;
            return $retval;
        }
    }
}


Class Scheduler
{
    /**
     * @var SplQueue
     */
    protected $taskQueue;
    /**
     * @var int
     */
    protected $tid = 0;

    /**
     * Scheduler constructor.
     */
    public function __construct()
    {
        /* 原理就是维护了一个队列，
         * 前面说过，从编程角度上看，协程的思想本质上就是控制流的主动让出（yield）和恢复（resume）机制
         * */
        $this->taskQueue = new SplQueue();
    }

    /**
     * 增加一个任务
     *
     * @param Generator $task
     * @return int
     */
    public function addTask(Generator $task)
    {
        $tid = $this->tid;
        $task = new Task($tid, $task);
        $this->taskQueue->enqueue($task);
        $this->tid++;
        return $tid;
    }

    /**
     * 把任务进入队列
     *
     * @param Task $task
     */
    public function schedule(Task $task)
    {
        $this->taskQueue->enqueue($task);
    }

    /**
     * 运行调度器
     */
    public function run()
    {
        while (!$this->taskQueue->isEmpty()) {
            // 任务出队
            $task = $this->taskQueue->dequeue();
            $res = $task->run(); // 运行任务直到 yield

            if (!$task->isFinished()) {
                $this->schedule($task); // 任务如果还没完全执行完毕，入队等下次执行
            }
        }
    }
}


function task1() {
    for ($i = 1; $i <= 10; ++$i) {
        echo "This is task 1 iteration $i.\n";
        yield; // 主动让出CPU的执行权
    }
}

function task2() {
    for ($i = 1; $i <= 5; ++$i) {
        echo "This is task 2 iteration $i.\n";
        yield; // 主动让出CPU的执行权
    }
}

$scheduler = new Scheduler; // 实例化一个调度器
$scheduler->addTask(task1()); // 添加不同的闭包函数作为任务
$scheduler->addTask(task2());
$scheduler->run();


//要求将同步程序拆分成异步程序，即阻塞的过程本身还是存在，但是不等待这个阻塞二是让出执行权
function task3() {
    /* 这里有一个远程任务，需要耗时10s，可能是一个远程机器抓取分析远程网址的任务，我们只要提交最后去远程机器拿结果就行了 */
    remote_task_commit();
    // 这时候请求发出后，我们不要在这里等，主动让出CPU的执行权给task2运行，他不依赖这个结果
    yield;
    yield (remote_task_receive());
}   

function task4() {
    for ($i = 1; $i <= 5; ++$i) {
        echo "This is task 2 iteration $i.\n";
        yield; // 主动让出CPU的执行权
    }   
}
```

