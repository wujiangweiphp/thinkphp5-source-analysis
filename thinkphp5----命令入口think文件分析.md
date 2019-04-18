# thinkphp5命令入口think文件分析.md 

* [thinkphp5\-\-\-\-命令入口文件分析\.md](#thinkphp5----%E5%91%BD%E4%BB%A4%E5%85%A5%E5%8F%A3%E6%96%87%E4%BB%B6%E5%88%86%E6%9E%90md)
    * [1\.入口think](#1%E5%85%A5%E5%8F%A3think)
    * [2\. console\.php](#2-consolephp)
      * [2\.1 应用初始化App::initCommon](#21-%E5%BA%94%E7%94%A8%E5%88%9D%E5%A7%8B%E5%8C%96appinitcommon)
        * [2\.1\.1 初始化self::init](#211-%E5%88%9D%E5%A7%8B%E5%8C%96selfinit)
      * [2\.2 Console::init()](#22-consoleinit)
        * [2\.2\.1 实例化console](#221-%E5%AE%9E%E4%BE%8B%E5%8C%96console)
        * [2\.2\.2 获取默认输入定义getDefaultInputDefinition()](#222-%E8%8E%B7%E5%8F%96%E9%BB%98%E8%AE%A4%E8%BE%93%E5%85%A5%E5%AE%9A%E4%B9%89getdefaultinputdefinition)
        * [2\.2\.3 获取默认的命令getDefaultCommands()](#223-%E8%8E%B7%E5%8F%96%E9%BB%98%E8%AE%A4%E7%9A%84%E5%91%BD%E4%BB%A4getdefaultcommands)
        * [2\.2\.4 加入新命令add(new $command())](#224-%E5%8A%A0%E5%85%A5%E6%96%B0%E5%91%BD%E4%BB%A4addnew-command)
      * [2\.3 run方法](#23-run%E6%96%B9%E6%B3%95)
        * [2\.3\.0 收集参数和准备输出流](#230-%E6%94%B6%E9%9B%86%E5%8F%82%E6%95%B0%E5%92%8C%E5%87%86%E5%A4%87%E8%BE%93%E5%87%BA%E6%B5%81)
        * [2\.3\.1 configureIO方法](#231-configureio%E6%96%B9%E6%B3%95)
        * [2\.3\.2 doRun方法](#232-dorun%E6%96%B9%E6%B3%95)
        * [2\.3\.3 执行命令doRunCommand](#233-%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4doruncommand)

### 1.入口think

```php
#!/usr/bin/env php
<?php
// 定义项目路径
define('APP_PATH', __DIR__ . '/application/');
// 加载框架引导文件
require __DIR__.'/thinkphp/console.php';
```
来看第一句
```
#!/usr/bin/env php
```
这句是许多shell文件声明文件执行的软件命令，一般我们常见的是下面这种
```
#!/usr/bin/php
```
这俩句有什么区别呢，为什么使用第一句而不是用更简洁的第二句？
第二句指明的是执行软件必须在 `/usr/bin/php` 有这`php`软连接或者可执行文件存在
而第一句是不管这个目录存不存在`php`,只要'PATH'中存放了对应的`php`的环境配置路径，会自动去取第一条，这样避免了部分没有设置软连接的仍然能正常执行

### 2. console.php

接着我们来看代码，第一句声明应用路径，第二句引入入口文件 `console.php`，我们打开这个文件：

```php
namespace think;

// ThinkPHP 引导文件
require __DIR__ . '/base.php';

// 执行应用
App::initCommon();
Console::init();
```
引入引导文件 `base.php` ,[入口文件](https://github.com/wujiangweiphp/thinkphp5-source-analysis/blob/master/thinkphp5----%E5%85%A5%E5%8F%A3%E6%96%87%E4%BB%B6%E5%88%86%E6%9E%90.md) 里也是引入的相同的文件，主要作用是
设置全局路径常量，初始化全局环境配置，引入加载器，引入异常处理，引入配置文件

#### 2.1 应用初始化`App::initCommon`

我们来看下面这句，干了啥：
```php
App::initCommon();
```
跟踪进去找到文件`tp5/thinkphp/library/think/App.php`
```php
public static function initCommon()
{
    if (empty(self::$init)) {
        if (defined('APP_NAMESPACE')) {
            self::$namespace = APP_NAMESPACE;
        }
        Loader::addNamespace(self::$namespace, APP_PATH);

        // 初始化应用
        $config       = self::init();
        self::$suffix = $config['class_suffix'];

        // 应用调试模式
        self::$debug = Env::get('app_debug', Config::get('app_debug'));

        if (!self::$debug) {
            ini_set('display_errors', 'Off');
        } elseif (!IS_CLI) {
            // 重新申请一块比较大的 buffer
            if (ob_get_level() > 0) {
                $output = ob_get_clean();
            }
            ob_start();
            if (!empty($output)) {
                echo $output;
            }
        }

        if (!empty($config['root_namespace'])) {
            Loader::addNamespace($config['root_namespace']);
        }

        // 加载额外文件
        if (!empty($config['extra_file_list'])) {
            foreach ($config['extra_file_list'] as $file) {
                $file = strpos($file, '.') ? $file : APP_PATH . $file . EXT;
                if (is_file($file) && !isset(self::$file[$file])) {
                    include $file;
                    self::$file[$file] = true;
                }
            }
        }

        // 设置系统时区
        date_default_timezone_set($config['default_timezone']);

        // 监听 app_init
        Hook::listen('app_init');
        self::$init = true;
    }
    return Config::get();
}
```
第一句是判断是否已经初始化，接下来三句
```php
if (defined('APP_NAMESPACE')) {
    self::$namespace = APP_NAMESPACE;
}
Loader::addNamespace(self::$namespace, APP_PATH);
```
如果未定义应用常量，则默认加入的应用配置命名空间是 `app`(该 `APP`类的`$namespace`默认值是`app`)

```php
$config  = self::init();
```
初始化应用
```php
self::$suffix = $config['class_suffix'];
```
这句话设置类的后缀，如后缀为 `Controller` 则在 `ParseUrl` 时会自动生成类名为 `ClassNameController` 然后自动引入

```php
// 应用调试模式
self::$debug = Env::get('app_debug', Config::get('app_debug'));
if (!self::$debug) {
    ini_set('display_errors', 'Off');
} elseif (!IS_CLI) {
    // 重新申请一块比较大的 buffer
    if (ob_get_level() > 0) {
        $output = ob_get_clean();
    }
    ob_start();
    if (!empty($output)) {
        echo $output;
    }
}
```
根据环境变量中的 `app_debug` 查看是否开启debug，如果未开启 则设置关闭报错，但是如果不是，好像也没有说开启
的意思，`base.php` 中有一句
```php
\think\Error::register();
```
这个方法如下
```php
public static function register()
{
    error_reporting(E_ALL);
    set_error_handler([__CLASS__, 'appError']);
    set_exception_handler([__CLASS__, 'appException']);
    register_shutdown_function([__CLASS__, 'appShutdown']);
}
```
这里就开启了所有的报错信息，然后设置了处理函数

```php
if(!IS_CLI) {
  // 重新申请一块比较大的 buffer
  if (ob_get_level() > 0) {
      $output = ob_get_clean();
  }
  ob_start();
  if (!empty($output)) {
      echo $output;
  }
}
```
`IS_CLI` 是否为命令行 可以通过全局常量 `PHP_SAPI == 'cli'` 来设置
`ob_get_level()` 当前队伍最末尾的缓冲块的序列号 默认是1
`ob_get_clean()` 得到当前缓冲区的内容并删除当前输出缓冲区 他其实是执行了 `ob_get_contents()` 和 `ob_end_clean()` 两步操作
`ob_start()` 开启一个新的缓冲区块 如果之前有缓冲内容，直接输出

```php
if (!empty($config['root_namespace'])) {
    Loader::addNamespace($config['root_namespace']);
}
```
添加额外的命名空间 --- 用来引入新的文件配置目录及模块 比如 `api/chat/controller/group.php`
可以配置为：

```php
'root_namespace' => [
    'chat'  => '../api/chat/',
]
```
下面就剩下加载额外的帮助函数列表
```php
if (!empty($config['extra_file_list'])) {
    foreach ($config['extra_file_list'] as $file) {
        $file = strpos($file, '.') ? $file : APP_PATH . $file . EXT;
        if (is_file($file) && !isset(self::$file[$file])) {
            include $file;
            self::$file[$file] = true;
        }
    }
}
```
`config.php` 中的对应配置是 `common.php` 文件，我们可以在同类目录中添加 `helper.php` 等帮助函数

剩下的几句就是设置时区和设置监听初始化，接下来我们实际看一下初始化了什么


##### 2.1.1 初始化self::init

```php
private static function init($module = '')
{
    // 定位模块目录
    $module = $module ? $module . DS : '';

    // 加载初始化文件
    if (is_file(APP_PATH . $module . 'init' . EXT)) {
        include APP_PATH . $module . 'init' . EXT;
    } elseif (is_file(RUNTIME_PATH . $module . 'init' . EXT)) {
        include RUNTIME_PATH . $module . 'init' . EXT;
    } else {
        // 加载模块配置
        $config = Config::load(CONF_PATH . $module . 'config' . CONF_EXT);

        // 读取数据库配置文件
        $filename = CONF_PATH . $module . 'database' . CONF_EXT;
        Config::load($filename, 'database');

        // 读取扩展配置文件
        if (is_dir(CONF_PATH . $module . 'extra')) {
            $dir   = CONF_PATH . $module . 'extra';
            $files = scandir($dir);
            foreach ($files as $file) {
                if ('.' . pathinfo($file, PATHINFO_EXTENSION) === CONF_EXT) {
                    $filename = $dir . DS . $file;
                    Config::load($filename, pathinfo($file, PATHINFO_FILENAME));
                }
            }
        }

        // 加载应用状态配置
        if ($config['app_status']) {
            Config::load(CONF_PATH . $module . $config['app_status'] . CONF_EXT);
        }

        // 加载行为扩展文件
        if (is_file(CONF_PATH . $module . 'tags' . EXT)) {
            Hook::import(include CONF_PATH . $module . 'tags' . EXT);
        }

        // 加载公共文件
        $path = APP_PATH . $module;
        if (is_file($path . 'common' . EXT)) {
            include $path . 'common' . EXT;
        }

        // 加载当前模块语言包
        if ($module) {
            Lang::load($path . 'lang' . DS . Request::instance()->langset() . EXT);
        }
    }

    return Config::get();
}
```
先看前五句
```php
$module = $module ? $module . DS : '';
// 加载初始化文件
if (is_file(APP_PATH . $module . 'init' . EXT)) {
    include APP_PATH . $module . 'init' . EXT;
} elseif (is_file(RUNTIME_PATH . $module . 'init' . EXT)) {
    include RUNTIME_PATH . $module . 'init' . EXT;
}
```
如果定义了模块，则首先会引入模块根目录中的 `init.php` 初始化文件（一般比较适合做权限 和 部分基础参数的初始化）

```php
$config = Config::load(CONF_PATH . $module . 'config' . CONF_EXT);
```
加载当前模块的`config.php` 配置文件，这就说明了每个模块在未创建`init.php`的前提下，我们都可以创建`config.php`文件
并且会被框架自动引入，这也说明了 `init.php` 的作用也是配置

```php
$filename = CONF_PATH . $module . 'database' . CONF_EXT;
Config::load($filename, 'database');
```
按模块读取数据配置文件，在没有 'init.php' 的前提下，也是可以每个模块都加一个 `database.php` 文件的

```php
if (is_dir(CONF_PATH . $module . 'extra')) {
    $dir   = CONF_PATH . $module . 'extra';
    $files = scandir($dir);
    foreach ($files as $file) {
        if ('.' . pathinfo($file, PATHINFO_EXTENSION) === CONF_EXT) {
            $filename = $dir . DS . $file;
            Config::load($filename, pathinfo($file, PATHINFO_FILENAME));
        }
    }
}
```
tp5 默认的配置文件目录也是 `application` 那么配置文件就是存放在 `application/extra` 目录


【注意】上面所有有关`module`有值的假设目前 TP5 中并没有地方有传过，所以不会访问对应的模块相关的配置文件

```php
Config::load(CONF_PATH . $module . $config['app_status'] . CONF_EXT);
```
加载场景配置文件，会覆盖现有的配置文件 比如家里的工作环境数据库什么的 可以定义个`home.php` 或者办公司的 `office.php`

```php
if (is_file(CONF_PATH . $module . 'tags' . EXT)) {
    Hook::import(include CONF_PATH . $module . 'tags' . EXT);
}
```
加载tags.php


```php
$path = APP_PATH . $module;
if (is_file($path . 'common' . EXT)) {
    include $path . 'common' . EXT;
}
```
加载 `common.php`

```php
 if ($module) {
    Lang::load($path . 'lang' . DS . Request::instance()->langset() . EXT);
}
```
实例化语言设置

#### 2.2 Console::init()

```php
public static function init($run = true)
{
    static $console;
    if (!$console) {
        $config = Config::get('console');
        // 实例化 console
        $console = new self($config['name'], $config['version'], $config['user']);
        // 读取指令集
        if (is_file(CONF_PATH . 'command' . EXT)) {
            $commands = include CONF_PATH . 'command' . EXT;

            if (is_array($commands)) {
                foreach ($commands as $command) {
                    class_exists($command) &&
                    is_subclass_of($command, "\\think\\console\\Command") &&
                    $console->add(new $command());  // 注册指令
                }
            }
        }
    }
    return $run ? $console->run() : $console;
}
```
没有实例化进入if判断
```php
$config = Config::get('console');
```
好像`config.php`中并没有 `console`这个选项，但是能打印出来内容如下
```
Array
(
    [name] => Think Console
    [version] => 0.1
    [user] =>
)
```
全局搜索，能查询到来自 `tp5/thinkphp/convention.php`
这个文件什么时候引进来的呢，我们记得之前的 `console.php` 中有引入 `base.php`这个文件的最后一句

```php
\think\Config::set(include THINK_PATH . 'convention' . EXT);
```
实例化console
```php
$console = new self($config['name'], $config['version'], $config['user']);
```
加载命令配置文件 `command.php` 命令配置文件
```php
if (is_array($commands)) {
    foreach ($commands as $command) {
        class_exists($command) &&
        is_subclass_of($command, "\\think\\console\\Command") &&
        $console->add(new $command());  // 注册指令
    }
}
```
命令类存在，且是`think\console\Command` 的子类，然后加入新的命令实例


##### 2.2.1 实例化console

```php
public function __construct($name = 'UNKNOWN', $version = 'UNKNOWN', $user = null)
{
    $this->name    = $name;
    $this->version = $version;
    if ($user) {
        $this->setUser($user);
    }

    $this->defaultCommand = 'list';
    $this->definition     = $this->getDefaultInputDefinition();

    foreach ($this->getDefaultCommands() as $command) {
        $this->add($command);
    }
}
```
前五句是名称用户的配置
```php
$this->defaultCommand = 'list';
```
默认命令设置为`list` 
```php
$this->definition     = $this->getDefaultInputDefinition();
```
获取默认输入的定义说明
```php
foreach ($this->getDefaultCommands() as $command) {
    $this->add($command);
}
```
获取默认的命令对象，并加入


##### 2.2.2 获取默认输入定义getDefaultInputDefinition()

```php
protected function getDefaultInputDefinition()
{
    return new InputDefinition([
        new InputArgument('command', InputArgument::REQUIRED, 'The command to execute'),
        new InputOption('--help', '-h', InputOption::VALUE_NONE, 'Display this help message'),
        new InputOption('--version', '-V', InputOption::VALUE_NONE, 'Display this console version'),
        new InputOption('--quiet', '-q', InputOption::VALUE_NONE, 'Do not output any message'),
        new InputOption('--verbose', '-v|vv|vvv', InputOption::VALUE_NONE, 'Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug'),
        new InputOption('--ansi', '', InputOption::VALUE_NONE, 'Force ANSI output'),
        new InputOption('--no-ansi', '', InputOption::VALUE_NONE, 'Disable ANSI output'),
        new InputOption('--no-interaction', '-n', InputOption::VALUE_NONE, 'Do not ask any interactive question'),
    ]);
}
```
返回了多个实例化对象，这也是新加入自定义命令实例化所需要的设置的
```php
new InputArgument('command', InputArgument::REQUIRED, 'The command to execute'),
```
输入命令
```php
new InputOption('--help', '-h', InputOption::VALUE_NONE, 'Display this help message'),
```
输入参数

##### 2.2.3 获取默认的命令getDefaultCommands()
```php
 protected function getDefaultCommands()
{
    $defaultCommands = [];
    foreach (self::$defaultCommands as $class) {
        if (class_exists($class) && is_subclass_of($class, "think\\console\\Command")) {
            $defaultCommands[] = new $class();
        }
    }
    return $defaultCommands;
}
```
实例化默认的命令，默认的命令如下：
```php
private static $defaultCommands = [
    "think\\console\\command\\Help",
    "think\\console\\command\\Lists",
    "think\\console\\command\\Build",
    "think\\console\\command\\Clear",
    "think\\console\\command\\make\\Controller",
    "think\\console\\command\\make\\Model",
    "think\\console\\command\\optimize\\Autoload",
    "think\\console\\command\\optimize\\Config",
    "think\\console\\command\\optimize\\Route",
    "think\\console\\command\\optimize\\Schema",
];
```

##### 2.2.4 加入新命令add(new $command())

```php
public function add(Command $command)
{
    if (!$command->isEnabled()) {
        $command->setConsole(null);
        return false;
    }
    $command->setConsole($this);
    if (null === $command->getDefinition()) {
        throw new \LogicException(
            sprintf('Command class "%s" is not correctly initialized. You probably forgot to call the parent constructor.', get_class($command))
        );
    }
    $this->commands[$command->getName()] = $command;
    foreach ($command->getAliases() as $alias) {
        $this->commands[$alias] = $command;
    }
    return $command;
}
```
加入的命令类需要满足的条件 

包含方法：
```php
public function isEnabled():bool; 是否启用 父类默认启用
public function setConsole()  父类Command已经设置过
public function getDefinition() 获取默认的定义
public function getAliases() 别名
```

#### 2.3 run方法

```php
public function run()
{
    $input  = new Input();
    $output = new Output();
    $this->configureIO($input, $output);
    try {
        $exitCode = $this->doRun($input, $output);
    } catch (\Exception $e) {
        if (!$this->catchExceptions) throw $e;

        $output->renderException($e);

        $exitCode = $e->getCode();

        if (is_numeric($exitCode)) {
            $exitCode = ((int) $exitCode) ?: 1;
        } else {
            $exitCode = 1;
        }
    }
    if ($this->autoExit) {
        if ($exitCode > 255) $exitCode = 255;
        exit($exitCode);
    }
    return $exitCode;
}
```
实例化输入`Input`和输出`Output`类,然后配置 `configureIO()` 再调用 `doRun`方法

##### 2.3.0 收集参数和准备输出流
我们看 `new Input()`做了什么
```php
public function __construct($argv = null)
{
  if (null === $argv) {
      $argv = $_SERVER['argv'];
      // 去除命令名
      array_shift($argv);
  }
  $this->tokens = $argv;
  $this->definition = new Definition();
}
```
很明显这里是做了 命令行上的参数收集 `$_SERVER['argv']`
再看看 `new Output()`做了什么
```php
public function __construct($driver = 'console')
{
  $class = '\\think\\console\\output\\driver\\' . ucwords($driver);
  $this->handle = new $class($this);
}
```
设置处理类 为'Console'
```php
public function __construct(Output $output)
 {
     $this->output    = $output;
     $this->formatter = new Formatter();
     $this->stdout    = $this->openOutputStream();
     $decorated       = $this->hasColorSupport($this->stdout);
     $this->formatter->setDecorated($decorated);
 }
```
处理类做了打开实例化的输出流，并设置显示格式和优化显示，这样我们在输出的时候，就直接使用`$output->write()`写进输出流了


##### 2.3.1 configureIO方法

```php
protected function configureIO(Input $input, Output $output)
{
    if (true === $input->hasParameterOption(['--ansi'])) {
        $output->setDecorated(true);
    } elseif (true === $input->hasParameterOption(['--no-ansi'])) {
        $output->setDecorated(false);
    }

    if (true === $input->hasParameterOption(['--no-interaction', '-n'])) {
        $input->setInteractive(false);
    }

    if (true === $input->hasParameterOption(['--quiet', '-q'])) {
        $output->setVerbosity(Output::VERBOSITY_QUIET);
    } else {
        if ($input->hasParameterOption('-vvv') || $input->hasParameterOption('--verbose=3') || $input->getParameterOption('--verbose') === 3) {
            $output->setVerbosity(Output::VERBOSITY_DEBUG);
        } elseif ($input->hasParameterOption('-vv') || $input->hasParameterOption('--verbose=2') || $input->getParameterOption('--verbose') === 2) {
            $output->setVerbosity(Output::VERBOSITY_VERY_VERBOSE);
        } elseif ($input->hasParameterOption('-v') || $input->hasParameterOption('--verbose=1') || $input->hasParameterOption('--verbose') || $input->getParameterOption('--verbose')) {
            $output->setVerbosity(Output::VERBOSITY_VERBOSE);
        }
    }
}
```
核心就四个方法
```
//输入
$input->hasParameterOption('-vvv')       //是否有选项 -vvv
$input->getParameterOption('--verbose')  //获取选项 -verbose
$input->setInteractive(false);           //设置无交互  
//输出
$output->setVerbosity(Output::VERBOSITY_QUIET); //设置静默输出
$output->setDecorated(true);  //是否美化文字
```

##### 2.3.2 doRun方法

```php
public function doRun(Input $input, Output $output)
{
    // 获取版本信息
    if (true === $input->hasParameterOption(['--version', '-V'])) {
        $output->writeln($this->getLongVersion());
        return 0;
    }
    $name = $this->getCommandName($input);
    // 获取帮助信息
    if (true === $input->hasParameterOption(['--help', '-h'])) {
        if (!$name) {
            $name  = 'help';
            $input = new Input(['help']);
        } else {
            $this->wantHelps = true;
        }
    }
    if (!$name) {
        $name  = $this->defaultCommand;
        $input = new Input([$this->defaultCommand]);
    }
    return $this->doRunCommand($this->find($name), $input, $output);
}
```
前三行是检查参数中是否包含版本参数
```php
if (true === $input->hasParameterOption(['--version', '-V'])) {
    $output->writeln($this->getLongVersion());
    return 0;
}
```
获取版本信息 并直接换行输出版本信息

```php
$name = $this->getCommandName($input);
// 获取帮助信息
if (true === $input->hasParameterOption(['--help', '-h'])) {
    if (!$name) {
        $name  = 'help';
        $input = new Input(['help']);
    } else {
        $this->wantHelps = true;
    }
}
if (!$name) {
    $name  = $this->defaultCommand;
    $input = new Input([$this->defaultCommand]);
}
```
获取对应的命令名称，并推送给 `Input`实例化
```php
$this->doRunCommand($this->find($name), $input, $output)
```
`find`根据名称查找命令，会最终去`$this->commands`维护的命令对应的对象中查找


##### 2.3.3 执行命令doRunCommand
```php
protected function doRunCommand(Command $command, Input $input, Output $output)
{
    return $command->run($input, $output);
}
```
最终会调用用户的实例化类中的`run` 方法
