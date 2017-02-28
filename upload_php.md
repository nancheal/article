#php文件上传绕过
>小密圈的一则协助，又是学到了新姿势，在出现上传限制的情况，如
```php
<?php $name=$_GET['name'];
if (preg_match ("/(\.php[0-9]$)|(\.php)/i", $name)) { 
die('hack attack'); 
 } else { 
  //写文件
 }
?>」
```
怎么去上传php文件

1. **配置不当导致其他后缀解析**
在mac下测试将httpd.conf里AddType application/x-httpd-php .pht后，就会导致pht文件解析为php文件了，你可能会有这样的疑惑，为什么会有这样的解析规则的存在，很简单pht，phtml都是一种方便记忆的后缀名，程序员会这样用，其次，在看网上的分析的时候，发现在debian系的发行版中，这类解析规则是默认存在的，默认有的还有
```php
php3,php4,php5,pht,phtml...
```

2. **.htaccess文件构成后门**
首先apche开启.htacess支持
```vim
Options FollowSymLinks
#.htaccess下
AllowOverride All
#
LoadModule rewrite_module modules/mod_rewrite.so 
```
上传.htaccess，将.jpg解析为php
```
AddType application/x-httpd-php .jpg
```
更多利用技巧[https://github.com/sektioneins/pcc/wiki/PHP-htaccess-injection-cheat-sheet](https://github.com/sektioneins/pcc/wiki/PHP-htaccess-injection-cheat-sheet)

3. **.user.ini文件构成后门**
在配置了fastcgi的服务器上会动态的加载这个用户自定义的php.ini文件，详细内容见[http://cb.drops.wiki/drops/tips-3424.html](http://cb.drops.wiki/drops/tips-3424.html)
上传.user.ini包含后门
```php
auto_prepend_file=01.gif
```