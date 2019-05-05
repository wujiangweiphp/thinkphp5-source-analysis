# thinkphp5----Route路由对象分析.md

* [thinkphp5\-\-\-\-Route路由对象分析\.md](#thinkphp5----route%E8%B7%AF%E7%94%B1%E5%AF%B9%E8%B1%A1%E5%88%86%E6%9E%90md)
    * [1\. 请求的实例化时机](#1-%E8%AF%B7%E6%B1%82%E7%9A%84%E5%AE%9E%E4%BE%8B%E5%8C%96%E6%97%B6%E6%9C%BA)
      * [1\.1 获取path](#11-%E8%8E%B7%E5%8F%96path)
      * [1\.2 检查路由导入路由配置](#12-%E6%A3%80%E6%9F%A5%E8%B7%AF%E7%94%B1%E5%AF%BC%E5%85%A5%E8%B7%AF%E7%94%B1%E9%85%8D%E7%BD%AE)
      * [1\.3 获取到路由规则后，做了检查路由是否存在](#13-%E8%8E%B7%E5%8F%96%E5%88%B0%E8%B7%AF%E7%94%B1%E8%A7%84%E5%88%99%E5%90%8E%E5%81%9A%E4%BA%86%E6%A3%80%E6%9F%A5%E8%B7%AF%E7%94%B1%E6%98%AF%E5%90%A6%E5%AD%98%E5%9C%A8)
      * [1\.4 如果路由规则不存在，解析对应的控制器模板参数是否存在](#14-%E5%A6%82%E6%9E%9C%E8%B7%AF%E7%94%B1%E8%A7%84%E5%88%99%E4%B8%8D%E5%AD%98%E5%9C%A8%E8%A7%A3%E6%9E%90%E5%AF%B9%E5%BA%94%E7%9A%84%E6%8E%A7%E5%88%B6%E5%99%A8%E6%A8%A1%E6%9D%BF%E5%8F%82%E6%95%B0%E6%98%AF%E5%90%A6%E5%AD%98%E5%9C%A8)
      * [1\.5 返回路由检测结果](#15-%E8%BF%94%E5%9B%9E%E8%B7%AF%E7%94%B1%E6%A3%80%E6%B5%8B%E7%BB%93%E6%9E%9C)
    * [2\. 路由规则的引入](#2-%E8%B7%AF%E7%94%B1%E8%A7%84%E5%88%99%E7%9A%84%E5%BC%95%E5%85%A5)
    * [3\. 无规则路由解析](#3-%E6%97%A0%E8%A7%84%E5%88%99%E8%B7%AF%E7%94%B1%E8%A7%A3%E6%9E%90)
      * [3\.1 路由模块根据是否绑定模块来重新组织](#31-%E8%B7%AF%E7%94%B1%E6%A8%A1%E5%9D%97%E6%A0%B9%E6%8D%AE%E6%98%AF%E5%90%A6%E7%BB%91%E5%AE%9A%E6%A8%A1%E5%9D%97%E6%9D%A5%E9%87%8D%E6%96%B0%E7%BB%84%E7%BB%87)
      * [3\.2 解析路径](#32-%E8%A7%A3%E6%9E%90%E8%B7%AF%E5%BE%84)
      * [3\.3 解析模块和控制器](#33-%E8%A7%A3%E6%9E%90%E6%A8%A1%E5%9D%97%E5%92%8C%E6%8E%A7%E5%88%B6%E5%99%A8)
      * [3\.4 解析方法和额外的参数](#34-%E8%A7%A3%E6%9E%90%E6%96%B9%E6%B3%95%E5%92%8C%E9%A2%9D%E5%A4%96%E7%9A%84%E5%8F%82%E6%95%B0)
      * [3\.5  返回路由分发项](#35--%E8%BF%94%E5%9B%9E%E8%B7%AF%E7%94%B1%E5%88%86%E5%8F%91%E9%A1%B9)
    * [4\. 路由分发](#4-%E8%B7%AF%E7%94%B1%E5%88%86%E5%8F%91)
    * [5\. 模块加载](#5-%E6%A8%A1%E5%9D%97%E5%8A%A0%E8%BD%BD)
      * [5\.1 获取方法名](#51-%E8%8E%B7%E5%8F%96%E6%96%B9%E6%B3%95%E5%90%8D)
      * [5\.2 设置控制器和方法](#52-%E8%AE%BE%E7%BD%AE%E6%8E%A7%E5%88%B6%E5%99%A8%E5%92%8C%E6%96%B9%E6%B3%95)
      * [5\.3 加载控制器](#53-%E5%8A%A0%E8%BD%BD%E6%8E%A7%E5%88%B6%E5%99%A8)
      * [5\.4 方法的调用](#54-%E6%96%B9%E6%B3%95%E7%9A%84%E8%B0%83%E7%94%A8)


### 1. 请求的实例化时机 

前面我们分析 `App.php`中的`run`方法的时候，第115-117行
```php
if (empty($dispatch)) {
    $dispatch = self::routeCheck($request, $config);
}
```
有一个应用调度信息的获取，我们跟踪进去看看 
```php
public static function routeCheck($request, array $config)
{
    $path   = $request->path();
    $depr   = $config['pathinfo_depr'];
    $result = false;

    // 路由检测
    $check = !is_null(self::$routeCheck) ? self::$routeCheck : $config['url_route_on'];
    if ($check) {
        // 开启路由
        if (is_file(RUNTIME_PATH . 'route.php')) {
            // 读取路由缓存
            $rules = include RUNTIME_PATH . 'route.php';
            is_array($rules) && Route::rules($rules);
        } else {
            $files = $config['route_config_file'];
            foreach ($files as $file) {
                if (is_file(CONF_PATH . $file . CONF_EXT)) {
                    // 导入路由配置
                    $rules = include CONF_PATH . $file . CONF_EXT;
                    is_array($rules) && Route::import($rules);
                }
            }
        }

        // 路由检测（根据路由定义返回不同的URL调度）
        $result = Route::check($request, $path, $depr, $config['url_domain_deploy']);
        $must   = !is_null(self::$routeMust) ? self::$routeMust : $config['url_route_must'];

        if ($must && false === $result) {
            // 路由无效
            throw new RouteNotFoundException();
        }
    }

    // 路由无效 解析模块/控制器/操作/参数... 支持控制器自动搜索
    if (false === $result) {
        $result = Route::parseUrl($path, $depr, $config['controller_auto_search']);
    }

    return $result;
}
```

#### 1.1 获取path 
```php
$path = $request->path();
```
该方法是去掉url后面的参数以及文件的后缀后，返回相应的路径
如： `http://tp5.test.com/api/api/aaa/index.html?a=12` 返回的路径是 `/api/api/aaa/index`
```php
$depr   = $config['pathinfo_depr'];
```
这句则是获取路径分割符，默认是 `/`

#### 1.2 检查路由导入路由配置
```php
if (is_file(RUNTIME_PATH . 'route.php')) {
    // 读取路由缓存
    $rules = include RUNTIME_PATH . 'route.php';
    is_array($rules) && Route::rules($rules);
} else {
    $files = $config['route_config_file'];
    foreach ($files as $file) {
        if (is_file(CONF_PATH . $file . CONF_EXT)) {
            // 导入路由配置
            $rules = include CONF_PATH . $file . CONF_EXT;
            is_array($rules) && Route::import($rules);
        }
    }
}
```
这里`Route`对象用到了两个函数，不管走的是哪一个，`Route`就像前面的`App`对象的初始化一样，在没有此对象的时候
系统会执行自动加载，所以调用即进行了实例化。
然后都是引入了配置的路由文件，为什么调用了两个不同的方法呢？稍后解析，我们先看完整个步骤

#### 1.3 获取到路由规则后，做了检查路由是否存在

#### 1.4 如果路由规则不存在，解析对应的控制器模板参数是否存在

#### 1.5 返回路由检测结果


### 2. 路由规则的引入

第一步中我们看到路由规则的引入分别使用了两个不同的方法 `import`和`rules`

我们先看 `rules`方法：

```php
public static function rules($rules = '')
{
    if (is_array($rules)) {
        self::$rules = $rules;
    } elseif ($rules) {
        return true === $rules ? self::$rules : self::$rules[strtolower($rules)];
    } else {
        $rules = self::$rules;
        unset($rules['pattern'], $rules['alias'], $rules['domain'], $rules['name']);
        return $rules;
    }
}
```
该方法实现了三个功能
1. 如果传入的是数组，则做设置操作，将规则赋值给`Route`对象的`$rules`成员变量
2. 如果传递的是`ture`或者其他字符串，则`true`返回所有路由规则，字符串返回对应的路由
3. 如果啥也没传，返回的是删除了别名 域名 名称 正则匹配后的用户自定义的路由
所以，`run`中的方法其实是使用的设置路由这个功能

再来看 `import`方法：

```php
 public static function import(array $rule, $type = '*')
{
    // 检查域名部署
    if (isset($rule['__domain__'])) {
        self::domain($rule['__domain__']);
        unset($rule['__domain__']);
    }

    // 检查变量规则
    if (isset($rule['__pattern__'])) {
        self::pattern($rule['__pattern__']);
        unset($rule['__pattern__']);
    }

    // 检查路由别名
    if (isset($rule['__alias__'])) {
        self::alias($rule['__alias__']);
        unset($rule['__alias__']);
    }

    // 检查资源路由
    if (isset($rule['__rest__'])) {
        self::resource($rule['__rest__']);
        unset($rule['__rest__']);
    }

    self::registerRules($rule, strtolower($type));
}
```
我们看看官方给的例子：
```php
return [
    '__pattern__' => [
        'name' => '\w+',
    ],
    '[hello]'     => [
        ':id'   => ['index/hello', ['method' => 'get'], ['id' => '\d+']],
        ':name' => ['index/hello', ['method' => 'post']],
    ]
];
```

| 变量名 | 说明 |
| ---- | ---- |
| `__domain__` | 检查域名 |
| `__pattern__` | 检查变量命名规则 |
| `__alias__` | 检查路由别名 |
| `__rest__`  | 检查资源型路由 |

显然下面这个`[hello]` 都没有，我们看接下来的这个注册方法

```php
protected static function registerRules($rules, $type = '*')
{
    foreach ($rules as $key => $val) {
        if (is_numeric($key)) {
            $key = array_shift($val);
        }
        if (empty($val)) {
            continue;
        }
        if (is_string($key) && 0 === strpos($key, '[')) {
            $key = substr($key, 1, -1);
            self::group($key, $val);
        } elseif (is_array($val)) {
            self::setRule($key, $val[0], $type, $val[1], isset($val[2]) ? $val[2] : []);
        } else {
            self::setRule($key, $val, $type);
        }
    }
}
```
根据规则 进了
```php
self::group($key, $val);
```
这个方法；`[hello]` 被剥离成 `hello`, 然后我们继续跟进去
```php
public static function group($name, $routes, $option = [], $pattern = [])
{
    if (is_array($name)) {
        $option = $name;
        $name   = isset($option['name']) ? $option['name'] : '';
    }
    // 分组
    $currentGroup = self::getGroup('name');
    if ($currentGroup) {
        $name = $currentGroup . ($name ? '/' . ltrim($name, '/') : '');
    }
    if (!empty($name)) {
        if ($routes instanceof \Closure) {
            $currentOption  = self::getGroup('option');
            $currentPattern = self::getGroup('pattern');
            self::setGroup($name, array_merge($currentOption, $option), array_merge($currentPattern, $pattern));
            call_user_func_array($routes, []);
            self::setGroup($currentGroup, $currentOption, $currentPattern);
            if ($currentGroup != $name) {
                self::$rules['*'][$name]['route']   = '';
                self::$rules['*'][$name]['var']     = self::parseVar($name);
                self::$rules['*'][$name]['option']  = $option;
                self::$rules['*'][$name]['pattern'] = $pattern;
            }
        } else {
            $item          = [];
            $completeMatch = Config::get('route_complete_match');
            foreach ($routes as $key => $val) {
                if (is_numeric($key)) {
                    $key = array_shift($val);
                }
                if (is_array($val)) {
                    $route    = $val[0];
                    $option1  = array_merge($option, isset($val[1]) ? $val[1] : []);
                    $pattern1 = array_merge($pattern, isset($val[2]) ? $val[2] : []);
                } else {
                    $route = $val;
                }

                $options  = isset($option1) ? $option1 : $option;
                $patterns = isset($pattern1) ? $pattern1 : $pattern;
                if ('$' == substr($key, -1, 1)) {
                    // 是否完整匹配
                    $options['complete_match'] = true;
                    $key                       = substr($key, 0, -1);
                } elseif ($completeMatch) {
                    $options['complete_match'] = true;
                }
                $key    = trim($key, '/');
                $vars   = self::parseVar($key);
                $item[] = ['rule' => $key, 'route' => $route, 'var' => $vars, 'option' => $options, 'pattern' => $patterns];
                // 设置路由标识
                $suffix = isset($options['ext']) ? $options['ext'] : null;
                self::name($route, [$name . ($key ? '/' . $key : ''), $vars, self::$domain, $suffix]);
            }
            self::$rules['*'][$name] = ['rule' => $item, 'route' => '', 'var' => [], 'option' => $option, 'pattern' => $pattern];
        }

        foreach (['get', 'post', 'put', 'delete', 'patch', 'head', 'options'] as $method) {
            if (!isset(self::$rules[$method][$name])) {
                self::$rules[$method][$name] = true;
            } elseif (is_array(self::$rules[$method][$name])) {
                self::$rules[$method][$name] = array_merge(self::$rules['*'][$name], self::$rules[$method][$name]);
            }
        }

    } elseif ($routes instanceof \Closure) {
        // 闭包注册
        $currentOption  = self::getGroup('option');
        $currentPattern = self::getGroup('pattern');
        self::setGroup('', array_merge($currentOption, $option), array_merge($currentPattern, $pattern));
        call_user_func_array($routes, []);
        self::setGroup($currentGroup, $currentOption, $currentPattern);
    } else {
        // 批量注册路由
        self::rule($routes, '', '*', $option, $pattern);
    }
}
```
1. 如果`$name` 是数组，则直接获取其中的 `name`属性值赋值给自己
2. `group`路由数组中是否已经存在 该名称的路由前面自动加上`/`
3. 设置路由值
3.1. 如果路由值是闭包函数 就是直接一波调用
```php
if ($routes instanceof \Closure) {
    //more code here ....
    call_user_func_array($routes, []);
    //more code here ....
} 
```
3.2 如果不是，默认是数组
```php
foreach ($routes as $key => $val) {
    if (is_numeric($key)) {
        $key = array_shift($val);
    }
    if (is_array($val)) {
        $route    = $val[0];
        $option1  = array_merge($option, isset($val[1]) ? $val[1] : []);
        $pattern1 = array_merge($pattern, isset($val[2]) ? $val[2] : []);
    } else {
        $route = $val;
    }

    $options  = isset($option1) ? $option1 : $option;
    $patterns = isset($pattern1) ? $pattern1 : $pattern;
    if ('$' == substr($key, -1, 1)) {
        // 是否完整匹配
        $options['complete_match'] = true;
        $key                       = substr($key, 0, -1);
    } elseif ($completeMatch) {
        $options['complete_match'] = true;
    }
    $key    = trim($key, '/');
    $vars   = self::parseVar($key);
    $item[] = ['rule' => $key, 'route' => $route, 'var' => $vars, 'option' => $options, 'pattern' => $patterns];
    // 设置路由标识
    $suffix = isset($options['ext']) ? $options['ext'] : null;
    self::name($route, [$name . ($key ? '/' . $key : ''), $vars, self::$domain, $suffix]);
}
```
代码解释：
3.2.1 如果是数字的键，将值的第一个复制给key
```php
if (is_numeric($key)) {
    $key = array_shift($val);
}
```
3.2.2 获取路由，值是数组取第一个元素，否则就是值本身
```php
if (is_array($val)) {
    $route    = $val[0];
    $option1  = array_merge($option, isset($val[1]) ? $val[1] : []);
    $pattern1 = array_merge($pattern, isset($val[2]) ? $val[2] : []);
} else {
    $route = $val;
}
```
3.2.3 是否需要完整匹配 
根据路由键的配置末尾是否携带 `$` 或者是否配置了 `route_complete_match` 选项来判断是否需要完整匹配整个路由
```php
if ('$' == substr($key, -1, 1)) {
    // 是否完整匹配
    $options['complete_match'] = true;
    $key                       = substr($key, 0, -1);
} elseif ($completeMatch) {
    $options['complete_match'] = true;
}
```
3.2.4 解析变量
```php
$vars   = self::parseVar($key);
```
解析变量的核心操作是以下几句：
```php
foreach (explode('/', $rule) as $val) {
    $optional = false;
    if (false !== strpos($val, '<') && preg_match_all('/<(\w+(\??))>/', $val, $matches)) {
        foreach ($matches[1] as $name) {
            if (strpos($name, '?')) {
                $name     = substr($name, 0, -1);
                $optional = true;
            } else {
                $optional = false;
            }
            $var[$name] = $optional ? 2 : 1;
        }
    }
}

if (0 === strpos($val, '[:')) {
    // 可选参数
    $optional = true;
    $val      = substr($val, 1, -1);
}
if (0 === strpos($val, ':')) {
    // URL变量
    $name       = substr($val, 1);
    $var[$name] = $optional ? 2 : 1;
}
```
从规则上看 可选变量有以下几种形式

| 变量规则 | 说明 |
| `<abc?id>` | 常见于子域名中的二级域名动态变化 |
| `[:id]`  | url上的path中的可选参数 |
| `:id` | url上的path中的必填参数 |

3.2.5 将路由设置进 `name`
```php
self::name($route, [$name . ($key ? '/' . $key : ''), $vars, self::$domain, $suffix]);
```

route : index/hello
name : hello
key: :name
     :id
vars : [id=>1 name=>1]  (可选 2 必填 1)

3.2.6 将对应的请求方式注册进来

```php
foreach (['get', 'post', 'put', 'delete', 'patch', 'head', 'options'] as $method) {
    if (!isset(self::$rules[$method][$name])) {
        self::$rules[$method][$name] = true;
    } elseif (is_array(self::$rules[$method][$name])) {
        self::$rules[$method][$name] = array_merge(self::$rules['*'][$name], self::$rules[$method][$name]);
    }
}
```
设置对应请求规则及请求的参数名称。

整个路由检测做了两件事，
一是引入路由文件
二是根据路由文件中的规则判断是否存在此路由

### 3. 无规则路由解析

上面的规则如果都没有匹配到，则进入默认的地址解析 `App.php`第659行
```php
$result = Route::parseUrl($path, $depr, $config['controller_auto_search']);
```
我们跟踪进来，看看解析路由做了哪些事情
#### 3.1 路由模块根据是否绑定模块来重新组织
比如绑定 admin 模块 /index/hello ===> admin/index/hello
未绑定模块 /admin/index/hello ==>  |admin|index|hello

```php
if (isset(self::$bind['module'])) {
    $bind = str_replace('/', $depr, self::$bind['module']);
    // 如果有模块/控制器绑定
    $url = $bind . ('.' != substr($bind, -1) ? $depr : '') . ltrim($url, $depr);
}
$url              = str_replace($depr, '|', $url);
```
#### 3.2 解析路径
```php
list($path, $var) = self::parseUrlPath($url);
```
我们跟踪进来
```php
private static function parseUrlPath($url)
{
    // 分隔符替换 确保路由定义使用统一的分隔符
    $url = str_replace('|', '/', $url);
    $url = trim($url, '/');
    $var = [];
    if (false !== strpos($url, '?')) {
        // [模块/控制器/操作?]参数1=值1&参数2=值2...
        $info = parse_url($url);
        $path = explode('/', $info['path']);
        parse_str($info['query'], $var);
    } elseif (strpos($url, '/')) {
        // [模块/控制器/操作]
        $path = explode('/', $url);
    } else {
        $path = [$url];
    }
    return [$path, $var];
}
```
这个函数功能很简单，根据是否有 `?` 然后配合使用两个经典函数
`parse_url($url)` :解析路由 
`parse_str($query_str,$var)` : 是查询字符串转换成数组 与函数`http_build_query`函数功能正好相反
`explode` : 将路径分层放入数组

然后返回路径path和参数数组

#### 3.3 解析模块和控制器
```php
// 解析模块
$module = Config::get('app_multi_module') ? array_shift($path) : null;
if ($autoSearch) {
    // 自动搜索控制器
    $dir    = APP_PATH . ($module ? $module . DS : '') . Config::get('url_controller_layer');
    $suffix = App::$suffix || Config::get('controller_suffix') ? ucfirst(Config::get('url_controller_layer')) : '';
    $item   = [];
    $find   = false;
    foreach ($path as $val) {
        $item[] = $val;
        $file   = $dir . DS . str_replace('.', DS, $val) . $suffix . EXT;
        $file   = pathinfo($file, PATHINFO_DIRNAME) . DS . Loader::parseName(pathinfo($file, PATHINFO_FILENAME), 1) . EXT;
        if (is_file($file)) {
            $find = true;
            break;
        } else {
            $dir .= DS . Loader::parseName($val);
        }
    }
    if ($find) {
        $controller = implode('.', $item);
        $path       = array_slice($path, count($item));
    } else {
        $controller = array_shift($path);
    }
} else {
    // 解析控制器
    $controller = !empty($path) ? array_shift($path) : null;
}
```
1 解析模块：根据配置 `app_multi_module` 判断是否开启多模块，使用 `array_shift` 获取路径中的第一个名称
2 解析控制器：根据配置 `controller_auto_search` 判断是否自动搜索控制器，按照配置默认是不自动搜索的
    直接使用 `array_shift` 获取第二个名称作为控制器名称
    如果设置的自动搜索，会根据配置项 `url_controller_layer`(默认是`controller`)拼接上相应的模块
    通过 `Loader::parseName`去解析对应的模块名称 

下面这句话是将每次`_字母`匹配上的都大写字母
如： `abc_def` ===> `abcDef`

```php
$name = preg_replace_callback('/_([a-zA-Z])/', function ($match) {
                return strtoupper($match[1]);
            }, $name);
```
下面这句意思是将匹配上的大写字母更改为 `_`+大写字母
如 `abcDef` 会被替换成 `abc_Def` 这里的 `\\0` 表示在匹配项之前添加 `_` 
如果数字大于0 则是直接替换掉大写字母变成 `_`

```php
strtolower(trim(preg_replace("/[A-Z]/", "_\\0", $name), "_"));
```

这里用到了五个函数
`preg_replace_callback`: 回调函数替换 
`preg_replace` : 正则替换
`lcfirst` ：首字母小写
`ucfirst` ：首字母大写
`strtoupper` ：字母全转大写
`strtolower` ：字母全转小写

#### 3.4 解析方法和额外的参数

```php
// 解析操作
$action = !empty($path) ? array_shift($path) : null;
// 解析额外参数
self::parseUrlParams(empty($path) ? '' : implode('|', $path));
```

看看解析参数做了什么
```php
if ($url) {
    if (Config::get('url_param_type')) {
        $var += explode('|', $url);
    } else {
        preg_replace_callback('/(\w+)\|([^\|]+)/', function ($match) use (&$var) {
            $var[$match[1]] = strip_tags($match[2]);
        }, $url);
    }
}
// 设置当前请求的参数
Request::instance()->route($var);
```
这里用到了 数组相加。数组相加的规则是，已经有的键值不会被覆盖，没有的键值会被加进来

```php
$a = array('name'=>'aaa');
$b = array('name'=>'abb','age'=>12);
print_r($a+$b);
```
打印结果：
```php
Array
(
    [name] => aaa
    [age] => 12
)
```
`/(\w+)\|([^\|]+)/` : 实际上正则的是 `aaa|bac|dd|eee` 将 `aaa = bac` 和 `dd=eee` 取出来了
也就是传说中的pathinfo参数模式获取的参数。

#### 3.5  返回路由分发项

我们看到返回了模块 控制器 和方法 给`App` 中的`$dispatch` 变量

```php
$route = [$module, $controller, $action];
// more code here ...
return ['type' => 'module', 'module' => $route];
```

### 4. 路由分发

回到 `App.php` 中的`exec`方法，我们根据上面得到的 `type` 为 `module` 进入分支
```php
case 'module': // 模块/控制器/操作
                $data = self::module(
                    $dispatch['module'],
                    $config,
                    isset($dispatch['convert']) ? $dispatch['convert'] : null
                );
                break;
```

### 5. 模块加载

#### 5.1 获取方法名
```php
// 设置模块 设置过滤器 more code here

// 获取方法名
$actionName = strip_tags($result[2] ?: $config['default_action']);
if (!empty($config['action_convert'])) {
    $actionName = Loader::parseName($actionName, 1);
} else {
    $actionName = $convert ? strtolower($actionName) : $actionName;
}
```

#### 5.2 设置控制器和方法

```php
// 设置当前请求的控制器、操作
$request->controller(Loader::parseName($controller, 1))->action($actionName);
```
`Loader::parseName($controller, 1)` 控制器首字母大写 并将下划线去掉 紧跟的首字母大写

#### 5.3 加载控制器

```php
$instance = Loader::controller(
        $controller,
        $config['url_controller_layer'],
        $config['controller_suffix'],
        $config['empty_controller']
    );
```
这是控制器实例化的核心代码，具体实现如下：
```php
list($module, $class) = self::getModuleAndClass($name, $layer, $appendSuffix);
if (class_exists($class)) {
    return App::invokeClass($class);
}
```
核心调用了 `invokeClass`方法
```php
public static function invokeClass($class, $vars = [])
{
    $reflect     = new \ReflectionClass($class);
    $constructor = $reflect->getConstructor();
    $args        = $constructor ? self::bindParams($constructor, $vars) : [];

    return $reflect->newInstanceArgs($args);
}
```
根据反射类相关的特性，类进行了实例化，并返回实例化的对象

#### 5.4 方法的调用

```php
// 获取当前操作名
$action = $actionName . $config['action_suffix'];

$vars = [];
if (is_callable([$instance, $action])) {
    // 执行操作方法
    $call = [$instance, $action];
    // 严格获取当前操作方法名
    $reflect    = new \ReflectionMethod($instance, $action);
    $methodName = $reflect->getName();
    $suffix     = $config['action_suffix'];
    $actionName = $suffix ? substr($methodName, 0, -strlen($suffix)) : $methodName;
    $request->action($actionName);

} elseif (is_callable([$instance, '_empty'])) {
    // 空操作
    $call = [$instance, '_empty'];
    $vars = [$actionName];
} else {
    // 操作不存在
    throw new HttpException(404, 'method not exists:' . get_class($instance) . '->' . $action . '()');
}

Hook::listen('action_begin', $call);

return self::invokeMethod($call, $vars);

```
这里的核心代码是 

```php
self::invokeMethod($call, $vars);
```
该方法的实际操作如下 
```php
if (is_array($method)) {
    $class   = is_object($method[0]) ? $method[0] : self::invokeClass($method[0]);
    $reflect = new \ReflectionMethod($class, $method[1]);
} else {
    // 静态方法
    $reflect = new \ReflectionMethod($method);
}

$args = self::bindParams($reflect, $vars);

self::$debug && Log::record('[ RUN ] ' . $reflect->class . '->' . $reflect->name . '[ ' . $reflect->getFileName() . ' ]', 'info');

return $reflect->invokeArgs(isset($class) ? $class : null, $args);
```
获取方法对应的反射对象，然后注入对象和 参数，完成调用

这里需要注意的是：

类的静态方法调用不需要实例化对象 ： 
```php
$reflect = new \ReflectionMethod($method);
```
类的普通方法调用需要传递对象 ：  
```php
$reflect = new \ReflectionMethod($class, $method[1]);
```
方法的调用如下：
```php
$reflect->invokeArgs(isset($class) ? $class : null, $args)
```

至此，整个完整的路由调用已经完成





