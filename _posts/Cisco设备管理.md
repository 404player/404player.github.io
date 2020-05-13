---  
layout: post
title: Cisco设备管理
lastUpdate: 2020-05-12
author: 404player
header-img: img/post-bg-cisco.jpg
catalog: true 	
tags: [cisco,网络,CCNA,路由管理]
---  
  
已经好久没有写博文了，疫情期间感觉整个人的精气神都有些衰弱，不过就算强拉着扯着，人总得有点向上的欲望对吧？这次的文章整理的内容也比较简单，是关于Cisco(思科)一些网络设备的管理操作。简而言之，其实就是教你怎么通过敲命令行的方式将一些公司级别的网络设备调好。别看只是敲命令行这种基础操作，这其中隐含的奥妙大着呢！有位网安的前辈跟我说过：广州地铁在设施也算花了大价钱，但由于在网络工程师上下的成本不足，导致很多原本能调出来的功能（大多是一些安全防护功能）没能成功设置，所以广州地铁网络的安全性其实是不足的。由此可见，对网络设备的调试是多么重要的。对于我这么一个初出茅庐的小菜鸟，敲命令行也是一件十分享受的事，下面我们开始吧！
  
<!--more-->  
  
- - -  

## 1.设备硬件架构  
  
学Linux的时候看过`鸟叔`的 `Linux`私房菜，里面开头花了大篇幅来介绍电脑的硬件架构，说明了解设备的硬件架构对设备调试其实是很重要的，下面我们来了解一下网络设备`路由器`和`交换机`的硬件架构。  
  
### 1.1 路由器  
  
<img src = "https://img-blog.csdnimg.cn/20200512223825707.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">

上图画的就是路由器的硬件架构，关于这个思维导图，有几点细节需要说明一下：  
  
- Flash中存储的IOS指 得是Cisco研发的互联网操作系统，并不是我们所熟悉的苹果操作系统，你可以简单地将之理解为是路由器、交换机等网络设备的操作系统  
  
- ROM是只读存储器，断电也能保留数据，只能读出，不能写入  
    
```
由于ROM这个奇妙的特性，ROM中常存储有一些用于安全恢复，系统启动的程序。常玩电脑的哥们都听过'BIOS'这个玩意吧？它保存着计算机最基本的输出输入程序，开机后自检程序还有系统自启动程序等一系列底层控制程序。这个程序就是存储在ROM上，才能保证计算机系统最基本的安全性。  
  
对于路由器的ROM来说，主要由三部分组成：  
  
A. Bootstrap程序：引导程序，用于引导加载操作系统  
B. Rommon程序：用于做密码恢复及系统升级  
C. POST程序：用于实现加电自检
```   
  
- NVRAM这个是路由器特有的一个东西，用于放置配置文件，同时含有配置寄存器（用于影响路由器的启动进程）。   
  
```
路由器配置其实是一件繁琐的工作，所以，在设计其硬件架构的时候工程师往往会给自己考虑好`"后路"`。试想，假如没有'NVRAM'，配置文件将存储在'FLASH'中，假如此时系统中毒了，需要格式化'FLASH',重装系统，则配置过程则要重来一遍。有了'NVRAM',配置信息被完好地存储在另外一个空间，设计妙哉！！
```  
  
### 1.2 交换机硬件架构  
  
交换机的硬件架构跟路由器差不多，唯一的一点差别就是交换机没有`NVRAM`这个存储结构，所以`配置文件`只能存储在`FLASH`中。  
  
### 1.3 设备启动流程  
   
1.3.1 路由器启动流程  
  
a. 加电自检(POST)  
b. 从ROM中加载并运行bootstrap启动微代码  
c. 查看`NVRAM`的配置寄存值  
d. 寻找IOS映像文件并加载(FLASH---RAM)  
e. 寻找配置文件并加载(NVRAM---RAM)  
f. 正常运行  
  
1.3.2 交换机启动流程  
  
a.加电自检  
b.加载并运行bootstrap  
c.寻找并加载IOS映像文件  
d.寻找并加载配置文件  
e.正常运行  
  
- - -   
  
## 2.设备基础互联  
   
### 2.1设备连接方式  
  
