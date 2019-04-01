# thinkphp5-source-analysis

> 本项目将对Thinkphp5源码及使用作一个简要的分析，由于本人能力有限，难免会有错误遗漏，如有任何不妥之处，欢迎评论指正。

## thinkphp5 目录结构解析

<pre>
<h3>
|_application 应用目录
   |_index    模块
       |_controller  控制器
            |_index.php  控制器类
   |_extra   额外的配置文件目录
   |_command.php  控制台（console）配置文件
   |_common.php   公共函数存放文件
   |_config.php   公共配置（缓存/session/debug/路由模式等）
   |_route.php    路由配置文件
   |_tags.php     行为配置文件（类似于钩子Hook文件对各种行为进行监听）
|_extend   第三方扩展库存放目录
|_public   访问目录
    |_static      静态资源存放目录
    |_robots.txt  机器人抓取规则文件
    |_index.php   访问入口文件
|_runtime  运行资源存放目录
   |_log   日志文件存放目录
   |_cache 缓存文件存放目录
|_thinkphp 框架核心文件存放目录 
|_vendor   composer文件存放目录 
|_build.php  构建初始应用配置文件
|_composer.json  composer配置文件
|_think  框架命令引导shell文件
</h3>
</pre>

