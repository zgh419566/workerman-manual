# workerman/rabbitmq

workerman/rabbitmq是一个异步RabbitMQ客户端，使用AMQP协议。

## 项目地址：
https://github.com/walkor/rabbitmq

## 安装：
```
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/
composer require workerman/rabbitmq
```

## 示例


**receive.php**

```php
<?php
use Bunny\Channel;
use Bunny\Message;
use Workerman\Worker;
use Workerman\RabbitMQ\Client;

require __DIR__ . '/vendor/autoload.php';

$worker = new Worker();

$worker->onWorkerStart = function() {
    (new Client())->connect()->then(function (Client $client) {
        return $client->channel();
    })->then(function (Channel $channel) {
        return $channel->queueDeclare('hello', false, false, false, false)->then(function () use ($channel) {
            return $channel;
        });
    })->then(function (Channel $channel) {
        echo ' [*] Waiting for messages. To exit press CTRL+C', "\n";
        $channel->consume(
            function (Message $message, Channel $channel, Client $client) {
                echo " [x] Received ", $message->content, "\n";
            },
            'hello',
            '',
            false,
            true
        );
    });
};
Worker::runAll();
```

**send.php**
```php
<?php
use Bunny\Channel;
use Bunny\Message;
use Workerman\Worker;
use Workerman\RabbitMQ\Client;

require __DIR__ . '/vendor/autoload.php';

$worker = new Worker();

$worker->onWorkerStart = function() {
    (new Client())->connect()->then(function (Client $client) {
        return $client->channel();
    })->then(function (Channel $channel) {
        return $channel->queueDeclare('hello', false, false, false, false)->then(function () use ($channel) {
            return $channel;
        });
    })->then(function (Channel $channel) {
        echo " [x] Sending 'Hello World!'\n";
        return $channel->publish('Hello World!', [], '', 'hello')->then(function () use ($channel) {
            return $channel;
        });
    })->then(function (Channel $channel) {
        echo " [x] Sent 'Hello World!'\n";
        $client = $channel->getClient();
        return $channel->close()->then(function () use ($client) {
            return $client;
        });
    })->then(function (Client $client) {
        $client->disconnect();
    });
};
Worker::runAll();
```





官方的Rabbitmq示例教程很难对象操作有点多，我分享一个简单一些的操作
生产者、消费者都在一个程序中, producer and comsumer all in one

```php
require_once('./vendor/autoload.php');

use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;
use Workerman\Worker;
use Workerman\Lib\Timer;
use Workerman\Connection\TcpConnection;
use Workerman\Connection\AsyncUdpConnection;
use Workerman\Connection\AsyncTcpConnection;

$rabbit_connection = "";
$channel = "";

// 设置交换机名、路由键、队列名
$rabbit_conf = [
    'exchange_name' => 'upstream',
    'route_key'     => 'upstream',
    'queue_name'    => 'upstream',
];

$worker = new Worker();
//开启进程数量
$worker->count = 10;
$worker->name = "serviceName";

$worker->onWorkerStart = function () {
    global $rabbit_connection, $channel , $rabbit_conf;

    // 连接 rabbitmq 服务
    $rabbit_connection = new AMQPStreamConnection(RABBITMQ_SERVER_IP, RABBITMQ_SERVER_PORT, RABBITMQ_USERNAME, RABBITMQ_PASSWORD);

    // 获取信道
    $channel = $rabbit_connection->channel();

    // 声明队列 _CLASS_IN   _CLASS_IN
    $channel->queue_declare( $rabbit_conf['queue_name']."_CLASS_IN" , false, true, false, false);
    $channel->queue_declare( $rabbit_conf['queue_name']."_CLASS_OUT" , false, true, false, false);

    // 绑定队列 _CLASS_IN   _CLASS_OUT
    $channel->queue_bind($rabbit_conf['queue_name']."_CLASS_IN" , $rabbit_conf['exchange_name'], $rabbit_conf['route_key']."_CLASS_IN");
    $channel->queue_bind($rabbit_conf['queue_name']."_CLASS_OUT" , $rabbit_conf['exchange_name'], $rabbit_conf['route_key']."_CLASS_OUT");


    // 消费者订阅队列
    $channel->basic_consume($rabbit_conf['queue_name']."_CLASS_IN" , '', false, false, false, false,
        function ($msg){
            global $rabbit_connection,$channel , $rabbit_conf;
            $data_all_str = $msg->body;
            
            // 消息确认，表明已经收到这条信息
            $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
            
            //以下是业务逻辑代码    you code below
            //在一些业务逻辑中处理失败的任务需要继续送回到mq里面，为避免业务无限循环，建议对每个任务的重试次数进行限制，以本程序为例，可以给mq消息数组加个retryTime的限制
            $data_all = json_decode($data_all_str , true);
            
            //将数据发布到另外一个队列里面去   send data to another queue
            $data_all_out_json = json_encode($data_all , JSON_UNESCAPED_UNICODE );
            $data_all_out_msg = new AMQPMessage($data_all_out_json, ['content_type' => 'text/plain', 'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT]);
            $rabbit_channel->basic_publish($data_all_out_msg , $rabbit_conf['exchange_name'] , $rabbit_conf['route_key']."_CLASS_OUT");
        }
    );

    // 开始消费
    while (count($channel->callbacks)) {
        $channel->wait();
    }
};

Worker::runAll();
```
