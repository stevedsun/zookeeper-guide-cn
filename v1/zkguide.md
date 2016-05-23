原文地址：
<http://zookeeper.apache.org/doc/r3.4.6/zookeeperProgrammers.html>  
翻译：$teve  
博客地址：<http://www.v2steve.com>  

<!--more-->

本文假设你已经具有一定分布式计算的基础知识。你将在第一部分看到以下内容：

* ZooKeeper数据模型
* ZooKeeper Sessions
* ZooKeeper Watches
* 一致性保证(Consistency Guarantees)

接下来的4小节讲述了程序开发的实际应用：

* 创建模块——ZooKeeper操作指引
* 编程语言接口
* 简单示例演示程序的结构
* 常见问题和故障

本文的附录中包含和ZooKeeper相关的有用信息。

### ZooKeeper的数据模型

ZooKeeper有一个类似分布式文件系统的命名体系。区别在于Zookeeper每个一个节点或子节点都可以拥有数据。节点路径是一个由斜线分开的绝对路径，注意没有相对路径。只要满足下面要求的unicode字符都可以作为节点路径：

* 空字符不能出现在路径名
* 不能出现以下字符: \u0001 - \u0019 and \u007F - \u009F
* 以下字符不允许使用: \ud800 -uF8FFF, \uFFF0-uFFFF, \uXFFFE - \uXFFFF (where X is a digit 1 - E), \uF0000 - \uFFFFF
* 字符"."可以作为一个名字的一部分, 但是"."和".."不能单独作为相对路径使用, 以下用法都是无效的: "/a/b/./c"或者"/a/b/../c"
* "zookeeper"为保留字符

#### ZNodes

ZooKeeper树结构中的节点被称为znode。各个znode维护着一组用来标记数据和访问权限发生变化的版本号。这些版本号组成的状态结构具有时间戳。Zookeeper使用版本号和时间戳来验证缓存状态，调整更新。
每次znode中的数据发生变化，znode的版本号增加。例如，每当一个客户端恢复数据时，它就接收这个版本的数据，而当一个客户端提交了更新或删除记录，它必须同时提供这个znode当前正在发生变化的数据的版本。如果这个版本和目前真实的版本不匹配，则提交无效。
__提示，在分布式程序中，一个字节点可以代表一个通用的主机，服务器，集群中的一员，客户端程序等。但是在Zookeeper中，znode代表数据节点，Servers代表组成了Zookeeper服务的机器; quorum peers refer to the servers that make up an ensemble; 客户端代表任何使用ZooKeeper服务的主机或程序。

znode作为对程序开发来说最重要的信息，有几个特性需要特别关注下：

__Watches__
客户端可以在znode上设置Watch。znode发生的变化会触发watch然后清除watch。当一个watch被触发，Zookeeper给客户端发送一个通知。更多关于watch的内容请查看ZooKeeper Watches一节。

__数据存取__
命名空间中每个znode中的数据读写是原子操作。读操作读取znode中的所有数据位，写操作则替换所有数据。每个节点都有一个访问权限控制表（ACL）来标记谁可以做什么。
zookeeper不是设计成普通的数据库或大型对象存储的。它是用来管理coordination data。coordination data包括配置文件、状态信息、rendezvous等。这些数据结构的一个共同特点就是相对较小——以千字节为准。Zookeeper的客户端和服务会检查确保每个znode上的数据小于1M，实际平均数据要远远小于1M。
大规模数据的操作会引发一些潜在的问题并且延长在网络和介质之间传输的时间。如果确实需要大型数据的存储，那么可以采用如NFS或HDFS之类的大型数据存储系统，亦或是在zookeeper中存储指向存储位置的指针。

__临时节点（Ephemeral Nodes）__
zookeeper还有临时节点的概念，这些节点的生命周期依赖于创建它们的session是否活跃。session结束时节点即被销毁。也由于这种特性，临时节点不允许有子节点。

