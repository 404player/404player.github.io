---  
layout: post
title: PowerShell美化之折腾日记
lastUpdate: 2020-11-07
author: 404player
header-img: img/post-bg-powershell.jpg
catalog: true 	
tags: [PowerShell]
---    

作为`Windows`用户，我无法否认微软的美学境界上不了档次。  
  
相较于`Linux`高端大气上档次的终端，尽管`Windows`自带的原生`PowerShell`非常强大，我还是无法在那惨不忍睹的页面中操作我的命令行。  
    
一名计算机玩家，要有自己动手解决问题的能力。为了让自己更舒服地使用原生`PowerShell`,我花了一小时左右的时间对它进行了简单的美化，并把过程记录了下来。    
  
- - - 
  
## 一. 确认需求  
  
要进行一个项目之前，我们首先要确认一下需求。美化`Powershell`也不例外。  
  
我们首先对需要美化的几个点与解决方法进行列举：  
  
- demand: `Powershell`颜色与透明度问题
    - solution: 在`PowerShell`属性中进行设置  
- demand: 需要在`Powershell`中使用`git`,支持仓库显示和`tab`补全
    - solution: 安装`posh-git`  
- demand: 要像在`Linux`下一样，支持对不同文件类型的高亮
    - solution: 安装`DirColors`  
- demand: 需要像`Linux`神器`oh-my-zsh`类似的界面
    - solution: 安装`oh-my-posh`  
- demand: 选择一种适合命令行操作的字体  
    - solution: 安装更纱黑体  
- demand: 支持语法高亮，支持命令行智能提示
    - solution: 安装`PSReadLine`  
- demand: 达到`Linux`装机晒图的效果
    - solution: 安装`Screenfetch`
        
          
以上，我们对`PowerShell`美化的要求已经列举完毕了，可以开搞了，冲！！！    
  
在此之前，我们先来欣赏一下“原装”的`Powershell`丑陋的容颜    
  
