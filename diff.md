#diff实用小结
>今天在diff文件的时候发现出现了这样情况：把看似相同的行当为了不同行输出。第一直觉就是空格的影响，总结一下解决的方法

**fuzz.php**
```php
a
<?php
mysql_connect("localhost","root","root");
mysql_select_db ("security");
mysql_query("set names utf8");
for($i = 0 ; $i < 256 ; $i++){
	$c = chr($i);
	$name = mysql_real_escape_string('Dumb' . $c);
	$sql = "SELECT * FROM users WHERE username = '{$name}'";
	$row = mysql_fetch_array(mysql_query($sql));
	if ($row['username'] == 'Dumb') {
		echo "{$c} <br/>";
	}
}
?>
```
**test.php**
```php
b
c
<?php
mysql_connect("localhost","root","root");
mysql_select_db ("security");
mysql_query("set names utf8");
for($i = 0 ; $i < 256 ; $i++){
	$c = chr($i);
	$name = mysql_real_escape_string('Dumb' . $c);
	$sql = "SELECT * FROM users WHERE username = '{$name}'";
	$row = mysql_fetch_array(mysql_query($sql));
	if ($row['username'] == 'Dumb') {
		echo "{$c} <br/>";
	}
}
?>
```
**一个最简单的diff命令和结果应该像下面这样**
```shell
➜  Sites diff  fuzz.php test.php
//"1,14" means first file [1,14] lines
//"c" means change ; "a" means add ; "d" means delete
//"1,15" means second file [1,15] lines
1,14c1,15
...(details)
```
**忽略空格的变化**
```shell
➜  Sites diff -b  fuzz.php test.php
1c1,2
< a
---
> b
> c
➜  Sites diff --help
-b  --ignore-space-change  Ignore changes in the amount of white space.
```
**输出模式**
1.**normal(默认模式)**
```shell
➜  Sites diff -b --normal fuzz.php test.php
1c1,2
< a
---
> b
> c
```
2.**context(上下文模式)**
```shell
➜  Sites diff -b -c fuzz.php test.php
*** fuzz.php	2017-04-10 11:49:30.000000000 +0800
--- test.php	2017-04-10 13:23:19.000000000 +0800
***************
*** 1,4 ****
! a
  <?php
  mysql_connect("localhost","root","root");
  mysql_select_db ("security");
--- 1,5 ----
! b
! c
  <?php
  mysql_connect("localhost","root","root");
  mysql_select_db ("security");
```
3.**unified(合并模式)**
```shell
➜  Sites diff -b -u fuzz.php test.php
--- fuzz.php	2017-04-10 11:49:30.000000000 +0800
+++ test.php	2017-04-10 13:23:19.000000000 +0800
@@ -1,4 +1,5 @@
-a
+b
+c
 <?php
 mysql_connect("localhost","root","root");
 mysql_select_db ("security");
```
4.**-y(类gui)**
```shell
➜  Sites diff -b -y fuzz.php test.php
a							      |	b
							      >	c
<?php								<?php
mysql_connect("localhost","root","root");			mysql_connect("localhost","root","root");
mysql_select_db ("security");					mysql_select_db ("security");
mysql_query("set names utf8");					mysql_query("set names utf8");
for($i = 0 ; $i < 256 ; $i++){					for($i = 0 ; $i < 256 ; $i++){
	$c = chr($i);							$c = chr($i);
	$name = mysql_real_escape_string('Dumb' . $c);			$name = mysql_real_escape_string('Dumb' . $c);
	$sql = "SELECT * FROM users WHERE username = '{$name}		$sql = "SELECT * FROM users WHERE username = '{$name}
	$row = mysql_fetch_array(mysql_query($sql));			$row = mysql_fetch_array(mysql_query($sql));
	if ($row['username'] == 'Dumb') {				if ($row['username'] == 'Dumb') {
		echo "{$c} <br/>";						echo "{$c} <br/>";
	}								}
}								}
?>								?>
```
可以发现在这种模式下，diff对两个文件进行了左右的显示，并且标出了不同的地方，如果觉得太拥挤可以，像下面这样,加上--suppress-common-lines来抑制相同输出
```shell
➜  Sites diff -b -y --suppress-common-lines fuzz.php test.php
a							      |	b
							      >	c
```
当然这样的话可能不太好分辨不同的位置具体是哪一行，这种情况我选择vimdiff...