__序列节点——命名不唯一__
当你创建节点的时候，你会需要zookeeper提供一组单调递增的计数来作为路径结尾。这个计数对父znode是唯一的。用`%010d`的格式——用0来填充的10位数（计数如此命名是为了简单排序）。例如"<path>0000000001"，注意计数器是有符号整型，超过表示范围会溢出。

#### ZooKeeper中的时间

zookeeper有很多记录时间的方式：

* Zxid(ZooKeeper Transaction Id)： zookeeper每次发生改动都会增加zxid，zxid越大，发生的时间越靠后。
* Version numbers： 对znode的改动会增加版本号。版本号包括version (znode上数据的修改数), cversion (znode的子节点的修改数), aversion (znode上ACL（权限）的修改数)。
* Ticks : 多个server构成zookeeper服务时，各个server用ticks来标记如状态上报、连接超时等事件。ticks time还间接反映了session超时的最小值（两次tick time）；如果客户端请求的最小session timeout低于这个最小值，服务端会通知客户端最小超时置为这个最小值。
* Real time : 除了每次znode创建或改动时候将时间戳记录到状态结构中外，zookeeper不使用时钟时间。

#### ZooKeeper状态结构(Stat Structure)

存在于znode中的状态结构，由以下各个部分组成：

* czxid - znode创建产生的zxid
* mzxid - znode最后一次修改的zxid
* ctime - znode创建的时间的绝对毫秒数
* mtime - znode最后一次修改的绝对毫秒数
* version - znode上数据的修改数
* cversion - 子节点修改数
* aversion - znode的ACL修改数
* ephemeralOwner - 临时节点的所有者的session id。如果此节点非临时节点，该值为0
* dataLength - znode的数据长度
* numChildren - znode子节点数

### ZooKeeper Sessions

客户端通过创建一个handle和服务端建立session连接。一旦创建完成，handle就进入了CONNECTING状态，客户端库尝试连接一台构成zookeeper的server，届时进入CONNECTED状态。通常情况下操作会介于这两种状态之间。
一旦出现了不可恢复的错误：如session中止，鉴权失败或者应用直接结束handle，则handle会进入到CLOSED状态。下图是客户端的状态转换图：

<img>状态转换图</img>

应用在创建客户端session时必须提供一串逗号分隔的主机号:端口号，每对主机端口号对应一个ZooKeeper的server（如："127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002"），客户端库会尝试连接任意一台服务器，如果连接失败或是客户端主动断开连接，客户端会自动继续与下一台服务器连接，直到连接成功。

__3.2.0版本新增内容__: 一个新的操作“chroot”可以添加在连接字符串的尾部，用来指明客户端命令运行的根目录地址。类似unix的chroot命令，例如：
"127.0.0.1:4545/app/a" or "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002/app/a"，说明客户端会以"/app/a"为根目录，所有路径都相对于根目录来设置，如"/foo/bar"的操作会运行在"/app/a/foo/bar"。
这一特性在多用户环境下非常好用，每个使用zookeeper服务的用户可以设置不同的根目录。

当客户端获得和zookeeper服务连接的handle时，zookeeper会创建一个Zookeeper session分配给客户端，用一个64-bit数字表示。一旦客户端连接了其他服务器，客户端必须把这个session id也作为连接握手的一部分发送。出于安全目的，zookeeper给session id创建一个密码，任何zookeeper服务器都可以验证密码。
当客户端创建session时密码和session id一起发送到客户端来，当客户端重新连接其他服务器时，同时要发送密码和session id。

zookeeper客户端库里有一个创建zookeeper session的参数，叫做session timeout（超时），用毫秒表示。客户端发送请求超时，服务端在超时范围内响应客户端。session超时最小为2个ticktime，最大为20个ticktime。zookeeper客户端API可以协调超时时间。
当客户端和zookeeper服务器集群断开时，它会搜索session创建时的服务器列表。最后，当至少一个服务器和客户端重新建立连接，session或被重新置为"connected"状态（超时时间内重新连接），或被置为"expired（过期）"状态（超出超时时间）。不建议在断开连接后重新创建session。ZK客户端库会帮你重新连接。特别地，我们将启发式学习模式植入客户的库中来处理类似“羊群效应”等问题。只有当你的session过期时才重新创建（托管的）。
session过期的状态转换图示例同过期session的watcher：

