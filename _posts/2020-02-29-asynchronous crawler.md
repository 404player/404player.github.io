---
layout: post
title: 高性能异步爬虫总结
tags: [crawler,asynchronization,data analysis]
lastUpdate: 2020-02-29
author: 404player
header-img: img/post-bg-crawler.jpg  
catalog: true 	
---        

研究爬虫也有一段时间了，因为曾经见识过大佬给我写的多线程爬取B站整套教程的程序，一直对高性能的爬虫有浓厚的兴趣。在这段时间的学习中，见识到了一种非常牛逼的操作————单线程+多任务异步协程爬取数据，其高效的性能及对CPU的低占用让人眼前一亮。下面我们来一步步了解一下这种高级操作。    
<!--more-->
  
- - - 
## 1.单线程同步爬虫    
剖析爬虫的本质，其实就是模拟client发起请求去批量获取Server的响应数据。如果我们同时面对多个url爬取目标，还是采用一个线程并且串行爬取的形式的话，爬取效率将会非常低。因为我们总是等待一个目标爬取成功以后才去爬取下一个目标。注意：前面说的并不代表着单线程执行就等同于低效，如果单纯是计算的任务，单线程对CPU的利用率并不低。那为什么单线程爬取多任务的效率会如此低呢？主要是因为爬虫任务中存在请求等阻塞程序。那么我们怎么做才能提高爬虫的性能呢？    
  
    
## 2.多种形式爬取分析  
  
a. 针对多个url爬取目标，我们先采取同步串行的方式爬取看一下效果  
  
```python
from time import sleep
import time
def request(url):
    print('正在请求：',url)
    sleep(2)
    print('下载成功',url)
start = time.time()
urls = ['www.baidu.com','www.sogou.com','www.goubanjia.com']
for url in urls:
    request(url)
print(time.time()-start)
```  
上面的程序用`sleep`来代替请求操作（两者皆是阻塞程序），同步爬取会在提交一个任务后在原地等待任务结束，等拿到任务的结果后才会执行下一个任务，运行一下，我们看看结果：  
  
```
正在请求： www.baidu.com
下载成功 www.baidu.com
正在请求： www.sogou.com
下载成功 www.sogou.com
正在请求： www.goubanjia.com
下载成功 www.goubanjia.com
6.001564025878906
```  
由上面的执行结果我们可以看到，一共三个请求，每个请求两秒，串行操作执行下来一共耗时6秒  
  
b. 对于这种多个url爬取目标的任务，我们还可以考虑使用“线程池”或者“进程池”。这种方法和普通的多线程（或多进程）一样让每个连接都有单独的线程（或进程），这样任何一个连接的阻塞都不会影响其他的任务，同时池在一定程度上维持了合理数量的线程（或进程），缓解了频繁调用IO口带来的资源占用。我们尝试用这种方法去实现一下上面的例子：  
  
```python
# Author:Geekboy
from time import sleep
from multiprocessing.dummy import Pool
import time
def request(url):
    print('正在请求：',url)
    sleep(2)
    print('下载成功',url)
start = time.time()
urls = ['www.baidu.com','www.sogou.com','www.goubanjia.com']
# for url in urls:  
#     request(url)  
pool = Pool(3)
pool.map(request,urls)
print(time.time()-start)
```  
  
这个程序调用了`multiprocessing`的`pool`方法开启了三个线程，三个线程通知执行任务，每个request任务两秒，我们可以看到执行结果，时长由6秒减少到了2秒：  
  
```
正在请求： www.baidu.com
正在请求： www.sogou.com正在请求： www.goubanjia.com

下载成功 www.baidu.com
下载成功下载成功 www.sogou.com www.goubanjia.com
2.042771816253662
```  
那是不是意味着我们就解决了IO口阻塞这个问题呢？答案当然是否定的。首先我们要清楚的一点的是，计算机的资源是有限的，而开启线程是会占用资源的。线程池只是有效地降低了创建和销毁线程的频率，将线程维持在一个合理的数量上。这样做的确很好地减少了系统的开销，但是爬取任务很大，要同时对很多个目标站点发起请求时，线程池终究会有上限呢。当请求数大大超过线程池的上限时，“池”构成的系统对外界的响应其实与没有创建池区别不大。  
  
