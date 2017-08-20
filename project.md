#injectMan项目小节
####BaseHTTPRequestHandler
这是一个可以用来解析HTTP格式的库，sqlmap中也是使用了这个库来实现```-r```功能，因此我在项目中也是使用这个库来实现```-f <file>```功能，简单介绍一下用法
```python
from BaseHTTPServer import BaseHTTPRequestHandler
from StringIO import StringIO
request = HTTPRequst(text)
command = request.command
headers = request.headers
path = request.path
body = request.rfile.read()
class HTTPRequst(BaseHTTPRequestHandler):
    """
    HTTPRequest Prase class
    """
    def __init__(self,request_text):
        self.rfile = StringIO(request_text)
        self.raw_requestline = self.rfile.readline()
        self.error_code = self.error_message = None
        self.parse_request()
    def send_error(self,code,message):
        self.error_code = code
        self.error_message = message
```
command - HTTP请求方法
headers - HTTP请求头(包括cookies，HOST...)
path - HTTP请求路径
body - POST的请求参数
####requests
这个模块在发送post请求时遇到了一个问题，一般post代码如下
```python
import requests
req = requests.post(url,data=body,headers=headers)
```
假设在```headers```中设置了```www-form-encodeed```，那么在```data```中的数据按道理来说应该时被```urlencode```的，但是在项目未被```urlencode```，究其原因是我当时在在```data```中传入的是字符串，所以```requests```未对其进行```urlencode```，所以传入的应该是字典形式，才会安装```headers```里的声明进行加密
####代码重构之美
在这个项目中我深深体会到了代码重构的重要性，或者说是软件工程的重要性，像sqlmap这类的项目之所以难看懂，是因为自己的代码造诣还未达到那个程度
1、首先是```class```的问题，按照功能共性划分的```class```暴露给外部的不应该只是一个```main```或者```run```函数，应该是每一个```class```里的函数我都可以独立的拿出来使用，这样就要求在```class```的```__init__```函数中尽量少传入影响整个```class```的参数
2、自己实现了一个match多个值的findPos函数，效果还是不错的，如下：
```python
def findPos(self,str,sub):
    """
    FindPos function which find all sub's Position in str
    """
    pos = 0
    pos_list = []
    while True:
        pos = str.find(sub,pos)
        if pos<0:
            break
        pos_list.append(pos)
        pos += 1
    return pos_list
```
3、有的时候巧用格式化输出的变量，可以省去一些不必要的麻烦
4、先加再减的智慧
####python 爆出的错
1、```not -1```不等于 0
2、```.format() raise keyError```错误，应该在字典外再加上一层括号，像下面这样
```python
"{{'xxx':'{0}'}}".format()
```