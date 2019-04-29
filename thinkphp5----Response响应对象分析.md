# thinkphp5----Response响应对象分析.md

* [thinkphp5\-\-\-\-Response响应对象分析\.md](#thinkphp5----response%E5%93%8D%E5%BA%94%E5%AF%B9%E8%B1%A1%E5%88%86%E6%9E%90md)
    * [1\. Response对象的实例化时机](#1-response%E5%AF%B9%E8%B1%A1%E7%9A%84%E5%AE%9E%E4%BE%8B%E5%8C%96%E6%97%B6%E6%9C%BA)
    * [2\. create方法](#2-create%E6%96%B9%E6%B3%95)
    * [3\. 构造函数\_\_construct](#3-%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0__construct)
    * [4\. send方法](#4-send%E6%96%B9%E6%B3%95)
    * [5\. 扩展响应类](#5-%E6%89%A9%E5%B1%95%E5%93%8D%E5%BA%94%E7%B1%BB)
    * [6\. json()方法](#6-json%E6%96%B9%E6%B3%95)

### 1. Response对象的实例化时机

进入App.php的run方法 第139行
```php
$data = self::exec($dispatch, $config);
```
我们来看这个方法是正常执行下获取Response对象
```php
try{
    $data = self::exec($dispatch, $config);
} catch (HttpResponseException $exception) {
    $data = $exception->getResponse();
}
```
`exec` 方法中涉及到的
```php
case 'redirect': // 重定向跳转
    $data = Response::create($dispatch['url'], 'redirect')
        ->code($dispatch['status']);
    break;
```

可以看到，正常情况下，只有跳转才会立即创建对象，否则异常则会生成 响应相关的异常类

我们接着'run'方法往下看
```php
if ($data instanceof Response) {
    $response = $data;
} elseif (!is_null($data)) {
    //more code here ...
    $response = Response::create($data, $type);
} else {
    $response = Response::create();
}
```
根据前面的执行是否为`Response`对象 来判断是否需要创建改对象，所有的初始化都指向了同一个方法`create`,那我们来看看
这个方法究竟做了什么。

### 2. create方法

```php
public static function create($data = '', $type = '', $code = 200, array $header = [], $options = [])
{
    $class = false !== strpos($type, '\\') ? $type : '\\think\\response\\' . ucfirst(strtolower($type));
    if (class_exists($class)) {
        $response = new $class($data, $code, $header, $options);
    } else {
        $response = new static($data, $code, $header, $options);
    }
    return $response;
}
```
第一句代码是，如果类型中含有命令空间的 `\` ,则类为该命令空间，否则就会去 `\think\response` 中寻找
对应 的类。
然后就是实例化对应的类。这里有个比较有趣的地方就是

```php
$response = new static($data, $code, $header, $options);
```
这里使用的是  `new static` ,而不是 `new self` , 那么他们之间有什么区别么？
`new static` : 是 php5.3新加进来的，作用是如果有子类继承了或调用父类的静态方法实例化，该方法返回的是子类的实例化对象，
而不是父类本身。同样的还有 `static::$name` 等静态属性也是子类自己的，不是父类的

`new self` : 则是该方法所在的的类的实例化，方法在父类 不管父类还是子类调用都是实例化父类，方法在子类，
则实例化的是子类对象

更多信息可以参考：[这里](https://www.jb51.net/article/121420.htm)

### 3. 构造函数__construct

```php
public function __construct($data = '', $code = 200, array $header = [], $options = [])
{
    $this->data($data);
    if (!empty($options)) {
        $this->options = array_merge($this->options, $options);
    }
    $this->contentType($this->contentType, $this->charset);
    $this->header = array_merge($this->header, $header);
    $this->code   = $code;
}
```
实例化对象做了以下几步操作：
1. 填充数据
2. 合并配置项
3. 设置内容类型 和 字符编码
4. 设置头 和 响应码

### 4. send方法

`run`方法获取到响应的数据对象以后，直接返回该对象，然后
```php
App::run()->send();
```
我们来看看最终返回的`send`方法
```php
public function send()
{
    // more code ...

    // 处理输出数据
    $data = $this->getContent();

    // more code ...

    if (200 == $this->code) {
        $cache = Request::instance()->getCache();
        if ($cache) {
            $this->header['Cache-Control'] = 'max-age=' . $cache[1] . ',must-revalidate';
            $this->header['Last-Modified'] = gmdate('D, d M Y H:i:s') . ' GMT';
            $this->header['Expires']       = gmdate('D, d M Y H:i:s', $_SERVER['REQUEST_TIME'] + $cache[1]) . ' GMT';
            Cache::tag($cache[2])->set($cache[0], [$data, $this->header], $cache[1]);
        }
    }

    if (!headers_sent() && !empty($this->header)) {
        // 发送状态码
        http_response_code($this->code);
        // 发送头部信息
        foreach ($this->header as $name => $val) {
            if (is_null($val)) {
                header($name);
            } else {
                header($name . ':' . $val);
            }
        }
    }

    echo $data;

    if (function_exists('fastcgi_finish_request')) {
        // 提高页面响应
        fastcgi_finish_request();
    }

    // more code ...

    // 清空当次请求有效的数据
    if (!($this instanceof RedirectResponse)) {
        Session::flush();
    }
}
```
该方法做了以下几步操作：

1. 获取相应的数据
```php
$data = $this->getContent();
```
2. 如果响应正常，检查是否有缓存，如果有则返回响应头
```php
$cache = Request::instance()->getCache();
if ($cache) {
    $this->header['Cache-Control'] = 'max-age=' . $cache[1] . ',must-revalidate';
    $this->header['Last-Modified'] = gmdate('D, d M Y H:i:s') . ' GMT';
    $this->header['Expires']       = gmdate('D, d M Y H:i:s', $_SERVER['REQUEST_TIME'] + $cache[1]) . ' GMT';
    Cache::tag($cache[2])->set($cache[0], [$data, $this->header], $cache[1]);
}
```
这里设置了三个头部信息，来控制浏览器缓存：
`Cache-Control` : 开启缓存，且设置了最大生命周期 `max-age`
`Last-Modified` : 浏览器会根据这个上次修改时间对比 来判断是否需要调取缓存还是重新发起请求
`Expires` : 根据当前的请求时间和最大生命周期来判断是否已经过期，需要重新请求

3. 判断是否已经发送过响应头，如果没有则发送
```php
if (!headers_sent() && !empty($this->header)) {
    // 发送状态码
    http_response_code($this->code);
    // 发送头部信息
    foreach ($this->header as $name => $val) {
        if (is_null($val)) {
            header($name);
        } else {
            header($name . ':' . $val);
        }
    }
}
```
`headers_sent` : 是否已经发送过响应头
`http_response_code` : 发送响应码
`header` : 发送响应头

4. 输出数据
```php
echo $data;

if (function_exists('fastcgi_finish_request')) {
    // 提高页面响应
    fastcgi_finish_request();
}
```
这里有两个操作，一是输出数据，二是调用了函数`fastcgi_finish_request`

`fastcgi_finish_request`: 官方文档给的解释是----此函数冲刷(flush)所有响应的数据给客户端并结束请求。 这使得客户端结束连接后，需要大量时间运行的任务能够继续运行。
也就是输出之后，立即发送给浏览器或者客户端，并结束请求，但是该函数之后的代码仍然能够正常执行，这给了
部分日志记录任务较大开销的函数可以不阻塞这次请求。

【注意】：
1. `fastcgi_finish_request` 函数windows下没有此扩展函数 linux下有
2. 如果该函数后续的操作很复杂，用时比较长，就会导致该次 `session` 会话被锁定，而导致下一次请求可能被阻塞
详情可以参见[`fastcgi_finish_request`](https://www.php.net/manual/zh/function.session-write-close.php)

### 5. 扩展响应类

我们找到 `tp5/thinkphp/library/think/response/Json.php`

```php
namespace think\response;

use think\Response;

class Json extends Response
{
    // 输出参数
    protected $options = [
        'json_encode_param' => JSON_UNESCAPED_UNICODE,
    ];

    protected $contentType = 'application/json';

    /**
     * 处理数据
     * @access protected
     * @param mixed $data 要处理的数据
     * @return mixed
     * @throws \Exception
     */
    protected function output($data)
    {
        try {
            // 返回JSON数据格式到客户端 包含状态信息
            $data = json_encode($data, $this->options['json_encode_param']);

            if ($data === false) {
                throw new \InvalidArgumentException(json_last_error_msg());
            }

            return $data;
        } catch (\Exception $e) {
            if ($e->getPrevious()) {
                throw $e->getPrevious();
            }
            throw $e;
        }
    }

}
```

可以看到，该类实际上只做了四步操作
1. 继承基类 `Response`
2. 配置了json选项
3. 设置响应内容类型为 `application/json`
4. 重写了`output`方法

实际上核心就是 `output` 方法，该方法对数据进行了重新组装，并返回该组装后的数据，那他是什么时候被调用的呢。
前面我们`send`方法第1步中 
```php
$data = $this->getContent();
```
使用了 `getContent` 方法，该方法的实际操作
```php
public function getContent()
{
    if (null == $this->content) {
        $content = $this->output($this->data);
        // more code here ...
        $this->content = (string) $content;
    }
    return $this->content;
}
```
第一句就是调用`output`方法获取数据

【总结】：扩展响应方法，只要继承`Response`并重写 `output` 方法即可

### 6. `json()`方法

我们看一下常用的`json`方法做了什么
```php
function json($data = [], $code = 200, $header = [], $options = [])
{
    return Response::create($data, 'json', $code, $header, $options);
}
```
实例化了一个`json`的`Response`对象并返回，而我们在控制器的方法中也是调用的`return` 返回此方法的结果
同样的方法还有

| 方法名 | 说明 |
| ---- | ---- |
| `response` | 响应 |
| `view` | 视图响应 |
| `json` | json响应 |
| `jsonp` | jsonp响应 |
| `xml` | xml响应 |
| `redirect` | 跳转响应 |

我们简单说一下 `view`

```php
 protected function output($data)
{
    // 渲染模板输出
    return ViewTemplate::instance(Config::get('template'), Config::get('view_replace_str'))
        ->fetch($data, $this->vars, $this->replace);
}

public function getVars($name = null)
{
    if (is_null($name)) {
        return $this->vars;
    } else {
        return isset($this->vars[$name]) ? $this->vars[$name] : null;
    }
}

public function assign($name, $value = '')
{
    if (is_array($name)) {
        $this->vars = array_merge($this->vars, $name);
        return $this;
    } else {
        $this->vars[$name] = $value;
    }
    return $this;
}

public function replace($content, $replace = '')
{
    if (is_array($content)) {
        $this->replace = array_merge($this->replace, $content);
    } else {
        $this->replace[$content] = $replace;
    }
    return $this;
}
```
核心方法`output` 在输出之前搜集了 变量及对应的替换值，传给了`View`的`fetch`方法，最终进行相关的变量替换，并
返回替换后的结果，具体的参考 `View`视图相关的分析。


