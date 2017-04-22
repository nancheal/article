#Mac安装w3af的坑
**官方文档里的mac环境下的安装方法**
[http://docs.w3af.org/en/latest/install.html#installation-in-mac-osx](http://docs.w3af.org/en/latest/install.html#installation-in-mac-osx)
这一步是必须的否则，ld将无法建立后续的链接
```shell
sudo xcode-select --install
```
```shell
git clone https://github.com/andresriancho/w3af.git
```
这一步执行完，将会打印出需要pip的依赖
```shell
./w3af_console
```
其中这一步，建议按下面这样写，来安装最新版，安装w3af提示的版本会出现错误
```shell
sudo pip install pyOpenSSL
```
ok，现在安装了最新版的pyOpenSSL后，我们需要修改一些东西[https://github.com/andresriancho/w3af/issues/15260](https://github.com/andresriancho/w3af/issues/15260)，安装issus里的提示，将包的版本号改为我们安装的就可以了，有其他的问题欢迎留言^_^