1. 'connected' : session正确创建，客户端和服务集群正常连接
2. .... 客户端从服务器集群断开
3. 'disconnected' : 客户端失去和服务器集群的连接
4. .... 过了一段时间, 超过了集群判定session过期的超时时间, 客户端并没有发觉自己和服务集群断开了连接
5. .... 又过一段时间, 客户端恢复了同集群的网络连接
6. 'expired' : 最终客户端重新连上集群，然后被通知已经到期

另一个session建立时zookeeper需要的参数是默认watcher（监视者）。在客户端发生任何变化时，watcher都会发出通知。例如客户端失去和服务器的连接、客户端session到期等。watcher默认的初始状态是disconnected。（也就是说任何状态改变事件都由客户端库发送到watcher）。当新建一个连接时，第一个发送给watcher的事件通常就是session连接事件。

客户端发送请求会使session保持活动状态。客户端会发送ping包（译者注：心跳包）以保持session不会超时。Ping包不仅让服务端知道客户端仍然活动，而且让客户端也知道和服务端的连接没有中断。Ping包发送时间正好可以判断是否连接中断或是重新启动一个新的服务器连接。

和服务器的连接建立成功，当一个同步或异步操作执行后，有两种情况会让客户端库判断失去连接：

1. 应用在已经失效的session上请求了一个操作时
2. zookeeper服务器有一个等待中的操作时，客户端会从那台服务器断开连接。即服务器有等待的异步调用。

__3.2.0版本新增内容 —— SessionMovedException__ 一个客户端无法查看的内部异常SessionMovedException。这个异常发生在服务端收到一个请求，这个请求的session已经在另一个服务器上重新连接。发生这种情况的原因通常是客户端发送完请求后，由于网络延时，客户端超时重新和其他服务器建立连接，当请求包到达第一台服务器时，服务器发现session已经移除并关闭了和客户端的连接。客户端一般不用理会这个问题，但是有一种情况值得注意，当两台客户端使用事先存储的session id和密码试图创建同一个连接时，第一台客户端重建连接，第二台则会被中断。

### ZooKeeper Watches

所有zookeeper的读操作——getData(), getChildren(), exists()——都可以设置一个watch。Zookeeper的watch的定义是：watch事件是一次性触发的，发送到客户端的。在监视的数据发生变化时产生watch事件。以下三点是watch(事件)定义的关键点：

* 一次性触发:
当数据发生变化时，一个watch事件被发送给客户端。例如，如果一个客户端做了一次`getData("/znode1", true)`然后节点`/znode1`发生数据变化或删除，这个客户端将收到`/znode1`的watch事件。如果`/znode1`继续发生改变，不会再有watch发送，除非客户端又做了其他读操作产生了新的watch。
* 发送给客户端:
这就意味着，事件在发往客户端的过程中，可能无法在修改操作成功的返回值到达客户端之前到达客户端。watch是异步发送给watchers的。zookeeper提供一种保证顺序的方法：客户端在第一次看到某个watch事件之前不可能看到产生watch的修改的返回值。网络延时或其他因素可能导致不同客户端看到watch并返回不同时间更新的返回值。关键的一点是，不同的客户端看到发生的一切都必须是按照相同顺序的。
* watch依附的数据:
这是说改变一个节点有不通方式。用好理解的话说，zookeeper维护两组watch：data watch和child watch。getData()和exists()产生data watch。getChildren()引起child watch。watch根据数据返回的种类不同而不同。getData()和exists()返回关于节点的数据信息，而getChildren()返回子节点列表。因此setData()触发某个znode的data watch（假设事件成功）。create()成功会触发被创建的znode上的data watch和在它父节点上的child watch。delete()成功会触发data watch和child watch（因为没有了子节点）。

