# thinkphp5----Request对象请求分析.md

* [thinkphp5\-\-\-\-Request对象请求分析\.md](#thinkphp5----request%E5%AF%B9%E8%B1%A1%E8%AF%B7%E6%B1%82%E5%88%86%E6%9E%90md)
    * [1\. 请求实例化时机](#1-%E8%AF%B7%E6%B1%82%E5%AE%9E%E4%BE%8B%E5%8C%96%E6%97%B6%E6%9C%BA)
    * [2\. 获取基础文件](#2-%E8%8E%B7%E5%8F%96%E5%9F%BA%E7%A1%80%E6%96%87%E4%BB%B6)
    * [3\. 设置过滤器](#3-%E8%AE%BE%E7%BD%AE%E8%BF%87%E6%BB%A4%E5%99%A8)
    * [4\. 设置语言包](#4-%E8%AE%BE%E7%BD%AE%E8%AF%AD%E8%A8%80%E5%8C%85)
    * [5\. 获取path](#5-%E8%8E%B7%E5%8F%96path)
    * [6\. 设置分发](#6-%E8%AE%BE%E7%BD%AE%E5%88%86%E5%8F%91)
    * [7\. 获取请求头和请求参数](#7-%E8%8E%B7%E5%8F%96%E8%AF%B7%E6%B1%82%E5%A4%B4%E5%92%8C%E8%AF%B7%E6%B1%82%E5%8F%82%E6%95%B0)
      * [7\.1 获取header](#71-%E8%8E%B7%E5%8F%96header)
      * [7\.2  获取param](#72--%E8%8E%B7%E5%8F%96param)
      * [7\.3 获取请求方式](#73-%E8%8E%B7%E5%8F%96%E8%AF%B7%E6%B1%82%E6%96%B9%E5%BC%8F)
      * [7\.4 获取get参数](#74-%E8%8E%B7%E5%8F%96get%E5%8F%82%E6%95%B0)
      * [7\.5 获取post参数](#75-%E8%8E%B7%E5%8F%96post%E5%8F%82%E6%95%B0)
      * [7\.6 put等其他请求](#76-put%E7%AD%89%E5%85%B6%E4%BB%96%E8%AF%B7%E6%B1%82)
      * [7\.7 file方法](#77-file%E6%96%B9%E6%B3%95)
      * [7\.8 input方法](#78-input%E6%96%B9%E6%B3%95)
      * [7\.9 getFilter方法](#79-getfilter%E6%96%B9%E6%B3%95)
      * [7\.10 filterValue填充数据](#710-filtervalue%E5%A1%AB%E5%85%85%E6%95%B0%E6%8D%AE)
      * [7\.11 filterExp](#711-filterexp)
      * [7\.12 typeCast类型](#712-typecast%E7%B1%BB%E5%9E%8B)
    * [8\. input函数](#8-input%E5%87%BD%E6%95%B0)
    * [9\. 常用的判断和不常见的获取函数](#9-%E5%B8%B8%E7%94%A8%E7%9A%84%E5%88%A4%E6%96%AD%E5%92%8C%E4%B8%8D%E5%B8%B8%E8%A7%81%E7%9A%84%E8%8E%B7%E5%8F%96%E5%87%BD%E6%95%B0)


### 1. 请求实例化时机

涉及Request对象方法如下
```php
public static function instance($options = [])
protected function __construct($options = [])
public function getContent()
public function getInput()
```

> 我们知道，请求的唯一入口文件是 `tp5/public/index.php`
这个文件引入了启动文件 `tp5/thinkphp/start.php`
关键的一句 

```php
App::run()->send();
```
调用了 `App`中的`run`方法，而`run`方法的第一句

```php
$request = is_null($request) ? Request::instance() : $request;
```
就是实例化 `Request`请求对象，那么我们接触的第一次该对象的方法如下

```php
public static function instance($options = [])
{
    if (is_null(self::$instance)) {
        self::$instance = new static($options);
    }
    return self::$instance;
}
```
该方法用单例模式维护了一个请求生命周期中唯一生成的请求对象 `Reauest`

而实例化自己，调用的构造函数如下：
```php
protected function __construct($options = [])
{
    foreach ($options as $name => $item) {
        if (property_exists($this, $name)) {
            $this->$name = $item;
        }
    }
    if (is_null($this->filter)) {
        $this->filter = Config::get('default_filter');
    }
    // 保存 php://input
    $this->input = file_get_contents('php://input');
}
```
因为我们实例化的参数为空，所以实际上只是操作了 
```php
$this->filter = Config::get('default_filter');
```
从配置文件中读取过滤器
```php
$this->input = file_get_contents('php://input');
```
获取输入流 该输入流接收除`GET`方式之外的其他所有输入

注意：在 PHP 5.6 之前 php://input 打开的数据流只能读取一次 (官方文档)[https://php.net/manual/zh/wrappers.php.php]
所以，这里接收到的输入流，在任何地方如果想使用时，建议使用该对象的的如下方法获取

```php
public function getInput()
{
    return $this->input;
}
```

或者使用
```php
public function getContent()
{
    if (is_null($this->content)) {
        $this->content = $this->input;
    }
    return $this->content;
}
```

### 2. 获取基础文件

涉及Request对象方法如下
```php
public function baseFile($file = null)
```

如果配置文件`tp5/applicaiton/config.php` 设置了 `auto_bind_module` 选项 ，且入口文件并未定义 `BIND_MODULE` 绑定模块常量，那么就会调用此方法
```php
$name = pathinfo($request->baseFile(), PATHINFO_FILENAME);
```
改方法具体内容如下：

```php
public function baseFile($file = null)
{
    if (!is_null($file) && true !== $file) {
        $this->baseFile = $file;
        return $this;
    } elseif (!$this->baseFile) {
        $url = '';
        if (!IS_CLI) {
            $script_name = basename($_SERVER['SCRIPT_FILENAME']);
            if (basename($_SERVER['SCRIPT_NAME']) === $script_name) {
                $url = $_SERVER['SCRIPT_NAME'];
            } elseif (basename($_SERVER['PHP_SELF']) === $script_name) {
                $url = $_SERVER['PHP_SELF'];
            } elseif (isset($_SERVER['ORIG_SCRIPT_NAME']) && basename($_SERVER['ORIG_SCRIPT_NAME']) === $script_name) {
                $url = $_SERVER['ORIG_SCRIPT_NAME'];
            } elseif (($pos = strpos($_SERVER['PHP_SELF'], '/' . $script_name)) !== false) {
                $url = substr($_SERVER['SCRIPT_NAME'], 0, $pos) . '/' . $script_name;
            } elseif (isset($_SERVER['DOCUMENT_ROOT']) && strpos($_SERVER['SCRIPT_FILENAME'], $_SERVER['DOCUMENT_ROOT']) === 0) {
                $url = str_replace('\\', '/', str_replace($_SERVER['DOCUMENT_ROOT'], '', $_SERVER['SCRIPT_FILENAME']));
            }
        }
        $this->baseFile = $url;
    }
    return true === $file ? $this->domain() . $this->baseFile : $this->baseFile;
}
```
很明显初始化时，我们传入的数据为空，走的是第二个`elserif` 的逻辑，PSR规范中不推荐使用 `else if` 这种分开写法了建议写成文中这种。代码中提到的常量，我一股脑儿打印如下（假设访问的是 /index/api/getUser）：

[SCRIPT_FILENAME] => /data/www/tp5/public/index.php
[SCRIPT_NAME] => /index.php
[PHP_SELF] => /index.php/index/api/getUser
[ORIG_SCRIPT_NAME] =>
[DOCUMENT_ROOT] => /data/www/tp5/public

`DOCUMENT_ROOT` : 当前访问域名下的文档根目录
`PHP_SELF`      : 除域名外的访问地址(不包含?后的参数)
`SCRIPT_NAME`   : php脚本文件路径地址
`SCRIPT_FILENAME` : PHP脚本文件完整地址(从根域名开始)
`ORIG_SCRIPT_NAME` : cgi模式下的`SCRIPT_NAME`,此时 `SCRIPT_NAME`的值会变成php可执行的二进制文件[参考](https://www.jianshu.com/p/fea9f422793a)

所以判断了一圈，就是返回了当前入口文件的本地路径地址

### 3. 设置过滤器
包含函数
```php
//设置过滤器
public function filter($filter = null)
```
在App的`run`方法中第95行调用
```php
$request->filter($config['default_filter']);
```

### 4. 设置语言包
包含函数
```php
public function langset($lang = null)
```
分别在第 101/105/106行调用
```php
$request->langset(Lang::range());
```
函数原型，有值返回当前对象，没传值返回当前结果，有点类似jquery的字段设置 `$('name').val()` 函数
```php
public function langset($lang = null)
{
    if (!is_null($lang)) {
        $this->langset = $lang;
        return $this;
    } else {
        return $this->langset ?: '';
    }
}
```
这种设置就是链式，获取就是结果的方式值得一学，将两个函数的功能融于一个函数

### 5. 获取path
包含方法：

```php
public function path()
public function pathinfo()
public function ext()
```
App中的`run`方法第116行 
```php
$dispatch = self::routeCheck($request, $config);
```
路由检查，跟踪进去，该方法的第一行调用了path获取
```php
$path   = $request->path();
```
那么`path()`方法究竟干了啥
```php
public function path()
{
    if (is_null($this->path)) {
        $suffix   = Config::get('url_html_suffix');
        $pathinfo = $this->pathinfo();
        if (false === $suffix) {
            // 禁止伪静态访问
            $this->path = $pathinfo;
        } elseif ($suffix) {
            // 去除正常的URL后缀
            $this->path = preg_replace('/\.(' . ltrim($suffix, '.') . ')$/i', '', $pathinfo);
        } else {
            // 允许任何后缀访问
            $this->path = preg_replace('/\.' . $this->ext() . '$/i', '', $pathinfo);
        }
    }
    return $this->path;
}
```
函数的核心思想就是获取path并去掉其中的后缀，而path来源于下面这个函数
```php
public function pathinfo()
{
    if (is_null($this->pathinfo)) {
        if (isset($_GET[Config::get('var_pathinfo')])) {
            // 判断URL里面是否有兼容模式参数
            $_SERVER['PATH_INFO'] = $_GET[Config::get('var_pathinfo')];
            unset($_GET[Config::get('var_pathinfo')]);
        } elseif (IS_CLI) {
            // CLI模式下 index.php module/controller/action/params/...
            $_SERVER['PATH_INFO'] = isset($_SERVER['argv'][1]) ? $_SERVER['argv'][1] : '';
        }

        // 分析PATHINFO信息
        if (!isset($_SERVER['PATH_INFO'])) {
            foreach (Config::get('pathinfo_fetch') as $type) {
                if (!empty($_SERVER[$type])) {
                    $_SERVER['PATH_INFO'] = (0 === strpos($_SERVER[$type], $_SERVER['SCRIPT_NAME'])) ?
                    substr($_SERVER[$type], strlen($_SERVER['SCRIPT_NAME'])) : $_SERVER[$type];
                    break;
                }
            }
        }
        $this->pathinfo = empty($_SERVER['PATH_INFO']) ? '/' : ltrim($_SERVER['PATH_INFO'], '/');
    }
    return $this->pathinfo;
}
```
如果有配置 `var_pathinfo` 则使用它，
否则如果是 CLI 模式  取第一个参数
如果都没取到，看配置数组 `pathinfo_fetch` 是否有关于 `$_SERVER`的取值建议，并返回

最后一个是访问路径中的扩展后缀的获取
```php
public function ext()
{
    return pathinfo($this->pathinfo(), PATHINFO_EXTENSION);
}
```
使用 系统函数`pathinfo` 传递第二个标志 `PATHINFO_EXTENSION` 获取扩展名称

### 6. 设置分发

包含函数
```php
$request->dispatch($dispatch)
```
赋值路由分发任务$dispatch给$request对象

### 7. 获取请求头和请求参数

包含函数
```php
public function header($name = '', $default = null)
public function param($name = '', $default = null, $filter = '')
public function method($method = false)
public function post($name = '', $default = null, $filter = '')
public function put($name = '', $default = null, $filter = '')
public function get($name = '', $default = null, $filter = '')
public function route($name = '', $default = null, $filter = '')
public function file($name = '')
public function input($data = [], $name = '', $default = null, $filter = '')
protected function getFilter($filter, $default)
private function filterValue(&$value, $key, $filters)
public function filterExp(&$value)
private function typeCast(&$data, $type)
```

App的run方法中，如果设置了`debug` 会调用日志记录

```php
Log::record('[ HEADER ] ' . var_export($request->header(), true), 'info');
Log::record('[ PARAM ] ' . var_export($request->param(), true), 'info');
```

#### 7.1 获取header

```php
public function header($name = '', $default = null)
{
    if (empty($this->header)) {
        $header = [];
        if (function_exists('apache_request_headers') && $result = apache_request_headers()) {
            $header = $result;
        } else {
            $server = $this->server ?: $_SERVER;
            foreach ($server as $key => $val) {
                if (0 === strpos($key, 'HTTP_')) {
                    $key          = str_replace('_', '-', strtolower(substr($key, 5)));
                    $header[$key] = $val;
                }
            }
            if (isset($server['CONTENT_TYPE'])) {
                $header['content-type'] = $server['CONTENT_TYPE'];
            }
            if (isset($server['CONTENT_LENGTH'])) {
                $header['content-length'] = $server['CONTENT_LENGTH'];
            }
        }
        $this->header = array_change_key_case($header);
    }
    if (is_array($name)) {
        return $this->header = array_merge($this->header, $name);
    }
    if ('' === $name) {
        return $this->header;
    }
    $name = str_replace('_', '-', strtolower($name));
    return isset($this->header[$name]) ? $this->header[$name] : $default;
}
```
获取头信息的核心是，如果存在系统函数`apache_request_headers`,则以此函数的结果返回头信息
否则从 `$_SERVER`中取到`HTTP_`开头的并加上请求内容`CONTENT_TYPE`和请求长度`CONTENT_LENGTH` 然后把所有的键名
小写后返回，这里重新学到了个函数 
`array_change_key_case` 改变键名大小写，不传第二个参数默认是 小写

#### 7.2  获取param

```php
public function param($name = '', $default = null, $filter = '')
{
    if (empty($this->param)) {
        $method = $this->method(true);
        // 自动获取请求变量
        switch ($method) {
            case 'POST':
                $vars = $this->post(false);
                break;
            case 'PUT':
            case 'DELETE':
            case 'PATCH':
                $vars = $this->put(false);
                break;
            default:
                $vars = [];
        }
        // 当前请求参数和URL地址中的参数合并
        $this->param = array_merge($this->get(false), $vars, $this->route(false));
    }
    if (true === $name) {
        // 获取包含文件上传信息的数组
        $file = $this->file();
        $data = is_array($file) ? array_merge($this->param, $file) : $this->param;
        return $this->input($data, '', $default, $filter);
    }
    return $this->input($this->param, $name, $default, $filter);
}
```
请求的类型分成了四组
1. GET 请求
2. POST 请求
3. 其他请求（PUT|DELETE|PATCH）
4. 文件请求

然后将他们合并并返回

#### 7.3 获取请求方式

```php
public function method($method = false)
{
    if (true === $method) {
        // 获取原始请求类型
        return IS_CLI ? 'GET' : (isset($this->server['REQUEST_METHOD']) ? $this->server['REQUEST_METHOD'] : $_SERVER['REQUEST_METHOD']);
    } elseif (!$this->method) {
        if (isset($_POST[Config::get('var_method')])) {
            $this->method = strtoupper($_POST[Config::get('var_method')]);
            $this->{$this->method}($_POST);
        } elseif (isset($_SERVER['HTTP_X_HTTP_METHOD_OVERRIDE'])) {
            $this->method = strtoupper($_SERVER['HTTP_X_HTTP_METHOD_OVERRIDE']);
        } else {
            $this->method = IS_CLI ? 'GET' : (isset($this->server['REQUEST_METHOD']) ? $this->server['REQUEST_METHOD'] : $_SERVER['REQUEST_METHOD']);
        }
    }
    return $this->method;
}
```
可以看到 如果是 `IS_CLI`命令行模式，默认请求方式被设置成 `GET`
其他的根据 `$_SERVER['REQUEST_METHOD']` 获得
如果配置了 `var_method` 请求方式，则会自动已post方式去获对应的值

#### 7.4 获取get参数

```php
public function get($name = '', $default = null, $filter = '')
{
    if (empty($this->get)) {
        $this->get = $_GET;
    }
    if (is_array($name)) {
        $this->param      = [];
        return $this->get = array_merge($this->get, $name);
    }
    return $this->input($this->get, $name, $default, $filter);
}
```
毫无疑问是直接获取`$_GET` 全局变量
【坑】有趣的是 ，之前的 `IS_CLI` 被默认设置成 'GET', 那么 cli模式下应该将 `$_SERVER['argv']` 赋值给'get'
这里却没有

#### 7.5 获取post参数

```php
public function post($name = '', $default = null, $filter = '')
{
    if (empty($this->post)) {
        $content = $this->input;
        if (empty($_POST) && false !== strpos($this->contentType(), 'application/json')) {
            $this->post = (array) json_decode($content, true);
        } else {
            $this->post = $_POST;
        }
    }
    if (is_array($name)) {
        $this->param       = [];
        return $this->post = array_merge($this->post, $name);
    }
    return $this->input($this->post, $name, $default, $filter);
}
```
post方法处了我们常见的`$_POST`外，这里还根据头信息中包含 `application/json` 获取了json的数据加入post

【坑】作者肯定是觉得使用`json` 更普遍，但是使用 `xml`的也很多啊，比如微信消息就喜欢发xml

#### 7.6 put等其他请求

```php
public function put($name = '', $default = null, $filter = '')
{
    if (is_null($this->put)) {
        $content = $this->input;
        if (false !== strpos($this->contentType(), 'application/json')) {
            $this->put = (array) json_decode($content, true);
        } else {
            parse_str($content, $this->put);
        }
    }
    if (is_array($name)) {
        $this->param      = [];
        return $this->put = is_null($this->put) ? $name : array_merge($this->put, $name);
    }

    return $this->input($this->put, $name, $default, $filter);
}
```
【坑】如果输入的是json或者是 url参数串，可以被正常解析
`json_decode` 解析json串，但是如果头信息是普通文本，就有问题了
`parse_str` 只能正常解析 url查询串 如果是 xml 就啥也看不了了 所以很鸡肋

#### 7.7 file方法

```php
public function file($name = '')
{
    if (empty($this->file)) {
        $this->file = isset($_FILES) ? $_FILES : [];
    }
    if (is_array($name)) {
        return $this->file = array_merge($this->file, $name);
    }
    $files = $this->file;
    if (!empty($files)) {
        // 处理上传文件
        $array = [];
        foreach ($files as $key => $file) {
            if (is_array($file['name'])) {
                $item  = [];
                $keys  = array_keys($file);
                $count = count($file['name']);
                for ($i = 0; $i < $count; $i++) {
                    if (empty($file['tmp_name'][$i]) || !is_file($file['tmp_name'][$i])) {
                        continue;
                    }
                    $temp['key'] = $key;
                    foreach ($keys as $_key) {
                        $temp[$_key] = $file[$_key][$i];
                    }
                    $item[] = (new File($temp['tmp_name']))->setUploadInfo($temp);
                }
                $array[$key] = $item;
            } else {
                if ($file instanceof File) {
                    $array[$key] = $file;
                } else {
                    if (empty($file['tmp_name']) || !is_file($file['tmp_name'])) {
                        continue;
                    }
                    $array[$key] = (new File($file['tmp_name']))->setUploadInfo($file);
                }
            }
        }
        if (strpos($name, '.')) {
            list($name, $sub) = explode('.', $name);
        }
        if ('' === $name) {
            // 获取全部文件
            return $array;
        } elseif (isset($sub) && isset($array[$name][$sub])) {
            return $array[$name][$sub];
        } elseif (isset($array[$name])) {
            return $array[$name];
        }
    }
    return;
}
```
根据文件`$_FILES` 全局变量 做相应的解析


#### 7.8 input方法

```php
public function input($data = [], $name = '', $default = null, $filter = '')
{
    if (false === $name) {
        // 获取原始数据
        return $data;
    }
    $name = (string) $name;
    if ('' != $name) {
        // 解析name
        if (strpos($name, '/')) {
            list($name, $type) = explode('/', $name);
        } else {
            $type = 's';
        }
        // 按.拆分成多维数组进行判断
        foreach (explode('.', $name) as $val) {
            if (isset($data[$val])) {
                $data = $data[$val];
            } else {
                // 无输入数据，返回默认值
                return $default;
            }
        }
        if (is_object($data)) {
            return $data;
        }
    }

    // 解析过滤器
    $filter = $this->getFilter($filter, $default);

    if (is_array($data)) {
        array_walk_recursive($data, [$this, 'filterValue'], $filter);
        reset($data);
    } else {
        $this->filterValue($data, $name, $filter);
    }

    if (isset($type) && $data !== $default) {
        // 强制类型转换
        $this->typeCast($data, $type);
    }
    return $data;
}
```
`array_walk_recursive`对数组中的每个元素进行循环递归调用


#### 7.9 getFilter方法

获取过滤器，在之前初始化的时候，会将 config.php 中的 `default_filter`作为默认的过滤器
或者手动传递 `,`分割的过滤函数

```php
 protected function getFilter($filter, $default)
{
    if (is_null($filter)) {
        $filter = [];
    } else {
        $filter = $filter ?: $this->filter;
        if (is_string($filter) && false === strpos($filter, '/')) {
            $filter = explode(',', $filter);
        } else {
            $filter = (array) $filter;
        }
    }

    $filter[] = $default;
    return $filter;
}
```

#### 7.10 filterValue填充数据

```php
private function filterValue(&$value, $key, $filters)
{
    $default = array_pop($filters);
    foreach ($filters as $filter) {
        if (is_callable($filter)) {
            // 调用函数或者方法过滤
            $value = call_user_func($filter, $value);
        } elseif (is_scalar($value)) {
            if (false !== strpos($filter, '/')) {
                // 正则过滤
                if (!preg_match($filter, $value)) {
                    // 匹配不成功返回默认值
                    $value = $default;
                    break;
                }
            } elseif (!empty($filter)) {
                // filter函数不存在时, 则使用filter_var进行过滤
                // filter为非整形值时, 调用filter_id取得过滤id
                $value = filter_var($value, is_int($filter) ? $filter : filter_id($filter));
                if (false === $value) {
                    $value = $default;
                    break;
                }
            }
        }
    }
    return $this->filterExp($value);
}
```
【坑】不知道是bug还是啥，传递的三个参数中，第二个`$key`压根儿没用上，

有两组函数值得学习
第一组 函数自定义调用
`is_callable` : 名称是否可以被调用
`call_user_func` : 手动调用函数及对应参数
第二组 相关值的过滤验证
`filter_var(value,filter_id)`: 验证过滤器对应的值是否正确
`filter_id(filter_name)`: 过滤器对应的id
`filter_list()`: 支持的过滤器名称

| 过滤字符串 | 说明 |
| ---- | ---- |
| `int` | 转成整型  |
| `boolean` | 转成bool  |
| `float` |  转成float  |
| `validate_regexp` | 是不是有效的正则表达式  |
| `validate_url` | 是不是有效的url  |
| `validate_email` | 是不是有效的email |
| `validate_ip` | 是不是有效的ip  |
| `string` |  各种类型转成字符串(true == '1' | false == '')  |
| `stripped` | 过滤掉标签  |
| `encoded` | 使用url_encode函数编码 |
| `special_chars` |  是否有特殊字符  |
| `full_special_chars` | 是否全是特殊字符  |
| `unsafe_raw` |  不安全的流  |
| `email` |  邮箱  |
| `url` |  地址 |
| `number_int` | 整型数字  |
| `number_float` | 浮点型数字  |
| `magic_quotes` | 魔术引号过滤  |
| `callback` |  回调函数  |

更多使用详情可参考 [这里](https://blog.csdn.net/renzhenhuai/article/details/18077455)

#### 7.11 filterExp
```php
public function filterExp(&$value)
{
    // 过滤查询特殊字符
    if (is_string($value) && preg_match('/^(EXP|NEQ|GT|EGT|LT|ELT|OR|XOR|LIKE|NOTLIKE|NOT LIKE|NOT BETWEEN|NOTBETWEEN|BETWEEN|NOT EXISTS|NOTEXISTS|EXISTS|NOT NULL|NOTNULL|NULL|BETWEEN TIME|NOT BETWEEN TIME|NOTBETWEEN TIME|NOTIN|NOT IN|IN)$/i', $value)) {
        $value .= ' ';
    }
    // TODO 其他安全过滤
}
```
过滤所有SQL相关的关键词

#### 7.12 typeCast类型

```php
private function typeCast(&$data, $type)
{
    switch (strtolower($type)) {
        // 数组
        case 'a':
            $data = (array) $data;
            break;
        // 数字
        case 'd':
            $data = (int) $data;
            break;
        // 浮点
        case 'f':
            $data = (float) $data;
            break;
        // 布尔
        case 'b':
            $data = (boolean) $data;
            break;
        // 字符串
        case 's':
        default:
            if (is_scalar($data)) {
                $data = (string) $data;
            } else {
                throw new \InvalidArgumentException('variable type error：' . gettype($data));
            }
    }
}
```

`is_scalar` 是否为标量。标量变量是指那些包含了 integer、float、string 或 boolean的变量，而 array、object 和 resource 则不是标量

`gettype`函数 获取变量类型 常用类型如下

| 类型 | 说明 |
| ---- | ---- |
| `boolean` | bool值 |
| `integer` | 整型 |
| `double`  | (历史原因 "double" 在float的情况下不是返回 "float" 而是 "double") |
| `string`  | 字符串 |
| `array`   | 数组 |
| `object`  | 对象 |
| `resource` | 资源 |
|`resource (closed)` | 关闭的资源  PHP 7.2.0 |
| `NULL`  | 空 |
| `unknown type` | 未知类型 |


### 8. input函数

```php
function input($key = '', $default = null, $filter = '')
{
    $filter = empty($filter) ? 'trim' : $filter;
    if (0 === strpos($key, '?')) {
        $key = substr($key, 1);
        $has = true;
    }
    if ($pos = strpos($key, '.')) {
        // 指定参数来源
        list($method, $key) = explode('.', $key, 2);
        if (!in_array($method, ['get', 'post', 'put', 'patch', 'delete', 'route', 'param', 'request', 'session', 'cookie', 'server', 'env', 'path', 'file'])) {
            $key    = $method . '.' . $key;
            $method = 'param';
        }
    } else {
        // 默认为自动判断
        $method = 'param';
    }
    if (isset($has)) {
        return request()->has($key, $method, $default);
    } else {
        return request()->$method($key, $default, $filter);
    }
}
```

三个参数分别是键名、默认值和过滤器
这里有个比较好的常用方法是 
`?`+键名：表示该键是否存在 不管是否为空，补充了平常不容易判断是否有传递某个字段值为0或空串的字段
请求类型.键名 ：对应的请求类型 通用的类型使用 `param.键名`


### 9. 常用的判断和不常见的获取函数

| 函数名称 | 函数说明 |
| ---- | ---- |
| `$request->module()` | 获取模块名称 |
| `$request->controller()` | 获取控制器名称 |
| `$request->action()` | 获取方法名称 |
| `$request->ip()`     | 获取IP地址 |
| `$request->cache()` | 设置请求缓存 |
| `$request->action()` | 获取方法名称 |
| `$request->isGet()` | 是否为GET请求 |
| `$request->isPost()` |  是否为POST请求 |
| `$request->isPut()` |  是否为PUT请求 |
| `$request->isDelete()` |  是否为DELTE请求 |
| `$request->isHead()` |  是否为HEAD请求 |
| `$request->isPatch()` |  是否为PATCH请求 |
| `$request->isOptions()` |  是否为OPTIONS请求 |
| `$request->isCli()` |  是否为cli |
| `$request->isCgi()` |  是否为cgi |
| `$request->isSsl()` | 是否为https 请求 |
| `$request->isAjax()` | 是否为ajax请求 |
| `$request->isPjax()` | 是否为 pjax 请求 |
| `$request->isMobile()` | 是否为手机访问 |
