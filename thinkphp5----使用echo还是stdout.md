# tp5控制台输出使用echo还是stdout

```php 
$stream = @fopen('php://stdout', 'w') ?: fopen('php://output', 'w');
$message = 'hello world';
$newline = true;
@fwrite($stream, $message . ($newline ? PHP_EOL : ''));
fflush($stream);
```

为什么不是用echo，而是用fwrite呢

```php
ob_start();
$stream = @fopen('php://stdout', 'w');
$message = 'hello world';
$newline = true;
@fwrite($stream, $message . ($newline ? PHP_EOL : ''));
@fwrite($stream, $message . ($newline ? PHP_EOL : ''));
@fwrite($stream, $message . ($newline ? PHP_EOL : ''));
fflush($stream);

echo  $message ,"\n";
echo  $message ,"\n";
echo  $message ,"\n";
while(1){
    $ob_contents = ob_get_clean();
    sleep(3);
    echo  "after 3s \n";
    print "$ob_contents\n";
    break;
}
```
打印结果：
<pre>
hello world
hello world
hello world
after 3s
hello world
hello world
hello world
</pre>

如果我们将`stdout`换成`output`
```php
$stream = fopen('php://output', 'w');
```

打印结果：
<pre>
after 3s
hello world
hello world
hello world
hello world
hello world
hello world
</pre>

很显然，`stdout`会直接绕过`ob`系列缓冲区函数，`output` 则会被缓冲