网络设备主要有两种连接方式：  
  
A. 近端管理（带外管理）  
B. 远程管理（带内管理）   
  
<img src="https://img-blog.csdnimg.cn/20200513104101378.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70" width="600px" height="400px">  
  
上图就是两种连接方式的示意图    
  
### 2.2 设备连接线缆  
  
目前网络设备连接线缆主要有两种：   
  
① 光纤  
- 单模光纤：传播速度快，距离远（大概有几十公里），黄色  
- 多模光纤：传播速度慢，距离近（几公里），橙色  
  
② 网线（以太网线）    

线序： 568B: 橙白 橙 绿白 蓝 蓝白 绿 棕白 棕  
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;568A: 绿白 绿 橙白 蓝 蓝白 橙 棕白 棕  
  
线序为1，2，3，6的为实际通信网芯，其他做备份   
注：A，B的区别其实就是1和3互换，2和6互换  
  
  
A----A B----B称为直通线，A----B称为交叉线  
一般来说，同级设备用交叉线，不同级设备用直通线  
Ps:因为现在大部分设备支持自动翻转，所以用直通线也能在同级设备中通信  
  
- - -   
    
## 3. IOS基础操作  
  
### 3.1 准备工作  
① 下载GNS3网络模拟器  
② 下载Wireshark抓包工具  
③ 下载SecureCRT命令行管理工具  
④ 将Wireshark、SecureCRT与GNS3关联起来  
⑤ 下载一个IOS系统，并将之加载到GNS3中
  
下载我们就不说了，我们先说说怎么将抓包工具、命令行管理工具跟模拟器关联起来：    
  
选择`工具栏`中的`编辑`----`首选项`,此时会弹出一个框：  
  
<img src="https://img-blog.csdnimg.cn/20200513111513444.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70" width="600px" height="600px">  
  
在一般中选择终端设置，终端命令选择`SecureCRT`,然后返回桌面，在`SecureCRT`处右键点击属性，复制`目标`中的内容  
  
<img src="https://img-blog.csdnimg.cn/2020051311202386.png">  
  
复制以后，将`终端设置`中`终端命令行`的引号里面的内容用复制的内容**覆盖**掉，点击`apply`,然后`OK`。自此，`SecureCRT`算是关联完成了。  
    
`Wireshark`的关联大同小异，差别在于设置选项是在`首选项`中的`Capture`区块设置，设置方法跟`SecureCRT`是一样的。  

然后我们再来讲讲`IOS`的导入，点击`编辑`中的`IOS和Hypervisors`，在镜像文件中选择文件导入，点击保存就可。  
  
另外，为了编辑舒适度的需要，用户很有必要将`SecureCRT`的透明度调成半透明，具体方法可自行百度，出于篇幅所限，这里不加赘述。  
  
### 3.2 玩转命令行  
  
导入`IOS`后，我们就可以开始模拟路由器的调试工作了。小编导入的是`C3600`的IOS，所以本文就以此为例子作说明。  
  
在`GNS3`的侧边栏拖两台路由器到主编辑页面，此时路由器是没有`插槽`的，也就意味着两台路由器之间是没有办法连接进行通信的，所以我们要给路由器配置`插槽`。  
  
右击路由器图标，点击`配置`，此时会弹出一个输出框。    
  
<img src="https://img-blog.csdnimg.cn/20200513114150491.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70" width="600px">  
  
点击`R1`,选择`插槽`设置，在`slot 0`处选择`NM-1FE-TX`,意思就是给路由器添加一个快速以太网模块接口，按照上述方法给另一个路由器设置以后，点击侧边栏一个雪糕状的图标可以给路由器进行连线。  
  
<img src="https://img-blog.csdnimg.cn/20200513114953416.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">  
  
这就是最简单的一个网络拓扑结构。  
  
要对路由器进行命令行调试，首先我们要先说说路由器的三种权限：  
  
A. 用户模式：等同于普通用户，对系统有基本的调试功能。  
B. 特权模式：等同于管理员，对系统执行基本的管理。  
C. 配置模式：等同于管理员，对系统执行所有操作。  
  
