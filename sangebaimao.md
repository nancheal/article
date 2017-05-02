#三个白帽回顾
>今天看见一段代码很像之前三个白帽里的题，搜索了一下，发现了这道题的其他解法，小本本记下来

```php
<?php
if(isset($_GET['source'])){
    highlight_file(__FILE__);
    exit;
}
include_once("flag.php");
 /*
    shougong check if the $number is a palindrome number(hui wen shu)
 */
function is_palindrome_number($number) {
    $number = strval($number);
    $i = 0;
    $j = strlen($number) - 1;
    while($i < $j) {
        if($number[$i] !== $number[$j]) {
            return false;
        }
        $i++;
        $j--;
    }
    return true;
}
ini_set("display_error", false);
error_reporting(0);
$info = "";
$req = [];
foreach([$_GET, $_POST] as $global_var) {
    foreach($global_var as $key => $value) {
        $value = trim($value);
        is_string($value) && is_numeric($value) && $req[$key] = addslashes($value);
    }
}   
$n1 = intval($req["number"]);
$n2 = intval(strrev($req["number"]));
if($n1 && $n2) {
    if ($req["number"] != intval($req["number"])) {
        $info = "number must be integer!";
    } elseif ($req["number"][0] == "+" || $req["number"][0] == "-") {
        $info = "no symbol";
    } elseif ($n1 != $n2) { //first check
        $info = "no, this is not a palindrome number!";
    } else { //second check
        if(is_palindrome_number($req["number"])) {
            $info = "nice! {$n1} is a palindrome number!";
        } else {
            if(strpos($req["number"], ".") === false && $n1 < 2147483646) {
                $info = "find another strange dongxi: " . FLAG2;
            } else {
                $info = "find a strange dongxi: " . FLAG;
            }
        }
    }
} else {
    $info = "no number input~";
}
?>
```
拿到第一个flag的条件是：
1. **\$req["number"] == intval(\$req["number"])**
2. **intval(\$req["number"]) == intval(strrev(\$req["number"]))**
3. **number 不为回文数**

拿第一个flag的两种方法：
1. **intaval整数溢出**
[http://php.net/manual/zh/function.intval.php](http://php.net/manual/zh/function.intval.php)这里写到
```最大的值取决于操作系统。 32 位系统最大带符号的 integer 范围是 -2147483648 到 2147483647。举例，在这样的系统上,intval('1000000000000') 会返回 2147483647。64 位系统上，最大带符号的 integer 值是 9223372036854775807。```
那么其对大于等于最大值得数，都只会返回最大值，即```intval(09223372036854775807)```=```intval(70857745863027332290)```=```9223372036854775807```
2. **浮点数溢出**
在学习c语言的时候，相信大家都知道计算机中的等于不是严格的等于，都是会忽略小数点后多少位的近似等于，类似在php中```1.0```=```1.000000000000001```，于是构造```1000000000000000.00000000000000010```

拿到第二个flag的条件是：
1. **不能出现第一个flag中的方法**

拿第二个flag的方法：
1. **利用trim和intval在处理空白字符上的差异性**
构造```%0c121```

参考：
[http://www.tuicool.com/articles/vY7zIb7](http://www.tuicool.com/articles/vY7zIb7)
