#php缺陷小结
> php在某些方面的缺陷可能会导致程序的安全漏洞

1. **类型转换带来的问题**

	php的弱类型在带来便利的同时，也带来了很多的问题
    
    **== 问题**
    
    连等在比较字符串和数字时，会将字符串转为数字，像下面这样，而三等则不会出现
    ```php
    <?php
    //bool(true)
    var_dump(''==null);
    var_dump('0'==0);
    var_dump('abc'==0);
    var_dump('1abc'==1);
    var_dump(0==null);
    //十六进制转换问题
    var_dump("0x1e240"=="123456");
    ?>
    ```
	其在比较形如0e/d+的字符串（后面无其他字符）时，也会出现问题，其会将字符串解析为科学计数法，并且都等于0
    ```php
    <?php
    //bool(true)
    var_dump('0e123'=='0e456');
    var_dump('0e123'==0);
    var_dump('0e456'==0);
    ?>
    ```
    这样的话在md5比较的时候可能会有问题，像下面这样
    ```php
    <?php
    var_dump(md5('QNKCDZO'));//string(32) "0e830400451993494058024219903391"
    var_dump(md5('240610708'));//string(32) "0e462097431906509019562988736854"
    var_dump(md5('QNKCDZO')==md5('240610708'));//bool(true)
    ?>
    ```
    **md5 问题**
    
    上面提到了md5，其实md5本身还有问题，像这样
    ```php
    <?php
    var_dump(md5([1]));//NULL
	var_dump(md5([2]));//NULL
    ?>
    ```
    **in_array()问题**
    
    同理，也是类型转换产生的问题
    ```php
    <?php
    $array=[0,1,2,'3'];
    var_dump(in_array('abc',$array));//bool(true)
    var_dump(in_array('1abc',$array));//bool(true)
    ?>
    ```
    **switch,inteval问题**
    ```php
    <?php
    if(intval($a)>1000) {
    mysql_query("select * from news where id=".$a)
	}
	?>
	```
    ```php
    <?php
    $i ="2abc";
    switch ($i) {
    case 0:
    case 1:
    case 2:
    echo "i is less than 3 but not negative";
    break;
	case 3:
    echo "i is 3";
	}
    ?>
    ```
2. **错误的用法**

	**strcmp(str1,str2)问题**
    
    strcmp本质是在做字符串的减法运算:
    str1 < str2返回-1;
    str1 = str2返回0;
    str1 > str2返回1;
    ```php
    <?php
    var_dump(strcmp([1],'123'));//NULL
    ?>
    ```
    这里就可以连等相等的返回值0了
3. **脚本端与数据库端的语法差异**
	
    **is_numeric问题**
    
    mysql支持十六进制的语句，在脚本检测数据是否为数字的时候，如果查询语句中卷入了十六进制编码的恶意数据，那么问题就产生了