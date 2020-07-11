---
title: 我所理解的接口设计
date: 2018-03-13 22:51:23
tags: api
---

# 前言

自己做接口开发的时间也算不短了(三年)，想写这篇文章其实差不多已经有一年多的时间了。我将从下面的方向来对我所理解的接口设计做个总结：

> 接口参数定义 -> 接口版本化的问题 -> 接口的安全性 -> 接口的代码设计 -> 接口的可读性 -> 接口文档 -> 我遇到的坑

# 接口参数定义

接口设计中往可以抽象出一些新的公共参数，从事了近三年的接口开发工作中，我目前能想到了一些较为常见的公共接口参数如下：
    
公共参数 | 含意 | 定义该参数的意义
---------|----------|--------------
timestamp|毫秒级时间戳|1.客户端的请求时间标示 2.后端可以做请求过期验证 3.该参数参与签名算法增加签名的唯一性
app_key  |签名公钥|签名算法的公钥，后端通过公钥可以得到对应的私钥
sign     |接口签名|通过请求的参数和定义好的签名算法生成接口签名，作用防止中间人篡改请求参数
did      |设备ID|设备的唯一标示，生成规则例如android的mac地址的md5和ios曾今udid(目前无法获取)的md5, 1:数据收集 2.便于问题追踪 3.消息推送标示

# 接口版本化的问题

接口设计中有个算是历史上的难题 -> 接口版本化。曾经也去调研了很多关于接口版本化的资料和设计，最后我得到的结论大致如下：

- 接口的版本区分为
    + 大版本
        * 原则：大版本的数量最多控制到5个以内(我个人跟倾向于3个)，超过版本限制的版本提示升级到新版本
        * 方案
            - uri携带版本号，例如：v1/user/get
            - 请求参数，例如：user/get?v=1.0
    + 小版本
        * 原则：自己把控吧😄
        * 方案
            - uri携带版本号，例如：v1/user/get_01
            - 请求参数，小数点右边就是小版本，例如：user/get?v=1.1

# 接口的安全性

接口的设计肯定绕不开安全这两个字，为了达到尽可能的安全，我们需要**尽可能的增加被攻击的难度**，以下是我了解和使用到的一些常见的手段去增加接口的安全性(https这里就不讨论了)：

> 过期验证／签名验证／重访攻击／限流／转义

伪代码如下：

```
// 过期验证
if (microtime(true)*1000 - $_REQUEST['timestamp'] > 5000) {
    throw new \Exception(401, 'Expired request');
}
```

```
// 签名验证(公钥校验省略)
$params = ksort($_REQUEST);
unset($params['sign']);
$sign = md5(sha1(implode('-', $params) . $_REQUEST['app_key']));
if ($sign !== $_REQUEST['sign']) {
    throw new \Exception(401, 'Invalid sign');
}
```

```
/**
 * 重访攻击
 * @params noise string 随机字符串
 */
$key = md5("{$_REQUEST['REQUEST_URI']}-{$_REQUEST['timestamp']}-{$_REQUEST['noise']}-{$_REQUEST['did']}");
if ($redisInstance->exists($key)) {
    throw new \Exception(401, 'Repeated request');
}
```

```
// 限流
$key = md5("{$_REQUEST['REQUEST_URI']}-{$_REQUEST['REMOTE_ADDR']}-{$_REQUEST['did']}");
if ($redisInstance->get($key) > 60) {
    throw new \Exception(401, 'Request limit');
}
$redisInstance->incre($key);
```

```
// 转义
$username = htmlspecialchars($_REQUEST['username']);
```

# 接口的代码设计 -> 解耦业务 即插即用

> 这个过程的关键字：抽象成类 前置中间件 注入

接着就是我们代码设计的层面了，如何抽象公共的部分与业务代码解耦。