注：测试在特权，配置在配置。在用户模式中的所有命令都可以在特权模式中执行，在特权模式中所有命令却不能在配置模式中执行。  
  
好！！经过前面这么繁琐的操作，我们终于可以开始敲命令了！！（我码字也码的好辛苦）  
  
首先，我们先要启动路由器，右键点击路由器，选择`开始`就可以，然后右键选择`console`，此时`SecureCRT`命令行便调出来了。  
<img src="https://img-blog.csdnimg.cn/20200513120010596.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">  
  
下面我们分小任务来了解命令行操作  
  
#### 3.2.1 进入用户模式  

作为思科的一款高级模拟器，`GNS3`给我们提前做了很多设置，例如我们刚开始进入`console`命令行的时候，就进入了路由器的特权模式（#），而当我们使用`exit`命令想要退出到用户模式时，发现并不能成功，这是因为`GNS3`提前做了限制，所以我们必须删除掉它自带的一些权限命令才行。那该怎么做呢？  
  
首先要先进入配置模式：`configure terminal`  
进入配置模式以后发现前面名字已经变成了`R1(config)#`。随后我们输入`line console 0`进入调试接口，再运行`no privilege level 15`删除掉这条配置命令、最后我们连续用`exit`命令退出，就可以最终进入用户模式(>)。  
  
值得注意的一点是，我们打命令不一定要全部打上去，优秀的计算机工作者必须学会优雅得使用`tab`键进行命令补全。  
 
#### 3.2.2 用户模式常用命令  
  
既然我们已经能顺利地进入各种模式，现在我们要来了解用户模式下的常用命令。  
   
用户模式是路由器最低的权限，所以能用的命令功能仅限于查看。  
  
```
ping     //测试连通性  
traceroute  //链路追踪  
show arp //查看arp选项    
show clock  //查看系统时间
show version //查看系统版本信息，软件和硬件版本
```  
  
值得注意，对于一台还没有设置过的路由器来说，`ping`是暂时无法成功的。因为他还没有自己的`IP地址`，而且对外通信的端口还没有打开。  
  
同样，`show arp`对没有设置的路由器来说显示的应该也是一张空表。  
  
比较特别的是，`show clock`显示暂时还是路由器的出厂时间，现实中要对路由器调配，把路由器系统时间调试正确是必要的，这里暂时不讲（因为我也不会。。。）  
  
#### 3.2.3 特权模式常用命令  
  
从用户模式进入特权模式只需要一条命令，那就是`enable`   
  
进入特权模式以后，能查看的东西便比用户模式要多得多了，下面我们来看看。  
  
```  
show flash:   //查看硬盘大小  
show running-config //查看运行配置（内存里）  
show startup-config //查看内置配置（NVRAM）  
copy run start //保存配置
write  //保存配置   
show ip interface brief //查看接口三层地址（相当于windows的ipconfig）   
show interface f0/0 //查看接口具体地址（两层地址）
```  
  
特权模式中，最重要的`2`和`3`两条命令，简单来说，`show running-config`其实就是看写在内存里的命令，而`show startup-config`其实就是看写在`NVRAM`的命令。有基本计算机架构知识的同学都知道，写在内存的命令只要一断电就会丢失，所以要保证配置真正地安全地运行在设备上，必须使用`write`命令将内存的配置保存在`NVRAM`中。  
  
至于最后两条配置，其实用于我们在配置模式中配置好`IP`后来查看是否配置成功的，这个我们后面还会继续深入了解。  
  
#### 3.2.4 配置模式常用命令  
  
从特权模式进入配置模式也只需要一条命令：`configure terminal`  
  
顾名思义，在配置模式下的命令大多都跟路由器的配置有关：   
  
```  
hostname R1 //路由器改名   
no ip domain-lookup //关闭域名解析   
line console 0 //进入console口调试模式    

/******** 以下命令都在console调试下进行 *********/     
  
exec-timeout 0 0 //关闭发呆超时    
logging synchronous //日志输出同步    

/******** 以上命令都在console调试下进行 ********/
```      
   
`no ip domain-lookup`的意思就是当你用某个域名作为命令输入的时候，系统不会自动给你解析，这条命令在你打错命令时候可以为你节省不少时间。   
  
