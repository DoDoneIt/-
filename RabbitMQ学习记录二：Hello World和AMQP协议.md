&emsp;&emsp;RabbitMQ作为一个基于[AMQP协议](http://blog.csdn.net/zhangxinrun/article/details/6411841)实现的消息中间件，其主要功能很简单：接收和分发消息。官网上举了一个很形象的邮局例子：MQ不仅像邮箱一样接收信件，还像邮递员一样把信件送到目的地。

&emsp;&emsp;AMQP的一些重要概念比如**route_key**，**exchange**，**virtual**等需要不断地理解和学习，贯穿MQ的三个基本概念：
**Producer：**生产者。生产者顾名思义就是不断地发送消息。
**Queue ：**队列。存储消息。
**Consumer：**消费者。接收消息并处理。

&emsp;&emsp;官网的 Get Started 提供了一个Hello World例子来入门。官网也提供了一个php客户端库--[php-amqplib](https://github.com/php-amqplib/php-amqplib)，这个库是官方推荐的。直接composer安装。

composer.json文件写上：

```php
{
  "require":{
     "php-amqplib/php-amqplib":"2.5.*"
}
```

然后cmd命令行中cd切换到目标目录，输入：

```php
$ composer.phar install
```

官网上也给出了代码例子，生产者send.php的代码：

```php
//引入类
require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

/**
 * 创建和MQ Server的连接，socket连接
 * 1、MQ提供了默认的guest用户和guest密码，以及'/'的vhost
 *   (1) guest用户只能通过localhost接入(详见https://www.rabbitmq.com/access-control.html)
 * 2、如果想通过IP和其他的用户名连接，可以在Management UI中的Admin功能中创建用户，并赋予administrator权限和vhost
 */
$connection = new AMQPStreamConnection(
                                    $host = 'localhost',
                                    $port = 5672,
                                    $user = 'guest',
                                    $password = 'guest',
                                    $vhost = '/');
/**
 * 创建管道，通过管道我们和MQ Server直接打交道,大部分的业务操作是在Channel这个接口中完成。
 * 包括定义Queue、定义Exchange、绑定Queue与Exchange、发布消息等
 */
$channel = $connection-&gt;channel();

/**
 * 指定一个队列$queue，同名队列如果声明多次但只会创建一次
 */
$channel->queue_declare($queue = 'test', false, false, false, false);

$info = array("time"=>time() ,"name"=>"demo");
$msg_str = json_encode($info);

/**
 * AMQPMessage的构造函数入参是string，我们需要序列化一下，这里用json
 */
$msg = new AMQPMessage($msg_str);

/**
 * 向队列中发送一条消息，声明exchange，默认类型是direct并绑定route_key，怎么操作见下文截图
 * exchange和route_key也是AMQP协议中的重要概念
 */
$channel->basic_publish($msg,$exchange='exchange_demo',$route_key='test_route_key');

echo $msg_str."\n"; 

/* 
 *闭管道和连接 
 */ 
$channel->close(); 
$connection->close(); 
```

消费者reveive.php 代码：

```php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

/*
 *和生产者代码一样
 */
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest','/');

/*
 * 创建直接处理的管道
 */
$channel = $connection-&gt;channel();

/*
 * 声明消费者队列,和发布者的队列名一样
 * 为了防止先启动消费者，当下文调用basic_consume方法时，如果MQ服务器上未声明过队列，就会抛出IO异常
 */
$queue = 'test';
$channel-&gt;queue_declare($queue, false, false, false, false);

echo ' [*] Waiting for messages. To exit press CTRL+C', &quot;\n&quot;;

/*
 * 指定从哪队列取消息，有消息就交个callback函数处理
 */
$channel-&gt;basic_consume($queue, '', false, true, false, false, $callback);

$callback = function($msg) {
    echo &quot; [x] Received &quot;, $msg-&gt;body, &quot;\n&quot;;
};

/*
 * 监听队列中的消息,如果没有消息，就处于阻塞状态
 */
while(count($channel-&gt;callbacks)) {
    $channel-&gt;wait();
}

$channel-&gt;close();
$connection-&gt;close();

```

生产者和消费者代码写好后，cli方式运行，如下图：
<div align="center">
    <img width="300" src="https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/12151.png"/>
</div>
成功运行后，我们进入Management后台，可以查看各项运行数据。

在send.php 中我们用了默认账号和密码，在后台中我们可以新增用户名、密码以及区分赋予vhost操作权限：

<div align="center">
    <img width="300" src="https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/3242.png"/>
</div>
所以connection那段，可以写成：

```php

$connection = new AMQPStreamConnection(
                                    $host = '192.168.0.11',
                                    $port = 5672,
                                    $user = 'test22',
                                    $password = 'test22',
                                    $vhost = '/');
```

在send.php 中我们也特殊指定了exchange和route_key，在代码运行之前，我们需要手动创建exchange并绑定route_key。
<div align="center">
    <img width="300" src="https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/exchange.png"/>
</div>
<div align=center><center>(1、在vhost下创建exchange)</center></div>
<div align="center">
    <img width="300" src="https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/route_key.png"/>
</div>
<div align=center><center>(2、在exchange里面绑定route_key)</center></div>
