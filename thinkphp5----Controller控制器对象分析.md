# thinkphp5----Controller控制器对象分析.md

* [thinkphp5\-\-\-\-Controller控制器对象分析\.md](#thinkphp5----controller%E6%8E%A7%E5%88%B6%E5%99%A8%E5%AF%B9%E8%B1%A1%E5%88%86%E6%9E%90md)
    * [1\. 控制器实例化时机](#1-%E6%8E%A7%E5%88%B6%E5%99%A8%E5%AE%9E%E4%BE%8B%E5%8C%96%E6%97%B6%E6%9C%BA)
    * [2\. 引入跳转性状](#2-%E5%BC%95%E5%85%A5%E8%B7%B3%E8%BD%AC%E6%80%A7%E7%8A%B6)
      * [2\.1 成功时的跳转或json输出](#21-%E6%88%90%E5%8A%9F%E6%97%B6%E7%9A%84%E8%B7%B3%E8%BD%AC%E6%88%96json%E8%BE%93%E5%87%BA)
      * [2\.2 错误时的跳转或json输出](#22-%E9%94%99%E8%AF%AF%E6%97%B6%E7%9A%84%E8%B7%B3%E8%BD%AC%E6%88%96json%E8%BE%93%E5%87%BA)
      * [2\.3 直接返回数据或json输出](#23-%E7%9B%B4%E6%8E%A5%E8%BF%94%E5%9B%9E%E6%95%B0%E6%8D%AE%E6%88%96json%E8%BE%93%E5%87%BA)
      * [2\.4 跳转](#24-%E8%B7%B3%E8%BD%AC)
    * [3\. 控制器初始化](#3-%E6%8E%A7%E5%88%B6%E5%99%A8%E5%88%9D%E5%A7%8B%E5%8C%96)
      * [3\.1 实例化模板对象](#31-%E5%AE%9E%E4%BE%8B%E5%8C%96%E6%A8%A1%E6%9D%BF%E5%AF%B9%E8%B1%A1)
      * [3\.2 实例化请求对象](#32-%E5%AE%9E%E4%BE%8B%E5%8C%96%E8%AF%B7%E6%B1%82%E5%AF%B9%E8%B1%A1)
      * [3\.3 执行初始化](#33-%E6%89%A7%E8%A1%8C%E5%88%9D%E5%A7%8B%E5%8C%96)
      * [3\.4 方法调用的前置操作](#34-%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8%E7%9A%84%E5%89%8D%E7%BD%AE%E6%93%8D%E4%BD%9C)
    * [4\. 展示方法](#4-%E5%B1%95%E7%A4%BA%E6%96%B9%E6%B3%95)
      * [4\.1 设置模板引擎](#41-%E8%AE%BE%E7%BD%AE%E6%A8%A1%E6%9D%BF%E5%BC%95%E6%93%8E)
      * [4\.2 设置模板变量](#42-%E8%AE%BE%E7%BD%AE%E6%A8%A1%E6%9D%BF%E5%8F%98%E9%87%8F)
      * [4\.3 获取模板替换后的内容字符串](#43-%E8%8E%B7%E5%8F%96%E6%A8%A1%E6%9D%BF%E6%9B%BF%E6%8D%A2%E5%90%8E%E7%9A%84%E5%86%85%E5%AE%B9%E5%AD%97%E7%AC%A6%E4%B8%B2)
      * [4\.4 展示输出模板](#44-%E5%B1%95%E7%A4%BA%E8%BE%93%E5%87%BA%E6%A8%A1%E6%9D%BF)
    * [5\. 数据验证](#5-%E6%95%B0%E6%8D%AE%E9%AA%8C%E8%AF%81)


### 1. 控制器实例化时机

前面我们分析的[路由](https://github.com/wujiangweiphp/thinkphp5-source-analysis/blob/master/thinkphp5----Route%E8%B7%AF%E7%94%B1%E5%AF%B9%E8%B1%A1%E5%88%86%E6%9E%90.md)里最后都会实例化对应的path
中的控制器类，而所有的控制器都继承了框架的控制器基类 `Controller`
所以此类的是实例化时机则是由其子类实例化时自动触发。

### 2. 引入跳转性状

```php
Loader::import('controller/Jump', TRAIT_PATH, EXT);
class Controller
{
    use Jump;
}
```

性状(`Trait`)是PHP5.4引入的一个新的特性，注意它的引入是在class的内部使用`use`关键词，而不是在外面
它其实是实现了类的方法的共享化。也就是性状里所有的属性，在类里是可以当成成员变量或成员方法直接使用的。

`Jump`性状一共提供了4个常用方法：

#### 2.1 成功时的跳转或json输出

```php
success($msg = '', $url = null, $data = '', $wait = 3, array $header = [])
```
#### 2.2 错误时的跳转或json输出

```php
error($msg = '', $url = null, $data = '', $wait = 3, array $header = [])
```
#### 2.3 直接返回数据或json输出

```php
result($data, $code = 0, $msg = '', $type = '', array $header = [])
```
#### 2.4 跳转
```php
redirect($url, $params = [], $code = 302, $with = [])
```

这4个方法中，前三个方法输出是`html`还是`json`是根据当前请求的类型是否为`ajax`
来判断使用配置中的 `default_ajax_return` 类型还是 `default_return_type`类型返回

如果我们需要重写默认的跳转模板，可以更改配置项
错误模板：`dispatch_error_tmpl`
成功模板：`dispatch_success_tmpl`

### 3. 控制器初始化

```php
public function __construct(Request $request = null)
{
    $this->view    = View::instance(Config::get('template'), Config::get('view_replace_str'));
    $this->request = is_null($request) ? Request::instance() : $request;

    // 控制器初始化
    $this->_initialize();

    // 前置操作方法
    if ($this->beforeActionList) {
        foreach ($this->beforeActionList as $method => $options) {
            is_numeric($method) ?
            $this->beforeAction($options) :
            $this->beforeAction($method, $options);
        }
    }
}
```
控制器的初始化做了4部操作：

#### 3.1 实例化模板对象
```php
$this->view    = View::instance(Config::get('template'), Config::get('view_replace_str'));
```
#### 3.2 实例化请求对象
```php
$this->request = is_null($request) ? Request::instance() : $request;
```
#### 3.3 执行初始化
```php
$this->_initialize();
```
【注意】：这个方法很有用，如果我们某些控制器的所有方法都需要做某个初始化操作，那么可以在控制器里
重写这个方法，不需要自己手动调用，基类会自动调用

#### 3.4 方法调用的前置操作

```php
if ($this->beforeActionList) {
    foreach ($this->beforeActionList as $method => $options) {
        is_numeric($method) ?
        $this->beforeAction($options) :
        $this->beforeAction($method, $options);
    }
}
```
这个前置操作是根据我们设置的`beforeActionList`成员变量来判断的，比如我们设置如下：
```php
protected  $beforeActionList = [
    'aaa',
    'bbbAbc' =>  ['except'=>'ccc'],
    'ddd' =>  ['only'=>'aaa,ccc'],
];
```
注意：
1. 设置的键如果是数字，那么值就直接是方法名称
2. 设置的键是字符串，则直接为方法名称，值为过滤参数

这里的过滤参数有两种，
一种是 `only` :  这个方法调用的前提是`only`里面有才调用 
一种是 `except` : 这个方法调用的前提是 `except` 里面没有才调用 
实际的使用示例可以参考[这里](https://www.kancloud.cn/manual/thinkphp5/118050)

### 4. 展示方法

#### 4.1 设置模板引擎
```php
engine($engine)
```
比如`Think` 具体模板引擎扩展可以参考 `tp5/thinkphp/library/think/view/driver/Think.php`

#### 4.2 设置模板变量
```php
assign($name, $value = '')
```
#### 4.3 获取模板替换后的内容字符串
```php
fetch($template = '', $vars = [], $replace = [], $config = [])
```
#### 4.4 展示输出模板
```php
display($content = '', $vars = [], $replace = [], $config = [])
```

### 5. 数据验证

```php
protected function validate($data, $validate, $message = [], $batch = false, $callback = null)
{
    if (is_array($validate)) {
        $v = Loader::validate();
        $v->rule($validate);
    } else {
        // 支持场景
        if (strpos($validate, '.')) {
            list($validate, $scene) = explode('.', $validate);
        }

        $v = Loader::validate($validate);

        !empty($scene) && $v->scene($scene);
    }

    // 批量验证
    if ($batch || $this->batchValidate) {
        $v->batch(true);
    }

    // 设置错误信息
    if (is_array($message)) {
        $v->message($message);
    }

    // 使用回调验证
    if ($callback && is_callable($callback)) {
        call_user_func_array($callback, [$v, &$data]);
    }

    if (!$v->check($data)) {
        if ($this->failException) {
            throw new ValidateException($v->getError());
        }

        return $v->getError();
    }

    return true;
}

```
这个方法其实是将验证器 `Validate` 在控制器中重写了一遍，调用很简单
```php
$result = $this->validate(
            [
                'name'  => 'thinkphp3',
                'email' => 'thinkphpqq.com',
            ],
            [
                'name'  => 'require|max:25',
                'email'   => 'email',
            ],
            [
                'name'  => array('require'=>'needed','max'=>'out'),
                'email'   => 'email'
            ]
        );
if(true !== $result){
    // 验证失败 输出错误信息
    dump($result);
}
```
这里的 `$result` 直接是错误信息。常见的验证规则部分如下：

类型验证器

| 名称 | 说明 |
| --- | --- |
| `require` | 必须存在 |
| `accepted` | 接受 值只能为 `1,on,yes` |
| `date` | 必须是日期 |
| `alpha` | 只允许字母 |
| `alphaNum` | 只允许字母和数字 |
| `alphaDash` | 字母、数字和下划线 破折号 |
| `chs` | 只允许汉字 |
| `chsAlpha` | 只允许汉字字母 |
| `chsAlphaNum` | 只允许汉字字母和数字 |
| `chsDash`| 汉字字母、数字和下划线 破折号 |
| `activeUrl` | 是否为有效的网址 |
| `ip` | ip | 
| `url` | url地址 |
| `float` | 浮点型 | 
| `number` | 数字 |
| `integer` | 整数 |
| `email` | 邮箱 |
| `boolean` | bool值 |
| `array` | 是否为数组 |
| `file` | 是否为文件|
| `image` | 是否为图片 |
| `token` | 是否为token | 

规则型验证器 

| 名称 | 说明 |
| --- | --- |
| `in` | 在什么里面 |
| `notIn` | 不在什么里面 |
| `between` | 范围 |
| `notBetween` | 不在范围 |
| `length` | 长度 |
| `max` | 最大 |
| `min` | 最小 | 
| `allowIp` | 允许ip |
| `denyIp` | 禁止ip |
| `gt` | 大于 |
| `lt` | 小于 |
| `egt` | 大于等于 |
| `elt` | 小于等于 |
| `eq` | 等于 |





