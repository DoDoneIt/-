随着业务的快速发展，系统越来越多，而一个好的业务系统架构讲究：高内聚低耦合，即尽量减弱对外部系统的依赖。举个例子：
一个典型的电商系统，对外有：PC网站、M站、APP、微信，对内可能会有：CMS、营销、CRM、订单、产品、供应商等。
对于复杂的系统，我们一般通过：消息队列、RPC、发布/订阅各种模型来解决系统间的解耦和数据通信。对于这些，最常用的一种开源消息中间件就是：RabbitMQ：

```
RabbitMQ是使用Erlang语言开发的开源消息队列系统，基于AMQP协议来实现。
AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。
AMQP协议更多用在企业系统内，对数据一致性、稳定性和可靠性要求很高的场景，对性能和吞吐量的要求还在其次。
-阿里中间件技术博客
```

#### MQ的安装

MQ安装很简单，在[官网](https://www.rabbitmq.com/download.html)上有各个系统的安装教程，而Windows下载安装包后直接安装。以下的介绍均在Windows环境。

安装包括一个Server和一个Management UI 后台。Windows 的 Server 提供了一个RabbitMQ Command 工具，这个CMD工具很好用。

![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/20161202155049.png)

<center>CMD工具</center>

![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/20161202154410.png)

<center>Management  UI</center>

Management  UI 默认是使用15672端口，默认用户名guest和密码guest，为了安全，guest账号只能通过http://localhost:15672/登录，可供操作的资源也被限制。
Management 方便管理MQ。比如查看客户端连接情况，可以查看消息推送消费的速率，磁盘写入和读取速率，新增QUEUE，设置Durability，查看/添加MESSAGES，添加Exchange，绑定Routing key和Queue.....
对于出现的每个名称，Management都给出了解释，建议细看：
![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/20161227113113.png)

<center>Management  UI 注释功能</center>

如果初识MQ，这种单机版的Server就行了。