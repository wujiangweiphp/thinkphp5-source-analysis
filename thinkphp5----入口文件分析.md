我们从框架的结构里可以看出，tp5的入口文件为 `public/index.php`

```php
define('APP_PATH', __DIR__ . '/../application/');
require __DIR__ . '/../thinkphp/start.php';
```
这个文件就俩行代码，第一行是声明应用的绝对路径地址，第二行是引入了框架的启动文件。
我们接着看 `thinkphp/start.php`

```php
namespace think;
require __DIR__ . '/base.php';
App::run()->send();
```
这个启动文件也很简单，三行代码，第一行声明了命令空间为`think`,第二行引入了基础文件，
第三行调用了`App`的`run`方法，并链式调用了 `send` 方法，那么`run`返回了一个什么对象的实例，然后`send`方法又做了什么操作呢？我们接着来看，`base.php`

```php
....... 省略若干常量定义

// 载入Loader类
require CORE_PATH . 'Loader.php';

// 加载环境变量配置文件
if (is_file(ROOT_PATH . '.env')) {
    $env = parse_ini_file(ROOT_PATH . '.env', true);

    foreach ($env as $key => $val) {
        $name = ENV_PREFIX . strtoupper($key);

        if (is_array($val)) {
            foreach ($val as $k => $v) {
                $item = $name . '_' . strtoupper($k);
                putenv("$item=$v");
            }
        } else {
            putenv("$name=$val");
        }
    }
}

// 注册自动加载
\think\Loader::register();

// 注册错误和异常处理机制
\think\Error::register();

// 加载惯例配置文件
\think\Config::set(include THINK_PATH . 'convention' . EXT);
```
这个`base` 引入了加载器类 `Loader.php`,然后将环境变量的配置文件解析出来，存入全局的环境变量中，重点是最后的三句

倒数第二句注册异常处理
```php
\think\Error::register();
```
这句跟踪进去
```php
public static function register()
{
    error_reporting(E_ALL);
    set_error_handler([__CLASS__, 'appError']);
    set_exception_handler([__CLASS__, 'appException']);
    register_shutdown_function([__CLASS__, 'appShutdown']);
}
```
分别设置报错级别为全部，设置错误处理类，设置异常处理类，设置关闭处理类

倒数第三句 加载惯例配置
```php
\think\Config::set(include THINK_PATH . 'convention' . EXT);
```
这个实际上是加载的 `thinkphp/convention.php`,该文件和 `application/config.php` 的内容几乎一模一样，不同的是，这个都有默认值，使用`set`方法，是将所有的
配置添加的全局的静态变量中。

核心的方法是倒数第一句 注册自动加载
```php
\think\Loader::register();
```
我们进入加载器，找到这个方法
```php
public static function register($autoload = null)
{
    // 注册系统自动加载
    spl_autoload_register($autoload ?: 'think\\Loader::autoload', true, true);

    // Composer 自动加载支持
    if (is_dir(VENDOR_PATH . 'composer')) {
        if (PHP_VERSION_ID >= 50600 && is_file(VENDOR_PATH . 'composer' . DS . 'autoload_static.php')) {
            require VENDOR_PATH . 'composer' . DS . 'autoload_static.php';

            $declaredClass = get_declared_classes();
            $composerClass = array_pop($declaredClass);

            foreach (['prefixLengthsPsr4', 'prefixDirsPsr4', 'fallbackDirsPsr4', 'prefixesPsr0', 'fallbackDirsPsr0', 'classMap', 'files'] as $attr) {
                if (property_exists($composerClass, $attr)) {
                    self::${$attr} = $composerClass::${$attr};
                }
            }
        } else {
            self::registerComposerLoader();
        }
    }

    // 注册命名空间定义
    self::addNamespace([
        'think'    => LIB_PATH . 'think' . DS,
        'behavior' => LIB_PATH . 'behavior' . DS,
        'traits'   => LIB_PATH . 'traits' . DS,
    ]);

    // 加载类库映射文件
    if (is_file(RUNTIME_PATH . 'classmap' . EXT)) {
        self::addClassMap(__include_file(RUNTIME_PATH . 'classmap' . EXT));
    }

    self::loadComposerAutoloadFiles();

    // 自动加载 extend 目录
    self::$fallbackDirsPsr4[] = rtrim(EXTEND_PATH, DS);
}
```
1. 将`Loader`的`autoload`方法注册为系统的自动加载函数
2. 找到`vendor`目录,并引入'autoload_static.php'文件，为了获取`composer`生成的随机类名文件，这里使用了一个小技巧，使用'get_declared_classes' 方法获取用户自定义声明好的类，使用'array_pop'刚好获取这个类的名称
3. `self::addNamespace`添加命名空间定义，主要目的是使用命名空间能直接映射到对应目录的文件
4. `self::addClassMap` 获取已经缓存好的`类--目录`映射，这样加载的时候就不需要再查询了
5. `self::loadComposerAutoloadFiles` 这一句就是引入文件

