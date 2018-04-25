##### 往往开启多进程的目的是为了一起干活加速效率，前面说了不同进程之间的内存空间都是相互隔离的，也就说进程A是无法读或写进程B中的任何数据内容的，反之亦然。但是，有些时候，多个进程之间必须要有相互通知的机制，用职场上的话来说就叫“及时沟通”。大家都在一起做同一件事情的不同部分，彼此之间“及时沟通”是很重要的。
##### 于是进程间通信就诞生了，英文缩写IPC，全称InterProcess Communication。
##### 常见的进程间通信方式有：管道（分无名和有名两种）、消息队列、信号量、共享内存和socket，最后一种方式今天不提，放到后面的php socket编程中去说，重点说前四种方式。
##### 管道是*NIX上常见的一个东西，大家平时使用linux的时候也都在用，简单理解就是|，比如ps -aux|grep php这就是管道，大概意思类似于ps进程和grep进程两个进程之间用|完成了通信。管道是一种半双工（现在也有系统已经支持全双工的管道）的工作方式，也就是说数据只能沿着管道的一个方向进行传递，不可以在同一个管道上反向传数据。管道分为两种，一种叫做未命名的管道，另一种叫做命名管道，未命名管道只能在拥有公共祖先的两个进程之间使用，简单理解就是只能用于父进程和和其子进程之间的通信，但是命名管道则可以用于任何两个毫无关连的进程之间的通信（一会儿将要演示的将是这种命名管道）。
##### 需要特殊指出的是消息队列、信号量和共享内存这三种IPC同属于XSI IPC（XSI可以认为是POSIX标准的超集，简单粗暴理解为C++之于C）。这三种IPC在*NIX中一般都有两个“名字”来为其命名，一个叫做标志符，一个叫做键（key）。标志符是一个非负整数，每当一个IPC结构被创建然后又被销毁后，标志符便会+1一直加到整数的最大整数数值，而后又从0开始重新计算。既然是为了多进程通信使用，那么多进程在使用XSI IPC的时候就需要使用一个名字来找到相应的IPC，然后才能对其进行读写（术语叫做多个进程在同一个IPC结构上汇聚），所以POSIX建议是无论何时创建一个IPC结构，都应指定一个键（key）与之关联。一句话总结就是：标志符是XSI IPC的内部名称，键（key）是XSI IPC的外部名称。
##### 使多个进程在XSI IPC上汇聚的方法大概有如下三种：
-  使用指定键IPC_PRIVATE来创建一个IPC结构，然后将返回的标志符保存到一个文件中，然后进程之间通过读取这个文件中的标志符进行通信。使用公共的头文件。这么做的缺点是多了IO操作。
- 将共同认同的键写入到公共头文件中。这么做的缺点这个键可能已经与一个IPCi结构关联，这样在使用这个键创建结构的时候就可能会出错，然后必须删除已有的IPC结构再重新创建。
- 认同一个文件路径名和项目ID，然后使用ftok将这两个参数转换成一个键。这将是我们使用的方式。

##### XSI IPC结构有一个与之对应的权限结构，叫做ipc_perm，这个结构中定义了IPC结构的创建者、拥有者等。

