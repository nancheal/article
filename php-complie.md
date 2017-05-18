#MAC 环境下编译php
>除非生产需要、和闲的蛋疼，否则尽量别编译低版本的php

血淋淋的教训啊，低版本的php要求编译的环境真蛋疼，autoconf、bison...按照php官方写的版本要求，一看我去都是世纪初的老版本了，我都准备直接docker一个低版本的编译环境了。
所以我编译了php7.0.20的版本，一看比我brew phpmyadmin依赖的php7.0.14略高一点，即使是高版本在mac下也有一些问题，就是mac自带的bison的版本是2.3的，又低了
```shell
brew search bison
brew install bison
```
这样install的是3.0.4的，那么就涉及到切换的版本的问题，本能的去/usr/bin/里直接修改替换bison，发现怎么弄都没有权限，好像就是root都没有权限的，要进recover中disable掉csrutil，有点烦..随手一打
```shell
➜ bison
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/bison: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/bison: missing operand
Try '/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/bison --help' for more information.
```
原来这个bison指的是xcode里的啊，那好就在这里面替换，不用那么的麻烦
```shell
sudo mv bison bison2.3
```
再将brew的bison cp到这里
```shell
sudo cp /usr/local/Cellar/bison/3.0.4/bin/bison /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/
```
如果有需要可以将原bison rename回来
接下来在
```shell
./buildconf
./confguire --disable-all
make
```
都正常了，验证一下
```shell
./sapi/cli/php -v
```
其实对于ctag这类工具亦是如此