watch在客户端已连接上的服务器里维护，这样可以保证watch轻量便于设置，维护和分发。当客户端连接了一台新的服务器，watch会在任何session事件时触发。当断开和服务器的连接时，watch不会触发。当客户端重新连接上时，任何之前注册过的watch都会重新注册并在需要的时候被触发。一般来说这一切都是透明的。只有一种可能会丢失watch：当一个znode在断开和服务器连接时被创建或删除，那么判断这个znode存在的watch因未创建而找不到。

__ZooKeeper如何保证watch可靠性__

zookeeper有如下方式：

* watch与其他事件、watch、异步回复保持有序，Zookeeper客户端库确保任何分发都是有序的。
* 客户端会在某个监视的znode数据更新之前看到这个znode的watch事件。
* watch事件的顺序由Zookeeper服务端观察到的更新顺序决定。

__watch注意事项__

* watch是一次性触发的；如果你收到watch事件后还想继续得到后续更改的通知，你需要再生成（设置）一个watch。
* 由于watch是一次性触发，你在获取某事件和发送新的请求来得到watch这个操作之间，无法确保观察到Zookeeper中那个节点在这期间的所有修改。你要准备好应付这种情况出现：znode会在收到事件和再次设置新事件（译者注：对节点的操作）之间发生了多次修改。（你可能并不关心，但是必须了解这可能发生）
* watch对象，或是function/context对，只会在得到通知时触发一次。例如，如果一个watch对象同时用来监控某个目标文件是否存在和监听getData()，之后那个文件被删除了。那么这个watch对象只会触发一次文件删除事件通知。
* 如果你断开了同服务器的连接（例如服务器挂了），你在重新连上之前得不到任何watch。出于这种原因，session event会被发送给所有重要的watch handler。可以使用session事件进入安全模式：当断开连接时你收不到任何事件，这样你的进程可以在那种模式下稳健地执行。(译者注：可以通过发送session event使客户端进入安全模式（伪断开连接状态），在安全模式你可以修改代码而不用担心程序收到事件通知)

### 使用ACL控制ZooKeeper访问权限
zookeeper使用ACL来控制对znode（zookeeper的数据节点）的访问权限。ACL的实现方式和unix的文件权限类似：用不同位来代表不同的操作限制和组限制。与标准unix权限不同的是，zookeeper的节点没有三种域——用户，组，其他。zookeeper里没有节点的所有者的概念。取而代之的是，一个由ACL指定的id集合和其相关联的权限。
注意，一个ACL只从属于一个特定的znode。对这个znode子节点也是无效的。例如，如果`/app`只有被ip172.16.16.1的读权限，`/app/status`有被所有人读的权限，那么`/app/status`可以被所有人读，ACL权限不具有递归性。
zookeeper支持插件式认证方式，id使用`scheme:id`的形式。`scheme`是id对应的类型方式，例如`ip:172.16.16.1`就是一个地址为172.16.16.1的主机id。
当客户端连接zookeeper并且认证自己，zookeeper就在这个与客户端的连接中关联所有与客户端一致的id。当客户端访问某个znode时，znode的ACL会重新检查这些id。ACL的表达式为`(scheme:expression,perms)`。`expression`就是特殊的scheme，例如，`(ip:19.22.0.0/16, READ)`就是把任何以19.22开头的ip地址的客户端赋予读权限。

#### ACL权限

ZooKeeper支持下列权限：

* CREATE：允许创建子节点
* READ：允许获得节点数据并列出所有子节点
* WRITE：允许设置节点上的数据
* DELETE：允许删除子节点
* ADMIN：允许设置权限

CREATE和DELETE操作是更细的粒度上的WRITE操作。有一种特殊的情况：

* 你想要A获得操作zookeeper上某个znode的权限，但是不可以对其子节点进行CREATE和DELETE。
* 只CREATE不DELETE：某个客户端在上一级目录上通过发送创建请求创建了一个zookeeper节点。你希望所有客户端都可以在这个节点上添加，但是只有创建者可以删除。（这就类似于文件的APPEND权限）

