#编码小结
>又是小密圈里的一个问题

**source-code**
```html
<?php
session_start();
require('conn.php');
$action = isset($_GET['action']) ? $_GET['action'] : '';

if ($action === 'login') {
    $username = isset($_POST['username']) ? addslashes(strtolower(trim($_POST['username']))) : '';
    $password = isset($_POST['password']) ? md5($_POST['password']) : '';
    if($username == ''){
        die('Invalid username!');
    }
    if ($username === 'admin') {
        if ($_SERVER['REMOTE_ADDR'] !== '127.0.0.1') {
            die('Permission denied!');
        }
    }
    $result = $mysqli->query("SELECT * FROM z_users where username = '{$username}' and password = '{$password}'");
    $row = mysqli_fetch_array($result);
    if(isset($row['username'])){
        $_SESSION['username'] = $username;
        $_SESSION['flag'] = $row['flag'];
        header('location: ./index.php?action=main');
    }else{
        echo "Invalid username or password";
    }
    exit;
}elseif($action === 'main'){
    if(!isset($_SESSION['username'])){
        header('location: ./index.php');
        exit;
    }
    echo "Hello, " . $_SESSION['username'].", ".$_SESSION['flag']."<br>\n";
} else {
    if(isset($_SESSION['username'])){
        header('location: ./index.php?action=main');
        exit;
    }
    ?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="zh-CN" lang="zh-CN">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="author" content="Ambulong">
<title>LOGIN</title>
</head>
<body>
<form action="./index.php?action=login" method="POST">
username: <input name="username" type="text"><br>
password: <input name="password" type="password"><br>  
<input name="action" value='login' type="hidden"><br>
<input name="submit" type="submit"><br>
</form>
</body>
</html>
<?php
}
show_source(__FILE__);
```
ps:这里的password是admin，我特意去找了之前一道雷同的题目，里面是给了账号、密码的

**几个编码的问题**
```mysql
mysql> show variables like "character_set_%";
+--------------------------+------------------------------------------------------+
| Variable_name            | Value                                                |
+--------------------------+------------------------------------------------------+
| character_set_client     | utf8                                                 |
| character_set_connection | utf8                                                 |
| character_set_database   | gbk                                                  |
| character_set_filesystem | binary                                               |
| character_set_results    | utf8                                                 |
| character_set_server     | utf8                                                 |
| character_set_system     | utf8                                                 |
```
看结果，在客户端上client，connection的编码都为utf8，而在服务端上为gbk，那么传入的数据也就会进行“utf-8 -> utf-8 -> gbk”这样的变换，那么这其实是一个大的编码集到一个小的编码集变换的过程，（虽然这样说有失准确，但是utf-8作为万国码确实涵盖了gbk里的字符）。
那么在这样的变换中，如果我们传入一个在utf-8里，却不在gbk里的字符会怎么样呢？报错，转换报错，那如果传入不全的utf-8呢（utf-8为一种变长的编码）？忽略，这是p师傅的博客偷学来的，确实很叼[https://www.leavesongs.com/PENETRATION/mysql-charset-trick.html](https://www.leavesongs.com/PENETRATION/mysql-charset-trick.html)(p师傅用的latin编码也可以认为是子集)，也就是说mysql对下面的查询是一视同仁的:
127.0.0.1/test.php?id=%c2
127.0.0.1/test.php

**test.php**
```php
<?php
$mysqli = new mysqli('localhost','root','root','security');
if(mysqli_connect_errno()){
    echo mysqli_connect_error();
    die;
}
$mysqli->set_charset('utf8');
$sql = 'SELECT * FROM users WHERE username = ?';
$stmt = $mysqli->stmt_init();
$stmt->prepare($sql);
$c = $_GET['id'];
$conditions = 'Dumb'. $c;
echo $conditions;
$stmt->bind_param('s',$conditions);
$stmt->execute();
$result = $stmt->get_result();
$data = $result->fetch_all(MYSQL_ASSOC);
var_dump($data);
?>
```
参考:[https://www.leavesongs.com/PENETRATION/mysql-charset-trick.html](https://www.leavesongs.com/PENETRATION/mysql-charset-trick.html)
