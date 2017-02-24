#php反序列化问题
>依稀记得去年java反序列化来的时候，真是风雨满楼啊，各种组件中招，今天在群里听到大菠萝批评‘java有反序列化问题，同理php等语言也会有这些问题’这种惯性思维不可取，并表示自己跟进去过java和php的序列化机制，发现两者并不相同，不得不感慨他这种专研的精神，赞一个，我就分析下php代码层的问题

#unserialize
这一方法在许多的编程语言中都有存在，其较大的方便了数组和对象的存储，但是利用不当可能会带来意想不到的危害 --[http://drops.wooyun.org/tips/3909?utm_source=tuicool&utm_medium=referral](http://drops.wooyun.org/tips/3909?utm_source=tuicool&utm_medium=referral)
* **0x00 最简单的利用就是利用它序列化的特性，来单纯的伪造数据**
* **0x01 反序列化的数据导致类里的魔法方法的执行**
	这种情况的有几种前提条件：
    1. 存在unserialize
    2. 类里存在以下的魔法方法
    ```php
    __construct(),__destruct()
    __call(),__callStatic()
    __get(),__set()
    __sleep(),__wakeup()
    __tostring(),__invoke()
    __set_state(),__clone(),__debugInfo()
    ```
    3. 魔法方法里写有危险的函数
    ```php
    #commad exe:
    exec();passthru();popen();system();eval()
    #file access
    file_put_contents();file_get_contents();unlink()
    ```
    **漏洞代码 index2.php**
    ```php
    <?php 
	class test{
		public $par="phpinfo()";
		function __destruct(){
			eval($this->par);
		}
	}
	unserialize($_GET['a']);
	?>
    ```
    **poc**
    ```url
    http://127.0.0.1/test/unserialize/index2.php?a=O:4:%22test%22:1:{s:3:%22par%22;s:7:%22echo%201;%22;}
    ```
    这样就会执行echo 1这个操作
* **0x02 session里的反序列化问题**
	1. php内置了多种处理器来序列化和反序列化存取seesion数据
处理器                         |对应的存储格式
------------------------------|-------------
php                           |键名 ＋ 竖线 ＋ 经过 serialize() 函数反序列处理的值
php_binary                    |键名的长度对应的 ASCII 字符 ＋ 键名 ＋ 经过 serialize() 函数反序列处理的值
php_serialize <br>(php>=5.5.4)|经过 serialize() 函数反序列处理的数组
这里的问题在于如果php.ini里的session.serialize_handler如果和脚本里的ini_set('session.serialize_handler', '')如果不同的话就会出现安全问题，如以php_serialize存入带有竖线的字符串，再以php反序列化的时候，就会解析出一个变量出来（ps：在php 5.6.13之前变量解析失败是会继续解析下一个变量的，而在这之后出现解析的错误，就会销毁sesseion）
	2. 这里可能会有疑惑，该怎么样去控制session呢，这里配置不当会导致session被控制，session.upload_progress.enabled被打开，且session.upload_progress.cleanup被关闭时，php会记录上传文件的进度，并且记录在session中! --[https://bugs.php.net/bug.php?id=71101](https://bugs.php.net/bug.php?id=71101)
	**一道ctf的题**
    **index3.php**
    ```php
    <?php
 
	ini_set('session.serialize_handler', 'php');
 
	require("./class.php");
 
	session_start();
 
	$obj = new foo1();
 
	$obj->varr = "phpinfo.php";
 
	?>
    ```
    **class.php**
    ```php
    <?php
	class foo1{
        public $varr;
        function __construct(){
                $this->varr = "index.php";
        }
        function __destruct(){
                if(file_exists($this->varr)){
                        echo "<br>文件".$this->varr."存在<br>";
                }
                echo "<br>这是foo1的析构函数<br>";
        }
	}

	class foo2{
        public $varr;
        public $obj;
        function __construct(){
                $this->varr = '1234567890';
                $this->obj = null;
        }
        function __toString(){
                $this->obj->execute();
                return $this->varr;
        }
        function __desctuct(){
                echo "<br>这是foo2的析构函数<br>";
        }
	}

	class foo3{
        public $varr;
        function execute(){
                eval($this->varr);
        }
        function __desctuct(){
                echo "<br>这是foo3的析构函数<br>";
        }
	}
	?>
    ```
    **利用post.html**
    ```html
    <form action="http://127.0.0.1/test/unserialize/index3.php?" method="POST" enctype="multipart/form-data">
	<input type="hidden" name="PHP_SESSION_UPLOAD_PROGRESS" value="123" />
	<input type="file" name="file" />
	<input type="submit" />
	</form>
    ```
    因为我没有用5.6以前的版本，所以在用brupsuit抓包后，并没有达到利用的效果，但是在session文件中确实可以发现畸形的serialize字符串进入了session中