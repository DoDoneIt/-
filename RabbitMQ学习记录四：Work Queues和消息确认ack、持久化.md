
#### 消息持久化

在我们熟悉的redis中，有持久化机制，保证server挂掉之后缓存不丢失，而在RabbitMQ中同样支持持久化：队列和消息均持久化，才能保证重启时候，队列和消息存在！需要在消费者和生产者都声明**队列持久化**，在生产者声明**消息持久化**

生产者new_task.php

[php]

/*
 * 声明队列持久化
 */
$channel-&gt;queue_declare($queue_name,false,$durable = true,false,false);
/*
 * 声明消息delivery_mode=2
 *http://stackoverflow.com/questions/2344022/what-is-the-delivery-mode-in-amqp 
 */ 
$data = $argv['1'];
$msg = new AMQPMessage($data, array('delivery_mode' =&gt; 2)); 

[/php]

<h4><span style="color: #ff0000;">消息确认ack</span></h4>
在上一篇消息分发机制中，我们提到了MQ<a href="https://www.rabbitmq.com/confirms.html">消息确认</a>ack：

为了公平分发消息，保证消费者消费消息不会阻塞也不会空闲，除非消费者处理并确认了前一个消息，否则MQ不会向其分派新消息。
<blockquote>In order to defeat that we can use the basic_qos method with the prefetch_count = 1 setting. This tells RabbitMQ not to give more than one message to a worker at a time. Or, in other words, don't dispatch a new message to a worker until it has processed and acknowledged the previous one. Instead, it will dispatch it to the next worker that is not still busy.</blockquote>
消息确认机制除了保证公平分发之外，还保证了消息不丢失。

<!--more-->

开启消息持久化可以保证server挂掉之后，消息和队列还存在，但是如果消费者脚本挂了话，怎么办？默认情况下MQ将消息分发给consumer之后，就会从内存中删除。如果脚本挂掉，那么消费者正在处理还未完成的消息会丢失，已经分发给消息者尚未处理的消息也会丢失。

为了确保消费者脚本挂掉消息不丢失，MQ支持消息确认。

消费者发送一个消息应答，告诉RabbitMQ这个消息已经接收并且处理完毕了，RabbitMQ可以删除它了。

如果一个消费者挂掉却没有发送应答，RabbitMQ会理解为这个消息没有处理完全，然后交给另一个消费者去重新处理。这样，你就可以确认即使消费者偶尔挂掉也不会不丢失任何消息了。在amqplib库中消息确认是默认打开的。

这里并没有用到超时机制。RabbitMQ仅仅通过Consumer的连接中断来确认该Message并没有被正确处理。也就是说，RabbitMQ给了Consumer足够长的时间来做数据处理。

通过消费者代码来看下new_work.php：

[php]

$callback = function($msg){
    $str= $msg-&gt;body;&lt;/pre&gt;
   try{
      //...do something
      //ack 向队列确认消息已经接收并处理完成
      //处理完消息后要返回ack，表示已成功处理，否则服务器将不会删除该消息，内存很快被吃掉
      $msg-&gt;delivery_info['channel']-&gt;basic_ack($msg-&gt;delivery_info['delivery_tag']);
    }catch (Exception $e){
      /*
       * 同时：如果脚本出现异常，比如数据库，语法之类的，没法ack消息，我们需要拒收消息，并通知MQ是否重新入队
       */
      $msg-&gt;delivery_info['channel']-&gt;basic_reject($msg-&gt;delivery_info['delivery_tag'] , $requeue = true);
    }
};

/* 
*保证每次只发发送不超过一个消息给消费者 
*在消费者处理并确认消息之前，不要发送新消息 
*$prefetch_count参数必须为1 
*/ 
$channel-&gt;basic_qos(null, $prefetch_count = 1, null);
/* 
* $no_ack参数需要设置为false，开启消费者消息确认 
*/ 
$channel-&gt;basic_consume($queue_name, '', true, $no_ack=false, false, false, $callback); 

[/php]

对于手动ack的队列，如果有多个消费者同时处理时会怎么样呢？

一般情况下是依次循环（round-robin）的往各个消费者放数据的，比如两个消费者，则a分别放1,3,5而b分别放2,4,6...

那假如消费者a先启动，消费者b后启动呢，由或者queue中已经有大量数据了呢？

当消费者a读到一定量的数据后(只是放在缓存中，并没有处理消息)，消费者b启动后先看看mq中是否还剩数据（剩余=原数据 - 已ack数据 - 其他consumer拿到并缓存但还未ack），有则取之，没有等待。若此时消费者a挂了，消费者b能拿到剩余未ack的消息。由此看来，一个消费者拿到的数据不一定是顺序的。

<a href="http://www.dada163.com/wp-content/uploads/2017/01/QQ截图20170124131551.png"><img class="aligncenter wp-image-1213 size-full" title="点击图片放大" src="http://www.dada163.com/wp-content/uploads/2017/01/QQ截图20170124131551.png" width="1143" height="650" /></a>

&nbsp;

<a href="http://www.dada163.com/wp-content/uploads/2017/01/QQ截图20170124142611.png"><img class="aligncenter size-full wp-image-1216" src="http://www.dada163.com/wp-content/uploads/2017/01/QQ截图20170124142611.png" alt="QQ截图20170124142611" width="667" height="620" /></a>

<a href="http://www.dada163.com/wp-content/uploads/2017/01/QQ截图20170124132130.png"><img class="aligncenter size-full wp-image-1220" src="http://www.dada163.com/wp-content/uploads/2017/01/QQ截图20170124132130.png" alt="QQ截图20170124132130" width="780" height="438" /></a>

&nbsp;

<strong>总结：</strong>
简单的队列模型中，生产者生产消息，消费者消费消息，两者之间通过exchange，channel，building，queue_name绑定连接。
考虑到不同消费者不同的消费能力，MQ遵循消息轮询，消息公平性分发机制，保证消费者能顺序拿消息，又不至于消息积压。
为了保证消息不丢失，MQ在server层面可以设置队列和消息持久化，在client端的消费者开启消息确认ack。

参考：
https://www.rabbitmq.com/confirms.html

https://www.zhihu.com/question/41976893

http://backend.blog.163.com/blog/static/2022941262014315104836716/

http://backend.blog.163.com/blog/static/202294126201322511327882/

https://my.oschina.net/moooofly/blog/92259

http://stackoverflow.com/questions/40503700/rabbitmq-durability