# muduo

### 使用流程

a.首先定义一个EventLoop(baseLoop), InetAddress,创建了一个server(构造函数由给定的参数初始化底层的server,相当于创建了一个TCPserver对象,)

b.然后设置回调,设置底层loop线程数量

c.分离了网络和开发,开发者只要关注onConnection方法,关注连接建立或者断开的回调以及可读写事件回调等

d.最后,调用server的start,启动loop

###  TcpServer

1.首先创建acceptor对象--->

accept->流程

创建一个非阻塞的Fd,打包成一个Channel(,往mainLoop的poll上扔.),然后设置一些tcp选项(Addr,Port),bind绑定了ListenAddr,,接着设置了一个关键的setReadCallback回调(也就是说网络连接的时候 accpt channel只关心读事件,执行Acceptor::handleRead)

(这里acceptChannel_.setReadCallback,当有新用户连接的时候,底层的channel就回去执行readCallback事件,也就是这里的Acceptor::handleRead,而这个handleRead也就newConnectionCallback (这个回调是accept设置的,而accept又是有TcpServer管理的,即TCPServer设置的))

2.然后创建threadPool-->

当前还没有开启loop线程

3.接着acceptor_->setNewConnectionCallback(这里就对应到了TcpServer::newConnection的回调)

4.接下来设置newConnectionCallBack, setTreadNum

5.设置start

这里通过atomic变量started 在一个线程里面只能只能去调用一次tcp::server的strat

a. threadPool_->start(threadInitCallback_); // 启动底层的loop线程池

-->创建loop子线程并开启loop.loop()

-->为了唤醒subloop, 每个线程都有一个wake_up fd注册在相应的loop的poll上

b.loop_->runInLoop(std::bind(&Acceptor::listen, acceptor_.get()))

-->把Acceptor::listen注册在baseloop上

6.最后就是baseloop.loop(),开启loop

#### 小结一下

listenfd

bind

setsockoption

setReadCallback-->handleRead-->newConnectionCallback

​																				|

​																				V

​															TCPServer的构造函数中

​															acceptor->setNewConnetionCallback

​																				|

​											     响应				   V												轮询算法选择subLoop<-ioLoop

​				 	有新用户连接--------------->TCP Server::newConnetion		--------> 创建TcpConnetion对象

​																																	 注册回调

​																									          	close的回调->TcpServer::removeConnection

​																					                	ioLoop->runInLoop->TcpConnection::connetEstablished

​                                                          连接建立----->TcpConnection::connectEstablished

1.防止意外tcpconnetion被杀死 用channel的tie的弱智能指针绑定

2.channel_->enableReading 然后向poller注册channel的epollin事件 (即注册到某个选择的subloop上)

3.执行新连接建立，执行回调   connectionCallback  连接成功

4.conn->shutdown--->执行TCP Server::removeConnetion ---> removeConnectionInLoop--->然后onnections_.erase(conn->name())--->ioLoop = conn->getLoop(获取连接的subLoop)--->TcpConnection::connectDestroyed--->设置状态和channel_->disableAll---> channel_->remove()(把channel从poller中删除掉)