[![Boiebt.png](https://s1.ax1x.com/2020/11/08/Boiebt.png)](https://imgchr.com/i/Boiebt)  
  

- - -     
      
## 二. 更改执行策略  
  
为了防止恶意脚本的执行，`Powershell`默认的执行策略是`Restricted`,这种执行策略可以执行单个的命令，但是不能执行脚本。也就是运行脚本的时候`Powershell`会报错。  
  
你可以通过`Get-ExecutionPolicy`来查看你的执行策略。    
  
折腾的方式千千万万，尽管我到最后也没有运行脚本，但刚开始我还是有必要将执行策略改动一下，提高一下我操作的权限。  
  
```
Set-ExecutionPolicy RemoteSigned
```   
    
`RemoteSigned`策略的意思是：当执行从网络上下载的脚本时，需要脚本具有数字签名，否则不会运行这个脚本。如果是在本地创建的脚本则可以直接执行，不要求脚本具有数字签名。    
  
- - -
  
## 三. 更改颜色和透明度    
  
更改颜色和透明度是最迫切最迫切的需求，原装的`Powershell`*屎蓝屎蓝*的页面实在是**太丑了**！！！  
  
这个需求很简单，只需要在顶边栏`右键->属性`，就可以弹出这么一个框框  
  
<img src="https://s1.ax1x.com/2020/11/08/Bokc4K.png" height="400px">  
  
个人喜好是将**字号**改成`16`，将**终端背景**调成`黑色`，透明度设为`85%`,最后在**布局**那里将**窗口大小**改成高度为`32`，宽度为`120`。    
  
- - -   
  
## 四. 安装`posh-git`  
  
作为一个`git`使用者，当然无法忍受终端不支持`git`的阴间配置。  
  
安装`posh-git`的方法很简单，执行下面命令  
  
``` 
Install-Module posh-git -Scope CurrentUser
```  
    
当提示你安装`NuGet`的时候，直接选择`yes`。  
  
安装过程中还会出现这样的警告  
  
```
不受信任的存储库
你正在从不受信任的存储库安装模块。如果你信任该存储库，请通过运行 Set-PSRepository
cmdlet 更改其 InstallationPolicy 值。是否确实要从“PSGallery”安装模块?
[Y] 是(Y)  [A] 全是(A)  [N] 否(N)  [L] 全否(L)  [S] 暂停(S)  [?] 帮助
```  
  
同样选择`Y`,静待安装完成。  
  
大功告成以后，还需要导入`posh-git`  
  
```
Import-Module posh-git
```  

自此，你的`Powershell`就可以支持`git`命令行的操作了。  

- - -   
  
## 五. 安装`DirColors`  
  
当你在`Linux`中执行`ls`命令的时候，会列出所在目录下所有的显式文件，不同文件格式（目录/文件)会进行不同颜色的高亮显示。我们的需求就是在`Windows`上也做到这一点。  
  
安装`DirColors`的方法不难  
  
```
Install-Module DirColors -Scope CurrentUser
Import-Module DirColors
```  
  
典型的安装加导入，当然这次的导入只能维持单次窗口的配置，也就是说，以后再次打开`Powershell`还能有这个效果，必须将其导入`profile`文件中，这个后面一起导入， 不要着急。  
  
先来看一眼效果  
  
<img src="https://s1.ax1x.com/2020/11/08/BoEX0s.png" height="400px">  
  
- - -   
  
## 六. 安装`oh-my-posh`  
  
`oh-my-posh`的`github`项目地址是这个：`https://github.com/JanDeDobbeleer/oh-my-posh`,你可以在项目下的`README`文档里进行你自己的样式选择。  
  
获取方法还是同样的安装导入，`oh-my-posh`还多了一个样式选择  
  
```
Install-Module oh-my-posh -Scope CurrentUser
Import-Module oh-my-posh
Set-Theme Agnoster
```  
  
`Agnoster`只是我选择的样式，大家尽可以自由选择。  
  
同样，这种导入只适用于当前窗口，所以后面还要写入`profile`文件。  
  
- - -   
  
## 六. 安装更纱黑体  
  
字体的美化在终端显得极其重要，没有人会喜欢在炫酷的命令行操作中还喜欢看到`宋体楷书`这种字体吧。  
    
更纱黑体的项目地址：`https://github.com/be5invis/Sarasa-Gothic/releases`  
  
我下载的是`release`中的`ttc`，下载完成后全部安装，就可以在`Powershell`的属性中看到并选择了。  
  
效果看上图。  
  
- - -   
  
##  七. 安装`PSReadLine`  
  
要打造一个`Windows`版本的`Linux`终端，怎么可能不支持语法高亮呢？  
  
看到这里的小伙伴已经很熟悉业务了，安装导入一步走！！！    
  
```
Install-Module PSReadLine -Scope CurrentUser
Import-Module PSReadLine
```  
  
但是要进行智能提示，单单这一步还不够。在进行配置之前，先让你们羡慕嫉妒恨一下我的智能提示。  
  
<img src="https://s1.ax1x.com/2020/11/08/BoZQP0.png">  
  
看到后面那一串灰灰的字体没？那就是智能提示！酷不酷，爽不爽？配置走起！  
  
如果你老以前就安装了`PSReadLine`，你先得升级一下  
  
```
Install-Module PSReadLine -Force -SkipPublisherCheck -AllowPrerelease
```  
  
智能提示的原理就是从你输入命令的历史记录中找到匹配的隐式显示，所以还需要安装一个`PowerShellGet`  
  
```
Install-Module -Name PowerShellGet -Force
```  
    
接下来就需要把`PSReadLine`的`PredictionSourceSource`设置为`History`  
  
```
Set-PSReadLineOption -PredictionSource History 
```  
  
现在就大功告成了。前文说过，这些设置只在单次的`Powershell`窗口上起作用，要一劳永逸地配置，还需要配置`profile`文件。  
  
- - -   
  
## 八. 修改配置文件  
  
`PowerShell`为我们提供了`profile`,每次打开`Powershell`，都会先执行`profile`文件中的命令。  
  
为了找到`profile`文件，可以在`Powershell`中输入`$profile`，会显示`profile`文件的绝对路径。  
  
```
$profile

C:\Users\admin\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
```  
  
你可以根据路径找到它，如果没有这个文件，就在`C:\Users\admin\Documents\WindowsPowerShell`中新建一个`Microsoft.PowerShell_profile.ps1`，并将下面命令写入  
  
```
Import-Module DirColors
Import-Module oh-my-posh
# oh-my-posh会导入posh-git，所以不需要再次导入
Set-Theme 你自己喜欢的主题
Import-Module PSReadLine
Set-PSReadLineOption -PredictionSource History
```  
  
完成这一步，我们的美化之路就走到最后一步了。  
  
- - -   
  
## 九. 安装`Screenfetch`  
  
对于`Linux`使用者来说，装机以后必定要炫一波配置，`Screenfetch`就是必备神器之一了。  
  
配置`screenfetch`很简单，甚至不需要导入，只需要安装即可  
  
```
Install-Module windows-screenfetch -Scope CurrentUser
```   
    
看看效果，好家伙，十分炫酷  
  
<img src="https://s1.ax1x.com/2020/11/08/BonGM8.png">  
  

- - -  
  
## 十. 总结  
  
折腾，是每个计算机爱好者的美德，拥有一台自己用得顺手的电脑是一种资本，希望越来越多的同类人加入折腾的世界~  
  
<img src="https://s1.ax1x.com/2020/11/08/BonfiR.jpg">



  

  

  

  

  

  