### 3.多任务异步协程  
  
我们先来看一个小学时候碰到过的数学问题，这个问题对于现在的我们来说很简单，却有效的解释了多任务的异步协程的原理：
    
> 假设在一个周末的早上，小肆同学起床了，起床后他有几件事情要做，分别是：刷牙洗脸3分钟，做早饭5分钟，烧水10分钟，用洗衣机洗衣服15分钟，请问他要完成这些任务最少需要多少分钟？  
  
这个问题对于现在的我们来说没有任何难度，只需要把衣服放进洗衣机，马上跑去烧水，刷牙洗脸，做早饭，这些任务就能在15分钟完成，那这跟异步爬虫有什么关系呢？  
  
不要着急，首先我们要注意的一点的是：**洗衣机洗衣服，烧水这些操作就相当于异步爬虫当中的阻塞操作，异步爬虫的原理就是当任务遇到阻塞时，将阻塞挂起，然后去执行下一个任务，当此任务阻塞结束以后，再次往回走执行此任务的其他操作**。我们很容易理解到，多任务异步爬虫本质上是一种基于阻塞程序的对任务组的一种无限乱序循环，只有当所有任务执行完毕，循环才会停止。下面我们来了解一下多任务异步爬虫的相关概念。  
  
```
- event_loop:事件循环，相当于一个无限循环，可以把一些特殊函数注册到这个事件循环上。

- coroutine：中文翻译叫协程，在Python中常指代为协程对象类型，可以将协程对象注册到事件循环中，
 它会被事件循环调用。我们可以使用async关键字来定义一个方法，这个方法在调用时不会立即被执行，而
 是返回一个协程对象

- task:任务，它是对协程对象的进一步封装，包含了协程对象的各个状态

- future：代表将来执行或还没有执行的任务，实际上和task没有本质区别
```    
  
首先我们要定义一个协程，需要导入`asyncio`模块，这个模块可以帮我们检测IO，实现程序级别的切换（换言之实现异步IO）    
  
随后我们要先定义一个事件循环，这个事件循环由上文得知就是多任务异步协程中所说的无限循环，需要以一个特殊函数为参数注册到事件循环中，而协程对象恰好就是一个这样的特殊函数  
  
值得注意的一点是，特殊函数的定义要以`asyncio`模块的`async`开头（Python 3.4才出现）     
  
定义好函数后，task其实就是对协程对象的进一步封装，同时task也包含了协程的状态，后面我们讲到绑定回调的时候会提及，而因为task是协程的封装，本身也就是一个特殊的函数，所以往往我们是用task去注册到事件循环中的  
  
现在对这些概念还不甚理解没有关系，下面我们通过一个程序来对上面所说的东西进行一个实例化的说明  

```python
import asyncio #导入asyncio模块    

#使用async来定义特殊函数，此特殊函数便是一个协程   
async def request(url):
    print('正在请求：',url)
    print('下载成功',url)  
      
c = request('www.baidu.com') #实例化一个协程   
#注意实例化协程的时候函数并不会执行  
  
task = asyncio.ensure_future(c) #实例化一个任务对象task进一步封装协程c    
  
loop = asyncio.get_event_loop() #创建一个事件循环对象    
  
loop.run_until_complete(task) #将任务对象注册到事件循环对象中    
#注意：实例化事件循环对象的时候，同时也会启动事件循环对象，换言之，就是会执行特殊函数  
```  
  
我们来看看执行结果：  
```
正在请求： www.baidu.com
下载成功 www.baidu.com
```  
  
上面所写的程序中，协程对象只有一个，并不能很好地反映多任务异步协程的NB之处，下面我们还用上面的例子，来实现一个多任务的异步协程，看看究竟有没有想象中的高效：  
  