zookeeper没有文件所有者的概念，但有ADMIN权限。在某种意义上说，ADMIN权限指定了所谓的所有者。zookeeper虽然不支持查找权限（在目录上的执行权限虽然不能列出目录内容，却可以查找），但每个客户端都隐含着拥有查找权限。这样你可以查看节点状态，但仅此而已。（这有个问题，如果你在不存在的节点上调用了zoo_exists()，你将无权查看）

#### 内建ACL模式

ZooKeeper有下列内建模式：

* __world__  有独立id，anyone，代表任何用户。
* __auth__ 不使用任何id，代表任何已经认证过的用户
* __digest__ 之前使用了格式为`username:pathasowrd`的字符串来生成一个MD5哈希表作为ACL ID标识。在空文档中发送`username:password`来完成认证。现在的ACL表达式格式为`username:base64`, 用SHA1编码密码。
* __ip__ 用客户端的ip作为ACL ID标识。ACL表达式的格式为`addr/bits`，addr中最有效的位匹配上主机ip最有效的位。


#### ZooKeeper C client API

### 插件式ZooKeeper认证

zookeeper运行于复杂的环境下，有各种不同的认证方式。因此zookeeper拥有一套插件式的认证框架。内建认证scheme也是使用这套框架。
为了便于理解认证框架的工作方式，你首先要了解两种主要的认证操作。框架首先必须认证客户端。这步操作通常在客户端连接服务器的同时完成并且将从客户端发过来的（或从客户端收集来的）认证信息关联此次连接。认证框架的第二步操作是在ACL中寻找关联的客户端的条目。ACL条目是`<idspec, permissions>`格式。idspec可能是一个关联了连接的，和认证信息匹配的简单字符串，也可能是评估认证信息的表达式。这取决于认证插件如何实现匹配。下面是一个认证插件必须实现的接口：

    public interface AuthenticationProvider {
        String getScheme();
        KeeperException.Code handleAuthentication(ServerCnxn cnxn, byte authData[]);
        boolean isValid(String id);
        boolean matches(String id, String aclExpr);
        boolean isAuthenticated();
    }

第一个方法`getScheme`返回一个标识该插件的字符串。由于我们支持多种认证方式，认证证书或者idspec必须一直加上`scheme:`作为前缀。zookeeper服务器使用认证插件返回的scheme判断哪个id适用于该scheme。
当客户端发送与连接关联的认证信息时，handleAuthentication被调用。客户端指定和认证信息相应的模式。zookeeper把信息传给认证插件，认证插件的`getScheme`匹配scheme。实现`handleAuthentication`的方法通常在判断信息错误后返回一个error，或者在确认连接后使用`cnxn.getAuthInfo().add(new Id(getScheme(), data))`

认证插件在设置和ACL中都有涉及。当对某个节点设置ACL时，zookeeper服务器会传那个条目的id给`isValid(String id)`方法。插件需要判断id的连接来源。例如，`ip:172.16.0.0/16`是有效id，ip:host.com是无效id。如果新的ACL包括一个"auth"条目，就用`isAuthenticated`判断该scheme的认证信息是否关联了连接，是否可以被添加到ACL中。一些scheme不会被包含到auth中。例如，如果auth已经指定，客户端的ip地址就不作为id添加到ACL中。
在检查ACL时zookeeper有一个`matches(String id, String aclExpr)`方法。ACL的条目需要和认证信息相匹配。为了找到和客户端对应的条目，zookeeper服务器寻找每个条目的scheme，如果对某个scheme有那个客户端的认证信息，`matches(String id, String aclExpr)`会被调用并传入两个值，分别是事先由`handleAuthentication` 加入连接信息中认证信息的id，和设置到ACL条目id的`aclExpr`。认证插件用自己的逻辑匹配scheme来判断id是否在aclExpr中。

有两个内置认证插件：ip和digest。附加插件可以使用系统属性添加。在zookeeper启动过程中，会扫描所有以"zookeeper.authProvider"开头的系统属性。并且把那些属性值解释为认证插件的类名。这些属性可以使用`-Dzookeeeper.authProvider.X=com.f.MyAuth`或在服务器设置文件中添加条目来创建：

    authProvider.1=com.f.MyAuth
    authProvider.2=com.f.MyAuth2