一般写法, 定义个全局函数，然后每个接口开始时调用该函数：
```
// 全局定义一个函数
function check () {
    // 校验公共参数
    # code ...
    // 校验签名
    # code ...
    // 校验频率
    # code ...
    // 等等...
}
```

二般写法, 定义个父类方法，然后每个接口类继承该接口，构造函数调用改方法，其实和上面的换汤不换药：
```
// 父类方法

class father
{
    public function __construct()
    {
        $this->check();
    }

    public function check () {
        // 校验公共参数
        # code ...
        // 校验签名
        # code ...
        // 校验频率
        # code ...
        // 等等...
    }
}
```

重点来了，我提倡的第三般写法，对象链和前置中间件：
```
/**
 * 检验抽象类
 */
abstract class Check
{
    /**
     * 下一个check实体
     *
     * @var object
     */
    private $nextCheckInstance;
    
    /**
     * 校验方法
     *
     * @param Request $request 请求对象
     */
    abstract public function operate(Request $request);

    /**
     * 设置责任链上的下一个对象
     *
     * @param Check $check
     */
    public function setNext(Check $check)
    {
        $this->nextCheckInstance = $check;
        return $check;
    }

    /**
     * 启动
     *
     * @param Request $request 请求对象
     */
    public function start(Request $request)
    {
        $this->doCheck($request);
        // 调用下一个对象
        if (! empty($this->nextCheckInstance)) {
            $this->nextCheckInstance->start($request);
        }
    }
}

// 校验公共参数类
class ParamsCheck extends Check
{
    public function operate()
    {
       // 校验公共参数
        # code ... 
    }
}

// 校验签名类
class SignCheck extends Check
{
    public function operate()
    {
       // 校验签名
        # code ... 
    }
}

// 等等...

// 前置中间件类
class FrontMiddleware
{
    public function run()
    {
        // 初始化一个：必传参数校验的check
        $checkParams   =  new ParamsCheck();
        // 初始化一个：签名check
        $checkSign     =  new SignCheck();
        // 初始化一个：频率check
        $checkFrequent =  new FrequentCheck();
        // 等等...

        // 构成对象链
        $checkParams->setNext($checkSign)
                    ->setNext($checkFrequent)
                    ...
        // 启动
        $checkArguments->start();
    }
}
```

# 接口的可读性

关于可读性的不得不提到的就是RESTFUL，这里我就不讨论RESTFUL，大家可以自行补充相关知识。关于接口设计可读性的我的一些思考：

- url
    + 非RESTFUL: 资源／资源／操作(动词)， 例如 content/article/get -> 获取内容资源下的一篇文章资源
    + RESTFUL: 资源／资源／资源， 例如 get content/article/1 -> 获取内容资源下文章ID为1的文章资源
- method
    + 非RESTFUL: get便于查nginx日志，上传资源post, 没啥硬性要求
    + RESTFUL: 符合RESTFUL的思想
- request params: 个人更青睐于下划线命名，适当的单词缩写
- response params: 响应的code要符合http status
    + 200 -> 正常
    + 400 -> 缺少公共必传参数或者业务必传参数
    + 401 -> 接口校验失败 例如签名
    + 403 -> 没有该接口的访问权限
    + 499 -> 上游服务响应时间超过接口设置的超时时间
    + 500 -> 代码错误
    + 501 -> 不支持的接口method
    + 502 -> 上游服务返回的数据格式不正确
    + 503 -> 上游服务超时
    + 504 -> 上游服务不可用

```
// 响应的格式
{
    "code": 200,
    "msg": "ok",
    "data": {

    }
}
```

# 接口文档

好的接口文档就是生产力， swagger + api blueprint 自行google吧😄

# 我遇到的坑

这里遇到的一个比较大的坑就是http协议历史遗留的bug：

> 不区分url里的空格 和加号➕

带来的问题就是urldecode会把参数里的+号转为空格，所以这种场景的就得使用rawurldecode防止+转成空格。比如做接口的参数校验的时候～