```php
public static function loadComposerAutoloadFiles()
{
    foreach (self::$files as $fileIdentifier => $file) {
        if (empty($GLOBALS['__composer_autoload_files'][$fileIdentifier])) {
            __require_file($file);

            $GLOBALS['__composer_autoload_files'][$fileIdentifier] = true;
        }
    }
}
```

6. `self::$fallbackDirsPsr4[] = rtrim(EXTEND_PATH, DS)`就是加载我们自定义的第三方类库目录`extend`里的文件

看到这里我们还是没有提到`run`方法, 然后我们也并没有看到引入 `App`类的include语句。
仿佛是很神奇的就引入进来了，其实不然，我们回过头来看之前`Loader.php`的几个步骤

a. 第一步 将当前类的`autoload`方法注册为系统自动加载函数
b. 第三步`self::addNamespace` 将 
```php
'think'    => LIB_PATH . 'think' . DS,`
```
加入命令空间数组
c. `start.php`中第一句声明了命令空间为 `think`
```php
namespace think;
```
`start.php`中的第三句调用了 
```php
App::run()->start();
```
这里调用了`App`类，根据`a`中自动加载规则，自动查找 `think/app`类是否存在

为了验证我们设想的，在调用的时候动态加载该类所在的文件，我们分别在
`start.php` 中更改了部分代码，并添加了输出
```php
echo "-----------------add before ------------------<br/>";
echo App::$namespace , "<br/>";
echo "-----------------add after ------------------<br/>";
//App::run()->send();
```

`Loader.php`的`autoload`方法也增加了部分的输出：
```php
public static function autoload($class)
    {
        echo $class,"<br/>";
        // 检测命名空间别名
        if (!empty(self::$namespaceAlias)) {
            $namespace = dirname($class);
            if (isset(self::$namespaceAlias[$namespace])) {
                $original = self::$namespaceAlias[$namespace] . '\\' . basename($class);
                echo $original , "<br/>";
                if (class_exists($original)) {
                    return class_alias($original, $class, false);
                }
            }
        }

        if ($file = self::findFile($class)) {
            echo $file,"<br/>";
            // 非 Win 环境不严格区分大小写
            if (!IS_WIN || pathinfo($file, PATHINFO_FILENAME) == pathinfo(realpath($file), PATHINFO_FILENAME)) {
                __include_file($file);
                return true;
            }
        }

        return false;
    }