注意属性的后缀是唯一的。如果出现重复的情况`-Dzookeeeper.authProvider.X=com.f.MyAuth -Dzookeeper.authProvider.X=com.f.MyAuth2`，只有一个会被使用。同样，所有服务器都必须统一插件定义，否则客户端用插件提供的认证schemes连接服务器时会出错。

### 一致性保证

ZooKeeper是一个高性能，可扩展的服务。读和写操作都非常快速。之所以如此，全因为zookeeper有数据一致性的保证：

__顺序一致性__
客户端的更新会按照它们发送的次序排序。

__原子性__
更新的失败或成功，都不会出现半个结果。

__单独系统镜像__
不管客户端连哪个服务器，它看来都是同一个。

__可靠性__
一旦更新生效，它就会一直保存到下一次客户端更新。这就有两个推论：

1. 如果客户端得到成功的返回值，说明更新生效了。在一些错误情况下（连接错误，超时等）客户端不会知道更新是否生效。虽然我们使失败的几率最小化，但是也只能保证成功的返回值情况。（这就叫Paxos算法的单调性条件）
2. 客户端能看到的更新，即使是渡请求或成功更新，在服务器失败时也不会回滚。

__时效性__
客户端看到的系统状态在某个时间范围内是最新的（几十秒内），任何系统更改都会在该时间范围内被客户端发现。否则客户端会检测到断开服务。

用这些一致性保证可以在客户端中构造出更高级的程序如 leader election, barriers, queues, read/write revocable locks(无须在zookeeper中附加任何东西)。更多信息[Recipes and Solutions](http://zookeeper.apache.org/doc/r3.4.6/recipes.html)

> zookeeper不存在的一致性保证： 多客户端同一时刻看到的内容相同
> zookeeper不可能保证两台客户端在同一时间看到的内容总是一样，由于网络延迟等原因。假设这样一个场景，A和B是两个客户端，A设置节点/a下的值从0变为1，然后让B读/a，B可能读到旧的数据0。如果想让A和B读到同样的内容，B必须在读之前调用zookeeper接口中的sync()方法。

### 编程接口

### 常见问题和故障

下面是一些常见的陷阱：

1. 如果你使用watch，你必须监控好已经连接的watch事件。当ZooKeeper客户端断开和服务器的连接，直到重新连接上这段时间你都收不到任何通知。如果你正在监视znode是否存在，那么你在断开连接期间收不到它创建和销毁的通知。
2. 你必须测试ZooKeeper故障的情况。在大多数服务器都可用的情况下，ZooKeeper是可以维持工作的。关键问题是你的客户端程序是否能察觉到。在实际情况下，客户端与ZooKeeper的连接有可能中断（多数时候是因为Zookeeper故障或网络中断）。Zookeeper的客户端库关注于如何让你重新连接并且知道发生了什么。但是同时你也必须确保能够恢复你的状态和发送失败的请求。努力在测试库里测出这些问题，而不是在产品里——用几台服务器组成的zookeeper集群测试这个问题，尝试让它们重启。
3. 客户端维护的服务器列表必须和现有的服务器列表一致。如果客户端的列表是现有服务器列表的子集，还可以在非最佳状态工作，但是如果客户端列表里的服务器不在现有集群里你就悲剧了。
4. 注意存放事务日志的位置。性能评测最重要的部分就是日志，ZooKeeper会在回复响应之前先把日志同步到磁盘上。为了达到最佳性能，首选专用的磁盘来存日志。把日志放在繁忙的磁盘上会降低效率。如果你只有一个磁盘，就把记录文件放在NFS上然后增加SnapshotCount。这样虽然无法完全解决问题，但能缓解一些。
5. 正确地设置你java的堆空间大小。这是避免频繁交换的有效措施。无用的访问磁盘会让你的效率大打折扣。记住，在ZooKeeper中，一切都是有序的，如果一个服务器访问了磁盘，所有的服务器都会同步这个操作。

其他资料链接请自行官网查看。