```python
import asyncio
from time import sleep
import time
urls = ['www.baidu.com','www.sogou.com','www.goubanjia.com']
start = time.time()
async def request(url):  
    print('正在请求：',url)  
    #在多任务异步协程实现中，不可以出现不支持异步的相关代码  
    #sleep(2)  #time模块不支持异步  
    await asyncio.sleep(2)
    print('下载成功',url)
 #创建事件循环对象  
loop = asyncio.get_event_loop()
#任务列表：放置多个任务对象
tasks = []
for url in urls:
    c = request(url)
    task = asyncio.ensure_future(c)
    tasks.append(task)

loop.run_until_complete(asyncio.await(tasks))
print(time.time()-start)
```  
解释一下这个程序跟上一个程序细节上的差异  
  
首先值得注意的第一点：在定义协程函数的时候，函数中是不能出现不支持异步的相关代码，比如`sleep`便是不支持异步的，那怎么办呢？通过查询，可以发现`asyncio`模块提供与`sleep`函数作用等同的一个`asyncio.sleep`函数，我们替换一下便可。  
  
大家还会发现协程函数中出现了一个`await`的关键字，这个关键字简单说来就是挂起功能，意思就是将阻塞操作挂起，那怎么辨别阻塞操作呢？判断标准很简单，只需要将需要等的，不能马上返回结果的步骤视为阻塞便可。**记住，在定义协程函数的时候，所有阻塞步骤前必须加上`await`**。  
  
最后要注意的一点，这个程序是在循环内部实例化协程对象与任务对象，并将任务对象存储到`tasks`列表中，所以在将任务列表注册到事件循环中的时候，要注意加上`asyncio.await`，表示这个任务组在遇到阻塞后会挂起执行另一个任务，不加就会报错  
  
我们来看看程序的执行结果吧！！！  
  
```
正在请求： www.baidu.com
正在请求： www.sogou.com
正在请求： www.goubanjia.com
下载成功 www.baidu.com
下载成功 www.sogou.com
下载成功 www.goubanjia.com
2.00195574760437
```  
可以看到，多任务异步协程实现了和线程池差不多的时耗，而当任务越来越多，`单线程+多任务异步协程`的优势就会越来越明显，因为在单线程下，开启成百上千个异步协程并不会对资源造成太大的消耗，所有任务都是基于异步的，只是在阻塞的时候挂起去执行下一个任务，等阻塞结束，便回来继续执行，如此循环往复，直到所有任务结束。  
  
## 4. 给任务对象绑定回调  
什么是回调呢？  
  
回调其实是任务对象`task`的一个方法，该方法要传入一个参数，即为事先定义的回调函数。**回调的作用是让程序在启动了事件循环后，在对应任务对象执行完之后再去执行回调函数**。  
  
我们先写一个只有一个任务对象的程序，去说明如何给任务对象绑定回调。  
  
```python
# Author:Geekboy
import asyncio
async def request(url):
    print('正在请求：',url)
    print('下载成功',url)
    return url
#回调函数必须有一个参数：task
#task.result():任务对象中封装的协程对象对应的特殊函数内部的返回值
def callback(task):
    print('this is callback!')
    print(task.result())

c= request('www.baidu.com')

#给任务对象绑定一个回调函数
task = asyncio.ensure_future(c)
task.add_done_callback(callback)

loop =asyncio.get_event_loop()
loop.run_until_complete(task)
```  
  
观察源码，在定义回调函数的时候，必须传入一个参数，该参数便是实例化的任务对象，在实例化任务对象之后，可以调用`add_done_callback`方法，传入回调函数来执行。我们来看看程序的执行结果吧。  
  
```
正在请求： www.baidu.com
下载成功 www.baidu.com
this is callback!
www.baidu.com
```  
  
值得注意的一点是，前文提到了`task`任务对象是包含协程对象的各个状态的，也就是说我们可以通过`task.result()`取得协程对象的返回值。  
  
## 5. 多任务异步协程应用到爬虫中  

前面已经详细得阐述了多任务异步协程是怎么工作的，但是我们的主要目的还是将多任务异步协程应用到爬取数据中，那应该怎么实现呢？  

首先我们回顾一下多任务异步协程的其中两个大板块————协程和回调。  

协程的作用就是让我们提高爬取数据的效率，所以所有的请求及获取数据都要写在协程的特殊函数中。  

那回调的作用是什么呢？回忆爬虫的基本步骤，请求数据后，我们就要开始解析数据了。所以回调的作用很显然就是将接解析数据的程序封装。  