`console 0`是一个调试接口，进入这个接口以后，可以对路由器的 `IP`进行设置。    

发呆超时可以理解为一个锁定时间，超过这个时间以后系统就会进入一个锁定的状态。关闭发呆时间只是为了学习需要，在工程环境下，出于安全性考虑，需要设定合理的发呆超时。  

- - -    

## 4.IOS进阶操作      
  
下面我们长话短说，分任务对各种情况进行命令行的解说。    
  
### 4.1 配置路由器IP地址（在配置模式下）   
  
```  
line console 0   
int f0/0  
no shutdown    
ip address 12.1.1.1 255.255.255.255  
```  
  
说明： `int`是`interface`的缩写，表示要进入`f0/0`这个口，要看路由器对应哪个口，可以在`GNS3`处选择`编辑`的`show interface labels`。进入接口后要用`no shutdown`打开接口，并设置`IP地址`和`子网掩码`。子网掩码是必需要设置的，不然没法通信。       
  
此时，你执行`end`命令直接退回到特权模式，并执行`show ip int bri`(show ip interface btief的缩写)，可以查看到接口开启，并显示`IP`地址。  
  
<img src="https://img-blog.csdnimg.cn/20200513190252597.png">
   
### 4.2 配置远程虚拟终端    
  
相信不少玩网络的同学之前也玩过`telnet`远程通信，现在我们通过配置`R2`这一台远程服务器，使得`R1`能通过远程登录上去。  
  
第一步就需要把`R1`和`R2`的`IP`都设置好，使两台机子可以相互`ping`通。  
  
接下来让`R2`进入配置模式，输入以下命令：   
  
```  
line vty 0 15  //进入虚拟终端，0到15分别代表了16个远程登录接口  
no login   //设置无密码登录
```    
  设置成功后，回到`R1`命令行终端，输入`telnet 12.1.1.2`，(12.1.1.2是我设置的`R2`地址)，此时可以进入`R2`的虚拟终端，值得注意的是，由于`GNS3`的权限设置，虚拟终端命令行只能在用户模式下操作，不能进入特权模式。     
    

    
### 4.3 设置密码   
  
相信很多看完我前面文章的小伙伴都有一个疑惑，既然用户模式可以直接进入特权，特权又可以直接进入配置，那分权限管理又有什么意义呢？别急，在正常的工程环境下，进入特权模式和配置模式甚至在用户模式下工作都是需要密码登录的，那我们怎么设置密码呢？    
  
首先我们先来学习设置用户密码，第一步就是先进入路由器配置模式，执行下面命令： 
  
```  
line console 0  
password   123456  
login
```    
其中`login`的作用相当于确认，不确认的话密码无法生效。   
  
同理，要是我们要设置远程登陆虚拟终端的用户模式密码，则执行以下命令：   
  
```  
line vty 0 15   
password 123456  
login
```   
  
这样做其实还是有问题的，假如你在特权模式下执行`show run`(show running-config的缩写)命令，你会发现你所有设置的密码都会以明文形式显示，意味着无论你设置特权密码，配置密码设置得多么复杂，只要其他人进入特权模式，就能轻而易举地拿到你的密码。那怎么办呢？  
  
命令设计者当然帮我们想好了，只要将上面的`password`改成`serect`,那么你在`show run`下看到的，就是被加密过的密码，问题迎刃而解。    
  
让我们设想以下这么一个场景，学校网络设备管理员有好几个，有一个因为薪资问题对学校财务处怀恨在心，于是有一天远程登录上学校路由器进行了配置删除，给学校造成了巨大的财务损失。要追责的时候发现，你并不知道是哪个管理员登录的，这该怎么办呢？  
  
解决这种问题的方法是给每个管理员配置一个独一无二的账号密码，这样，后期追责的时候自然就能查出是谁进行的恶意操作，下面我们来看怎么给服务器进行账号密码的配置。  
  
同样我们也是要进入配置模式（后面不再说了，配东西就是要在配置模式:   
  
```  
username 404player serect 123456  
username geekboy serect cisco   
line console 0   
login local  //调用数据库   
line vty 0 15  
login local  
```   
  