```
然后,我们浏览器访问框架，打印结果如下：
```
think\Route
/www/tp5/thinkphp/library/think/Route.php
think\Config
/www/tp5/thinkphp/library/think/Config.php
think\Validate
/www/tp5/thinkphp/library/think/Validate.php
think\Console
/www/tp5/thinkphp/library/think/Console.php
think\Error
/www/tp5/thinkphp/library/think/Error.php
-----------------add before ------------------
think\App
/www/tp5/thinkphp/library/think/App.php
app
-----------------add after ------------------
think\Log
/www/tp5/thinkphp/library/think/Log.php
```
果然不出我们所料，`App.php`确实是在使用的时候加载进来的

`App.php`加载进来了,我们接着看`Run`方法做了什么

```php
public static function run(Request $request = null)
{
    $request = is_null($request) ? Request::instance() : $request;

    try {
        $config = self::initCommon();

        // 模块/控制器绑定
        if (defined('BIND_MODULE')) {
            BIND_MODULE && Route::bind(BIND_MODULE);
        } elseif ($config['auto_bind_module']) {
            // 入口自动绑定
            $name = pathinfo($request->baseFile(), PATHINFO_FILENAME);
            if ($name && 'index' != $name && is_dir(APP_PATH . $name)) {
                Route::bind($name);
            }
        }

        $request->filter($config['default_filter']);

        // 默认语言
        Lang::range($config['default_lang']);
        // 开启多语言机制 检测当前语言
        $config['lang_switch_on'] && Lang::detect();
        $request->langset(Lang::range());

        // 加载系统语言包
        Lang::load([
            THINK_PATH . 'lang' . DS . $request->langset() . EXT,
            APP_PATH . 'lang' . DS . $request->langset() . EXT,
        ]);

        // 监听 app_dispatch
        Hook::listen('app_dispatch', self::$dispatch);
        // 获取应用调度信息
        $dispatch = self::$dispatch;

        // 未设置调度信息则进行 URL 路由检测
        if (empty($dispatch)) {
            $dispatch = self::routeCheck($request, $config);
        }

        // 记录当前调度信息
        $request->dispatch($dispatch);

        // 记录路由和请求信息
        if (self::$debug) {
            Log::record('[ ROUTE ] ' . var_export($dispatch, true), 'info');
            Log::record('[ HEADER ] ' . var_export($request->header(), true), 'info');
            Log::record('[ PARAM ] ' . var_export($request->param(), true), 'info');
        }

        // 监听 app_begin
        Hook::listen('app_begin', $dispatch);

        // 请求缓存检查
        $request->cache(
            $config['request_cache'],
            $config['request_cache_expire'],
            $config['request_cache_except']
        );

        $data = self::exec($dispatch, $config);
    } catch (HttpResponseException $exception) {
        $data = $exception->getResponse();
    }

    // 清空类的实例化
    Loader::clearInstance();

    // 输出数据到客户端
    if ($data instanceof Response) {
        $response = $data;
    } elseif (!is_null($data)) {
        // 默认自动识别响应输出类型
        $type = $request->isAjax() ?
        Config::get('default_ajax_return') :
        Config::get('default_return_type');

        $response = Response::create($data, $type);
    } else {
        $response = Response::create();
    }

    // 监听 app_end
    Hook::listen('app_end', $response);

    return $response;
}
```
代码比较长，我们分批进行分析。

```php
$request = is_null($request) ? Request::instance() : $request;
```
1. 首先我们根据为`null`的参数对`Request`进行了实例化

```php
$config = self::initCommon();
```
然后初始化通用配置信息，比如是否开启debug，设置时区，开启钩子之类的

```php
// 模块/控制器绑定
if (defined('BIND_MODULE')) {
    BIND_MODULE && Route::bind(BIND_MODULE);
} elseif ($config['auto_bind_module']) {
    // 入口自动绑定
    $name = pathinfo($request->baseFile(), PATHINFO_FILENAME);
    if ($name && 'index' != $name && is_dir(APP_PATH . $name)) {
        Route::bind($name);
    }
}
```
接着根据是否设置常量 `BIND_MODULE` 或者配置了 `auto_bind_module`将绑定的模块和控制器，绑定到路由。这里顺带引入了路由类

```php
$request->filter($config['default_filter']);
```
根据配置的`default_filter`对请求参数进行过滤

```php
Lang::range($config['default_lang']);
// 开启多语言机制 检测当前语言
$config['lang_switch_on'] && Lang::detect();
$request->langset(Lang::range());

// 加载系统语言包
Lang::load([
    THINK_PATH . 'lang' . DS . $request->langset() . EXT,
    APP_PATH . 'lang' . DS . $request->langset() . EXT,
]);
```
加载语言包

```php
// 监听 app_dispatch
Hook::listen('app_dispatch', self::$dispatch);
// 获取应用调度信息
$dispatch = self::$dispatch;

// 未设置调度信息则进行 URL 路由检测
if (empty($dispatch)) {
    $dispatch = self::routeCheck($request, $config);
}

// 记录当前调度信息
$request->dispatch($dispatch);
```
路由检查并记录调度信息

```php
// 请求缓存检查
$request->cache(
    $config['request_cache'],
    $config['request_cache_expire'],
    $config['request_cache_except']
);
```
请求缓存

```php
$data = self::exec($dispatch, $config);
```
对调度信息和配置执行获得数据结果

```php
Loader::clearInstance();
```
清空实例化

```php
if ($data instanceof Response) {
    $response = $data;
} elseif (!is_null($data)) {
    // 默认自动识别响应输出类型
    $type = $request->isAjax() ?
    Config::get('default_ajax_return') :
    Config::get('default_return_type');

    $response = Response::create($data, $type);
} else {
    $response = Response::create();
}

// 监听 app_end
Hook::listen('app_end', $response);

return $response;
```
请求结果实例化成一个 `Response`对象，并返回该对象

到这里，我们就知道了 `send` 方法原来是 `Response`类提供的

```php
public function send()
{
    // 监听response_send
    Hook::listen('response_send', $this);

    // 处理输出数据
    $data = $this->getContent();

    // Trace调试注入
    if (Env::get('app_trace', Config::get('app_trace'))) {
        Debug::inject($this, $data);
    }

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

    // 监听response_end
    Hook::listen('response_end', $this);

    // 清空当次请求有效的数据
    if (!($this instanceof RedirectResponse)) {
        Session::flush();
    }
}
```
设置头信息，缓存信息和输出响应信息，并刷新当次`session`,这里用到了一个比较有用的函数是 `fastcgi_finish_request()` ,该函数是立即响应并返回，不再等待后续代码执行（后续代码仍然在后台执行），该函数只存在于Linux环境下