理解了这个过程以后，实现起来其实不难，要注意的一点是，`requests`模块中的请求函数并不支持异步，所以在异步实现爬取数据的时候，我们便不再用`requests`模块进行请求，而是改用了`aiohttp`进行请求。  
  
为了让解释更偏向于思路，我用`flask`模块搭建了一个简单服务器，我们可以用爬虫去爬取搭建服务器的数据，服务器源码如下（还有一个原因就是百度服务器响应速度过快，没办法把异步爬虫的高效反映出来）  
  
```python
# Author:Geekboy
from flask import Flask
import time

app = Flask(__name__)

@app.route('/404player')
def index_bobo():
    time.sleep(2)
    return 'Hello 404player'

@app.route('/geek')
def index_joy():
    time.sleep(2)
    return 'Hello geek'

@app.route('/boy')
def index_tom():
    time.sleep(2)
    return 'Hello boy'

if __name__ == '__main__':
    app.run(threaded=True)
```  
  
运行服务器以后，我们可以在浏览器中看到这样的画面：  
![慢速服务器](https://img-blog.csdnimg.cn/20200229121108380.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlZWtib3lfMTIz,size_16,color_FFFFFF,t_70)  
  
接下来我们开始写程序实现爬取，为了体现出请求操作与前面的不同，我们先来把请求操作（也就是协程函数）写一遍：
  
```python
async def get_pageText(url):
    async with aiohttp.ClientSession() as s:
        async with await s.get(url=url) as response:
            page_text = await response.text()
            return page_text
```  
  
在使用`aiohttp`模块时有两点需要注意，第一点就是在协程函数所有使用with执行的函数都要用`async`来修饰（事关with的底层实现，没有商量的余地！！！），第二点注意的就是所有阻塞操作全都要加`await`挂起。  
  
用with的原因是with不需要关闭，类比文件操作的`with open`,假如不用with，直接实现`s = aiohttp.ClientSession()`的话，就需要`s.close()`来释放内存，否则会出现内存占用过大等问题。  
  
下面我贴出整个程序源码，其实回调的部分我只是把源码打印出来去看效果，毕竟获取的源码其实就是一个字符串，并不需要解析，当我们去实现聚焦爬虫的时候，将解析操作写入回调即可。  
  
```python
import asyncio
import time
import aiohttp
# 单线程+多任务异步协程

urls = [
    'http://127.0.0.1:5000/404player',
    'http://127.0.0.1:5000/geek',
    'http://127.0.0.1:5000/boy'
]
#代理操作是跟request唯一不同的一点
#async with await s.get(url=url,proxy="http://ip:port") as response:
async def get_pageText(url):
    async with aiohttp.ClientSession() as s:
        async with await s.get(url=url) as response:
            page_text = await response.text()
            return page_text
#封装回调函数用于数据解析
def parse(task):
    #获取响应数据
    page_text = task.result()
    print(page_text+'即将进行数据解析')
    #解析操作
start = time.time()
tasks = []
for url in urls:
    c = get_pageText(url)
    task = asyncio.ensure_future(c)
    #给任务对象绑定回调函数用于数据解析
    task.add_done_callback(parse)
    tasks.append(task)
loop = asyncio.get_event_loop()
loop.run_until_complete(asyncio.wait(tasks))

print(time.time() - start)
```  
  
看一下程序的运行结果：
```
Hello 404player即将进行数据解析
Hello geek即将进行数据解析
Hello boy即将进行数据解析
2.0135281085968018
```  
可以看到2秒便将三个页面解析完毕，多任务异步协程的威力由此可见了！！！    
  
## 6.总结  
  
爬虫其实是一个很好玩的东西。  
  
不可否认现在反爬机制越来越严格，相关的法律也越来越完善，但是在合法的情况下，爬取一些数据作为资料还是一个比较有趣的过程。  
  
关于爬虫的骚操作其实很多，这篇文章主要的重点在于高性能。但是爬虫最吸引人的地方在于它多变的反反爬技术，灵活地绕过反爬机制的高级操作。大家可以多去了解。  
  
希望能和大家一起探讨，一起学习，共同进步。  

