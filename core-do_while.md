#php内核笔记 - do...while(0)在内核级应用中的妙用
>昨天心血来潮，准备看看php的核心，看见里面一些涉及到宏的操作基本上都是用do...while语句，究其原因，细听我娓娓道来。

百度了一下，其实不只是在php中，在许多的开源项目，像linux中这样的写法还是有很多的。究其原因，很简单在涉及到宏引用的地方基本上都会出现这样的写法。
**do...while(0)的特点就是强制执行循环体一次**
像下面这样的语句
```c
#define dosomething() a++;b++;
if(a>0)
	dosomething();
```
在宏展开后变成
```c
#define dosomething() a++;b++;
if(a>0)
	a++;
b++;
```
宏展开后语句发生了变化，那么可能会说我把宏引用中的多语句用```{}```替换do...while(0)是否可行
```c
#define dosomething() {a++;b++;}
if(a>0)
dosomething();
```
那么dosomething后的```;```就会像下面这样影响，导致语法错误
```c
#define dosomething() {a++;b++}
if(a>0){
a++;
b++;
};
```
而，使用do...while(0)的话就会被解析成
```c
#define dosomething() do{a++;b++}while(0)
if(a>0)
do{
a++;
b++;
}while(0);
```
简而言之，**这样的结构可以保证宏的正确展开**
第二种情况，就是php内核种需要将一些操作以这种形式留为空