上面命令成功给路由器配置了两个账号密码，并进入调试和远程登录接口成功调用了数据库里的账号密码，至此，路由器可以单独地通过账号密码近端或远程登录。  
  
最后一个知识点就是关于特权模式的密码配置了，其实很简单，在配置模式中执行命令：  
  
```  
enable serect 123456
```  
    
执行后，从用户模式进入特权模式的时候，就要输入密码了。   
  
在设置所有密码成功以后，一定要记得`end`到特权模式把配置给`write`到`NVRAM`里，否则一重启，所有配置都是白搭，要知道，现实工程中网络工程师的出场费可以很贵的。   
  
### 4.4 密码破解   
  
现阶段网络爱好者喜欢收集一些二手网络设备（新的太贵，买不起，随随便便几万几十万）。保管得好的话，这些设备在性能方面其实是完全没问题的，唯一容易出大问题的是，有时候连卖家都不知道密码是啥了，这个时候设备到手，破解密码这项技能是必不可缺的了。    
  
让我们回想一下路由器开启的流程，它是先加载IOS镜像再加载配置文件的，而密码文件就是存储在配置文件中。这个时候我们只要想办法跳过配置文件的加载过程，先直接进入系统中，删除掉`NVRAM`的密码文件，重启开机的时候不就不需要密码了吗？（当然你改了它也行，你喜欢喔）    
  
还记得我在介绍`NVRAM`的时候讲到过一个叫`配置寄存值`的东西吗？正常启动机器时，配置寄存值是`0x2102`,而只要将配置寄存值改为`0x2142`，路由器就会跳过加载配置文件的过程直接正常启动。  
  
这就有点像我们电脑的`安全模式`，当我们忘记电脑密码或者出现其他问题的时候，我们往往能通过进入`BIOS`直接进入`安全模式`进行设置。   
  
要进入跳过加载配置文件的`Rommon`模式，首先要重启路由器。  
  
在路由器没有重启完成的时候快速按下`ctrl+break`(没有`break`键可以尝试`ctrl+shift+pause`),此时进入了路由器的`Rommon`模式，接下来就是改动配置寄存值，然后重启：   
  
```  
confreg 0x2142   
reset
```   
  
重启后此时，你执行`end`命令直接退回到特权模式，并执行`show ip int bri`(show ip interface btief的缩写)，可以查看到接口开启，并显示`IP`地址。  
  
<img src="https://img-blog.csdnimg.cn/20200513190252597.png">此时，你执行`end`命令直接退回到特权模式，并执行`show ip int bri`(show ip interface btief的缩写)，可以查看到接口开启，并显示`IP`地址。  
  
<img src="https://img-blog.csdnimg.cn/20200513190252597.png">需要将配置文件加载到内存里，并进行密码管理：   
  
```
copy start run
conf t   
no username 404player  
no username geekboy  
line con 0  
no login local  
line vty 0 15
no login local    
end 
write   
config-register 0x2102   
end 
reload  
```   
  
执行完上述命令后，再次进入`IOS`时，发现密码已经没有了。   
  
特别要注意的一点是，写完密码管理的命令以后一定要`write`一下，否则配置无法保存到`NVRAM`,你所有的努力就白费了。   
  
最后要将配置寄存值改回正常状态`0x2102`，随后重启。   
  
### 4.5 备份配置文件 
  
备份这个实验`GNS3`无法完成，需要借助另外一个同样是思科的模拟器`Cisco Packet Tracer`才能完成。  
  
要完成备份的工作，首先需要一台配置好的路由器和一台用于备份的`TFTP服务器`  
  
<img src="https://img-blog.csdnimg.cn/20200513211209951.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">   
  
配置好路由器和服务器的`IP`和接口，确保两者可以ping通。  
  
此时只需要执行一条命令：`copy start tftp:`或者`copy run tftp:`，就会出现提示，让你输入`tftp服务器`的 `IP`地址，输入确认后会让你输入备份配置文件的文件名，当然，这个文件名系统是有默认的，直接回车选择默认文件名就可以。值得注意的是，这个文件名是需要记住的，还原配置的时候是需要输入备份的配置文件名的。    
  
