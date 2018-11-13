在上一篇中，我们跑起了一个Hello World：基本的producer和consumer。跟着[官网的Tutorials](https://www.rabbitmq.com/getstarted.html)教程，我们一步步来。那些深奥的协议和原理，也不需要着急全理解，MQ的集群和高可用也需要把这些基础敲一遍。

任务队列的应用场景往往是把耗时的同步操作放到后台异步执行，比如：短信发送，邮件通知，状态推送等等。。官网上给出了一个Work Queues教程来模拟耗时任务消息的发送和接收处理。

[任务发送(生产者)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/new_task.php)new_task.php

```php

//连接
$connection = new AMQPStreamConnection($host = 'localhost',$port = 5672,$user = 'guest',$password = 'guest',$vhost = '/');
//创建channel
$channel = $connection->channel();
//声明队列
$queue_name = "task_name";
$channel->queue_declare($queue_name,false,true,false,false);
/*
 * 官网上通过输入参数中有多少点号,作为consumer的worker.php执行sleep多少秒
 */
$data = $argv['1'];

$msg = new AMQPMessage($data,
                       array('delivery_mode' =&gt; 2) //消息持久化
                     );

$channel->basic_publish($msg, '', $queue_name);

//关闭...

```

[消息处理(消费者)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/worker.php)worker.php

```php

//连接
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();
//声明队列
$queue_name = "task_name";
$channel->queue_declare($queue_name, false, true, false, false);
echo ' [*] Waiting for messages. To exit press CTRL+C', "\n";
//回调处理
$callback = function($msg){
    echo " [x] Received", $msg->body;
    usleep(substr_count($msg->body, '.'));
    echo " [x] Done";
    //ack 主动确认消息
    $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
};
//在消费者返回ack之前，不要发送新消息
$channel->basic_qos(null, 1, null);
/*
 * $no_ack参数需要设置为false，开启消费者消息确认
 */
$channel->basic_consume($queue_name, '', false, $no_ack=false, false, false, $callback);

//等待...
//关闭...

```

#### 轮询分发Round-robin

有这样一个场景：如图，A系统的p不断朝MQ里面塞消息，B系统开启C1和C2两个脚本进程进行消费消息，那么MQ是怎么把消息分发给不同的消费者呢？

<div align="center">
<img class="aligncenter" src="https://www.rabbitmq.com/img/tutorials/python-two.png" width="332" height="111" />
</div>

RabbitMq遵循的原理是：在默认情况下，RabbitMQ将逐个发送消息到在序列中的下一个消费者(而不考虑每个任务的时长等等，且是提前一次性分配，并非一个一个分配)。平均每个消费者获得相同数量的消息。这种方式分发消息机制称为Round-Robin（轮询），像顺序投食一样。

<div align="center">
    <img width="500" src="https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/work-1-1.png"/>
</div>

#### 公平分发Fair

既然MQ轮询分发，而且是只要有消息进入队列就会投递，而不管消费者应答，仅仅简单地把第n条数据投递给第n个消费者，但是如果有的消费者处理能力强能很快地处理完消息，而有的弱，这就会导致一个问题：能力弱的消费者消息积压，服务器一直繁忙，而能力强的消费者一直处于空闲。

> This happens because RabbitMQ just dispatches a message when the message enters the queue. It doesn't look at the number of unacknowledged messages for a consumer. It just blindly dispatches every n-th message to the n-th consumer.


为了解决这个问题，我们在消费者脚本中添加一个方法：

```php

/*
 *保证每次只发送不超过一个消息给消费者
 *在消费者处理并确认消息之前，不要发送新消息
 *$prefetch_count参数必须为1
 */
$channel->basic_qos(null, $prefetch_count = 1, null);

```
<div align="center">
<img class="aligncenter" src="https://www.rabbitmq.com/img/tutorials/prefetch-count.png" width="396" height="111" />
</div>

写个脚本测试一下：

new_worker.php中的callback函数特定处理模拟耗时任务：

```php

$callback = function($msg){
    $str= $msg->body;
    echo "[x] Received ".$str;
    if($str == 'task'){//模拟耗时任务
        sleep(1);
    }else{
        usleep(2);
    }
    echo " [x] Done";
    //ack 向队列确认消息已经接收并处理完成
    $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
};

```

<div align="center">
    <img width="500" src="https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/05706-1.png"/>
</div>


参考：

https://www.rabbitmq.com/tutorials/tutorial-two-php.html

https://www.kancloud.cn/longxuan/rabbitmq-arron/117513    