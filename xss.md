#XSS相关

####基础知识
1、在```<title>```这类普通标签中，直接加```<script>```执行js代码
2、在```<input value="xss">```这类标签中，可选择```">```闭合标签，或者```"```闭合引号再使用onclick这类事件，去触发js代码
3、再```<textarea>```这类优先级大于```<script>```的标签中，需要先闭合```<textarea>```标签

####思想
```img```，```a```，```iframe``` 都可以承载xss代码，核心思想就是让页面执行我的JS代码

####编码与解码
在前端的代码中我们的输入可能有三种编码的形式```HTML```，```JS```，```URL```，几种编码顺序的举例
```html
document.getElementById('a').innerHTML= "."'".htmlspecialchars($value)."'".";
```
浏览器对上述代码的解码顺序是先```JS```，再```HTML```，所以我们可以对```$value```先进行JS编码来绕过一些限制
```html
$html .= '<a onclick=" ' .htmlencode($_GET['name']).'">click this url</a>';
```
浏览器对上述代码的解码顺序是先```HTML```，再```JS```，所以我们可以对```$GET['name']```先进行HTML编码来绕过一些限制
####技巧
1、查看，是否前端过滤XSS代码，可以抓包绕过
2、XSS黑名单如果过滤了onxxx代码，不一定是过滤了onxxx这种所有代码，尽量多尝试黑名单绕过如：```ondrag```
3、zeroclipboard.swf是有XSS漏洞的复制内容插件，看见注意，POC:[http://www.freebuf.com/sectool/108568.html](http://www.freebuf.com/sectool/108568.html)
4、XSS换行绕过检测，直接再抓取的数据包中输入回车换行