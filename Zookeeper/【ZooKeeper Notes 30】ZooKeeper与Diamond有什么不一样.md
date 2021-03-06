<p><span style="font-family: 'Comic Sans MS';">转载请注明：</span><a href="http://weibo.com/nileader" target="_blank"><span style="font-family: 'Comic Sans MS';">@ni掌柜</span></a><span style="font-family: 'Comic Sans MS';"> nileader@gmail.com</span></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;本文主要是讨论下两个类似产品：ZooKeeper和Diamond在配置管理这个应用场景上的异同点。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;Diamond，顾名思义，寄寓了开发人员对产品稳定性的厚望，希望它像钻石一样，提供稳定的配置访问。Diamond是淘宝网Java中间件团队的核心产品之一，服务于集团线上很多核心应用。目前已经开源，开源地址在：http://code.taobao.org/p/diamond/wiki/index/。</span></p> 
<p><span style="font-family: 'Comic Sans MS';"> </span></p> 
<p><strong><span style="font-size: 22px;">数据持久性</span></strong></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;Diamond主要针对的是持久数据，这些数据有个共同的特点是：集群中一批机器都会使用，但是数据的更新频率不大，且希望diamond能够永久存储。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;ZooKeeper即可以存储持久数据，也可以存储非持久数据。持久数据和diamond中的持久数据都类似，所谓的非持久数据是指这些数据的生命周期和数据创建者的会话生命周期绑定，一旦会话结束，那么这些非持久数据也会被清除。</span></p> 
<p><span style="font-family: 'Comic Sans MS';"> </span></p> 
<p><strong><span style="font-size: 22px;">推拉模型</span></strong></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;本质上，两个产品都是“拉”模式的，即都是通过客户端自己去服务器获取最新数据。具体实现上，两个产品分别如下：</span></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;在Diamond中，客户端每隔15s轮询服务器，比对数据是否更新，从而获取最新数据。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;在ZooKeeper中，则是通过客户端对相应的数据path注册Watcher，当数据有更新的时候，服务器会有事件通知，注意，这个通知仅仅是告诉客户端对应的数据有更新了，具体数据内容需要客户端根据自己的情况来决定是否需要获取最新数据。因此在实时性方面，ZooKeeper比Diamond高一些。</span></p> 
<p><span style="font-family: 'Comic Sans MS';"> </span></p> 
<p><strong><span style="font-size: 22px;">服务器数据存储</span></strong></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;在数据存储上，ZooKeeper和Diamond差别比较大。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;首先来看下Diamond的数据存储。Diamond的数据存储以mysql数据库为中心，所有在mysql中的数据都是最新的，客户端的所有写请求，都会首先写入数据库，同时会dump数据到Server的本地文件中，所有读请求都是直接走这个静态文件。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;在ZooKeeper中，所有运行时数据都是存储在内存中，客户端的所有读写操作都是针对这份内存数据来进行的。同时，内存中的数据，ZK会以快照的形式dump到指定文件中去，配合事务日志，帮助服务器在下次重启的时候，能够加载正确的数据到内存中去。</span></p> 
<p><span style="font-family: 'Comic Sans MS';"> </span></p> 
<p><strong><span style="font-size: 22px;">数据模型</span></strong></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;Diamond的数据都是以行组织的，这也更便于它使用mysql来管理数据。Diamond的基本数据结构包含dataid，group和content，根据group，可以将一组相关的数据组合起来。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">&nbsp; &nbsp;ZooKeeper中，使用树形结构来组织数据，每个节点类型于一个文件系统的路径，一个节点下面也可以创建多个子节点来规则一些相关的数据。</span></p> 
<p><span style="font-family: 'Comic Sans MS';"> </span></p> 
<p><strong><span style="font-size: 22px;">容灾</span></strong></p> 
<p><span style="font-family: 'Comic Sans MS';">在容灾方面，diamond做得相当的完备：</span></p> 
<p><span style="font-family: 'Comic Sans MS';">1．所有客户端的读请求，都是直接读取服务器端的本地静态文件，因此，即使数据库挂了，都不会影响diamond的读服务。而读服务在所有使用diamond的应用场景中，占到了绝大部分。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">2．Diamond客户端还保存了数据的快照，客户端每次从服务器成功获取数据后，都会把这份数据保存到本地文件系统中，称为快照文件。这个快照文件是为了防止在服务器无法获取数据的时候，能够在这个快照中获取数据。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">3．客户端还会有一个容灾目录，变个容灾目录是在服务器完全不可用的时候，运维人员可以手动在这个容灾目录中创建相关目录结构的数据，diamond就就会优先从这个目录中获取数据。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">4. 说到这里，我们就可以给diamond的数据获取优先级作一个总结：</span></p> 
<p><span style="font-family: 'Comic Sans MS';">首先都会从容灾目录中获取数据&#x2014;&#x2014;无法从容灾目录获取数据的话，就通过网络到服务器请求数据&#x2014;&#x2014;如果无法从服务器获取数据，那么就从本地的snapshot中获取数据。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">接下来看看ZooKeepe的容灾，做得很少，只有以下一点：</span></p> 
<p><span style="font-family: 'Comic Sans MS';">1.ZooKeeper实现了paxos算法，有效的解决了分布式单点问题。以一个3台机器构成的集群为例，任意一台ZK挂掉，都不会影响集群的数据一致性。</span></p> 
<p><span style="font-family: 'Comic Sans MS';">总结：在容灾方面，diamond有很大的优势，也符合了diamond的稳定性要求。</span></p> 
<p><span style="font-family: 'Comic Sans MS';"> </span></p> 
<p><strong><span style="font-size: 22px;">数据大小</span></strong></p> 
<p><span style="font-family: 'Comic Sans MS';">Diamond对单个数据的大小，没有严格的限制，通常2M左右的数据大小都是没有问题的。而在ZooKeeper中，由于全量数据都是存储在内存中，并且需求进行集群机器间的数据两步，所以对单个数据的大小有严格的限制，默认单个数据节点的最大数据大小是1M。</span></p> 
<div>
 <span style="font-family: 'Comic Sans MS';"><strong><span style="font-size: 22px;">数据追加与聚合</span></strong></span>
</div> 
<div>
 <span style="font-family: 'Comic Sans MS';">Diamond支持对数据的追加与聚合功能，即对同一个dataid的写入操作，可以设置为追加。而ZooKeeper目前不支持，只有覆盖写。</span>
</div> 
<p>&nbsp;</p>
<p>本文出自 “<a href="http://nileader.blog.51cto.com">ni掌柜的IT专栏</a>” 博客，请务必保留此出处<a href="http://nileader.blog.51cto.com/1381108/1046316">http://nileader.blog.51cto.com/1381108/1046316</a></p>
