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

看到这里我们还是没有提到`run`方法... 未完待续
