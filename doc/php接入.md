# pepper_bus php 接入


## 测试环境接入

> 测试环境后台：`http://10.209.221.224`，使用花椒域账号/密码登录

## 线上环境

> http://inner.pepperbus.huajiao.com/

### 添加系统

> URL: `http://10.209.221.224/#/system/index`

pepper_bus 资源隔离使用了系统的概念，不同系统下的存储、服务器资源互相隔离。在该页面选择添加系统，根据提示添加对应系统

### 添加服务器

> URL: `http://10.209.221.224/#/cluster/index`

当前服务器采用了 pepper\_bus + php-fpm 的部署方式，即服务器是运行 pepper\_bus 的节点，同时也是业务 worker 机节点。

在集群管理页面点击添加，选择对应的系统，测试环境节点使用 `10.138.114.222:19840`

### 添加存储

> URL: `http://10.209.221.224/#/storage/index`

存储是指 worker 数据落地的 redis。点击添加并选择对应的系统，测试环境 redis 使用以下配置：

```
IP: 10.142.103.50
端口：1336
密码：732b5c867745ae80
```

连接参数，如最大连接数、最大空闲连接数及空闲时间如无特殊需求不用填写，使用默认值即可。

### 添加队列

> URL: `http://10.209.221.224/#/queue/index`

队列是指消息生产者写入需要使用的配置。在队列页面点击 `添加 Queue`，按页面提示填写队列信息，其中队列认证是指生产者写入需要的密钥，测试环境可以统一使用 `783ab0ce`。

### 添加 topic

> URL: `http://10.209.221.224/#/queue/index` 或 `http://10.209.221.224/#/system/index`

可以在系统或队列页面内点击 `添加 Topic` 新增一个 topic。

同一个队列可以由多个 topic 消费。

在添加 Topic 窗口选择对应的系统、队列和存储，对于 php worker，运行类型选择 `CGI 类型`，消费文件填写对应的 php 在服务器上的绝对路径，如：`/home/q/system/pepper/live/front/src/worker/msg/ReplyMsgPushWorker.php`。

### php sdk 接入

* 通过 composer 引用 pepper_bus 的 SDK: `"pepper/bus": "~1.0"`

* 在业务代码的配置文件 `server_conf.test.php` 中加入测试环境的 pepper_bus 配置：

```
// pepperBus的配置
$BUS_CONF = [
    "password" => "783ab0ce",
    "servers" => [
        [ "host" => "10.209.221.190", "port" => 6379, "timeout" => 3],
    ]
];
```

### 生产者接入

> 生产者指队列消息产生方，即原 process_client 的 `ProcessClient::getInstance("huajiao")->addTask()` 的调用方

将 addTask 的调用更换为如下方式：
`PepperBus::getInstance()->addJob(<queue_name>, <queue_data>);`

其中，`<queue_name>` 是添加队列时输入的队列名称，`<queue_data>` 是队列数据，保持原 addTask 的第二个参数不变。

### 消费者接入

> 消费者即原 process_client 的 `(new ProcessClient("huajiao"))->addWorker()` 调用方。

将 addWorker 的调用做如下更新，注册 topic 消费到 pepper_bus：
`PepperBus::getInstance()->register(<queue_name>, <topic_name>, <class_name>, <function_name>)->do();`

其中：`<queue_name>` 是队列名称，`<topic_name>` 指 topic 名称。

### 代码发布及测试

#### 发布消费者（worker）

配置测试环境 worker 发布系统，添加 `10.208.65.21`，将 worker 代码发到测试环境

#### 测试 worker

在后台系统或队列页面点击 topic，进入 topic 页面，点击测试，向 worker 手动发送测试数据并检查返回值，200 为成功，常见错误有：

```
404: worker topic 未注册
500: 详情看 msg，可能原因有 php worker 文件配置错误或不存在
502: fpm 连接错误，请联系陆游或郭哲林
```

#### 发布生产者

将队列写入代码发布到测试环境

#### 后台观察

* pepper_bus 会将 worker 运行中报错的消息放到 retry 队列中。

* 代码发布完成后，在后台 topic 页面观察对应 topic 的消费情况，如果有消息进入到 retry 队列可以查看 topic 报错，方便业务方调度。

* 对于数据敏感型业务，在修复后可以将 restry 队列中的消息重放。

## 上线

TODO

### 注意事项

* 线上 mysql 需要后台添加授权，检查所有用到的数据库实例
*