##### 多进程通信之一：命名管道。 在php中，创建一个管道的函数叫做posix_mkfifo()，管道创建完成后其实就是一个文件，然后就可以用任何与读写文件相关的函数对其进行操作了，代码大概演示一下：
```php
<?php
// 管道文件绝对路径
$pipe_file = __DIR__.DIRECTORY_SEPARATOR.'test.pipe';
// 如果这个文件存在，那么使用posix_mkfifo()的时候是返回false，否则，成功返回true
if( !file_exists( $pipe_file ) ){
  if( !posix_mkfifo( $pipe_file, 0666 ) ){
    exit( 'create pipe error.'.PHP_EOL );
  }
}
// fork出一个子进程
$pid = pcntl_fork();
if( $pid < 0 ){
  exit( 'fork error'.PHP_EOL );
} else if( 0 == $pid ) {
  // 在子进程中
  // 打开命名管道，并写入一段文本
  $file = fopen( $pipe_file, "w" );
  fwrite( $file, "helo world." );
  exit;
} else if( $pid > 0 ) {
  // 在父进程中
  // 打开命名管道，然后读取文本
  $file = fopen( $pipe_file, "r" );
  // 注意此处fread会被阻塞
  $content = fread( $file, 1024 );
  echo $content.PHP_EOL;
  // 注意此处再次阻塞，等待回收子进程，避免僵尸进程
  pcntl_wait( $status );
}
```
##### 运行结果如下：
![](http://static.ti-node.com/6381869827836870657)

##### 多进程通信之二：消息队列。这个怕是很多人都听过，不过印象往往停留在kafka、rabbitmq之类的用于服务器解耦网络消息队列软件上。消息队列是消息的链接表（一种常见的数据结构），但是这种消息队列存储于系统内核中（不是用户态），一般我们外部程序使用一个key来对消息队列进行读写操作。在PHP中，是通过msg_*系列函数完成消息队列操作的。
```php
<?php
// 使用ftok创建一个键名，注意这个函数的第二个参数“需要一个字符的字符串”
$key = ftok( __DIR__, 'a' );
// 然后使用msg_get_queue创建一个消息队列
$queue = msg_get_queue( $key, 0666 );
// 使用msg_stat_queue函数可以查看这个消息队列的信息，而使用msg_set_queue函数则可以修改这些信息
//var_dump( msg_stat_queue( $queue ) );  
// fork进程
$pid = pcntl_fork();
if( $pid < 0 ){
  exit( 'fork error'.PHP_EOL );
} else if( $pid > 0 ) {
  // 在父进程中
  // 使用msg_receive()函数获取消息
  msg_receive( $queue, 0, $msgtype, 1024, $message );
  echo $message.PHP_EOL;
  // 用完了记得清理删除消息队列
  msg_remove_queue( $queue );
  pcnlt_wait( $status );
} else if( 0 == $pid ) {
  // 在子进程中
  // 向消息队列中写入消息
  // 使用msg_send()向消息队列中写入消息，具体可以参考文档内容
  msg_send( $queue, 1, "helloword" );
  exit;
}
```
##### 运行结果如下：
![](http://static.ti-node.com/6382072383888424961)
##### 但是，值得大家继续深入研究的是msg_send()和msg_receive()两个函数，这两个的每一个参数都是非常值得深入研究和尝试的。篇幅问题，这里就不再详细描述。

##### 多进程通信之三：信号量与共享内存。共享内存是最快是进程间通信方式，因为n个进程之间并不需要数据复制，而是直接操控同一份数据。实际上信号量和共享内存是分不开的，要用也是搭配着用。*NIX的一些书籍中甚至不建议新手轻易使用这种进程间通信的方式，因为这是一种极易产生死锁的解决方案。共享内存顾名思义，就是一坨内存中的区域，可以让多个进程进行读写。这里最大的问题就在于数据同步的问题，比如一个在更改数据的时候，另一个进程不可以读，不然就会产生问题。所以为了解决这个问题才引入了信号量，信号量是一个计数器，是配合共享内存使用的，一般情况下流程如下：
- 当前进程获取将使用的共享内存的信号量
- 如果信号量大于0，那么就表示这块儿共享资源可以使用，然后进程将信号量减1
- 如果信号量为0，则进程进入休眠状态一直到信号量大于0，进程唤醒开始从1

##### 一个进程不再使用当前共享资源情况下，就会将信号量减1。这个地方，信号量的检测并且减1是原子性的，也就说两个操作必须一起成功，这是由系统内核来实现的。
##### 在php中，信号量和共享内存先后一共也就这几个函数：
![](http://static.ti-node.com/6382081950860967937)
##### 其中，sem_*是信号量相关函数，shm_*是共享内存相关函数。
```php
<?php
// sem key
$sem_key = ftok( __FILE__, 'b' );
$sem_id = sem_get( $sem_key );
// shm key
$shm_key = ftok( __FILE__, 'm' );
$shm_id = shm_attach( $shm_key, 1024, 0666 );
const SHM_VAR = 1;
$child_pid = [];
// fork 2 child process
for( $i = 1; $i <= 2; $i++ ){
  $pid = pcntl_fork();
  if( $pid < 0 ){
    exit();
  } else if( 0 == $pid ) {
	// 获取锁
	sem_acquire( $sem_id );
	if( shm_has_var( $shm_id, SHM_VAR ) ){
	  $counter = shm_get_var( $shm_id, SHM_VAR );
	  $counter += 1;
	  shm_put_var( $shm_id, SHM_VAR, $counter );
	} else {
	  $counter = 1;
	  shm_put_var( $shm_id, SHM_VAR, $counter );
	}
	// 释放锁，一定要记得释放，不然就一直会被阻锁死
	sem_release( $sem_id );
	exit;
  } else if( $pid > 0 ) {
    $child_pid[] = $pid;
  }
}
while( !empty( $child_pid ) ){
  foreach( $child_pid as $pid_key => $pid_item ){
    pcntl_waitpid( $pid_item, $status, WNOHANG );
	unset( $child_pid[ $pid_key ] );
  }
}
// 休眠2秒钟，2个子进程都执行完毕了
sleep( 2 );
echo '最终结果'.shm_get_var( $shm_id, SHM_VAR ).PHP_EOL;
// 记得删除共享内存数据，删除共享内存是有顺序的，先remove后detach，顺序反过来php可能会报错
shm_remove( $shm_id );
shm_detach( $shm_id );
```
##### 运行结果如下：
![](http://static.ti-node.com/6382157565437935617)
##### 确切说，如果不用sem的话，上述的运行结果在一定概率下就会产生1而不是2。但是只要加入sem，那就一定保证100%是2，绝对不会出现其他数值。
---