所有操作执行完以后，路由器的配置就备份到`tftp服务器`里去了。    
  
<img src="https://img-blog.csdnimg.cn/20200513211914305.png">   
  
当你配置丢失的时候，还原也很简单，只要反过来执行`copy tftp: start`就可以了     
  
### 4.6 管理IOS文件     
  
#### 4.6.1 备份IOS操作系统
  
现实中我们也常常会出现要重装系统的情况，所以，对于网络工程师来说，备份和恢复IOS系统也是必备的技能之一。   
  
这个时候，我们刚刚学到的关于硬件架构的知识就起作用了。回忆一下，IOS系统镜像是存储在路由器的`FLASH`中的，此时我们可以在用户模式中`show falsh:`看一看，会出现下面的内容：  

<img src="https://img-blog.csdnimg.cn/20200513213115416.png">        
  
我们看到那个`.bin`结尾的文件就是路由器的系统镜像文件，通过上面学习我们已经很熟悉，其实备份和还原本质上都是`复制黏贴`，所以我们可以毫不犹豫地执行命令`copy flash: tftp:`。这时候会提示你输入备份文件名，你要将上面`show flash:`的那个`.bin`结尾的文件名复制下来黏贴到下面，回车！  
  
地址同样是填`TFTP服务器`的地址，确认备份后，便可将整个IOS文件复制到服务器上。      
  
#### 4.6.2 恢复IOS操作系统    
   
首先我们先讲讲正常恢复的情况，为了复现这种情况，首先我们可以把本机的IOS先删掉。    
  
执行`delete flash: c2900-universalk9-mz.SPA.151-4.M4.bin`(这个文件名通过`show flash`看就可以)。  
  
执行后不要重启，重启后便不能正常恢复了。此时你再`show flash:`，你会发现：  
  
<img src="https://img-blog.csdnimg.cn/20200513214520802.png">   
  
`IOS`文件已经不见了。   
  
此时只要反过来执行`copy tftp: flash:`,输入服务器地址和IOS文件名，就能实现正常恢复。    
  
在正常工程环境下，意外的发生是无法避免的。试想一下，假设你在恢复系统的过程中路由器突然断电了，这意味着你已经没有系统可以进入了，那怎么通过命令来恢复系统呢？   
  
我们来复现一下这种环境，只要在`delete`命令以后直接`reload`重启，就可以看到这种情况。      

<img src="https://img-blog.csdnimg.cn/20200513215341193.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70">     
  
进去以后可以通过命令`dir flash:`查看此时`flash`的存储情况。    
    
因为没有操作系统了，所有`tab`补全等一系列高级操作都是不存在的了，在这种苛刻的情况下想要恢复系统的确很麻烦，大家仔细看下面一系列命令。   
  
```  
rommon 1 >IP_ADDRESS=12.1.1.1         
rommon 2 >IP_SUBNET_MASK=255.255.255.0    
rommon 3 >DEFAULT_GATEWAY=12.1.1.2   
rommon 4 >TFTP_SERVER=12.1.1.2      
rommon 5 >TFTP_FILE=c2900-universalk9-mz.SPA.151-4.M4.bin    
rommon 6 > set  
rommon 7 > tftpdnld 
rommon 8> reset   
```   
  
记住这个时候的命令行是区分大小写的，其中`IP_ADDRESS`填的是路由器的`IP`,`DEFAULT_GATEWAY`和`TFTP_SERVER`填都是服务器的地址。`set`命令是用于查看你前面打的命令究竟有没有写进路由器中。  
  
执行完所有命令以后，`reset`重启，路由器IOS恢复。    
  
  
## 5.结语    
  
关于路由器设备管理这一块，目前就整理那么多，再深入我也不太会了。搞网络安全，基础还是在网络，前段时间学习这一块知识，可以说是受益匪浅，对网络这一块有了更深入的认识。本文是结合实操一起看的，干看其实很难完全理解，考虑到涉及到数量不少的工具和资源，有兴趣的同学可以加我微信，我整理好了可以分享一下。码文不易，文笔不好，请多见谅。期待跟各位技术宅的进一步交流。
  
<img src="https://img-blog.csdnimg.cn/2020051322143719.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70" align="middle">    

