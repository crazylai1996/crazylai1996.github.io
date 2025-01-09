---
title: Dubbo源码学习：心跳机制
date: 2025-01-09 23:31:10
tags: Dubbo

---



### 前言

在 Dubbo 中，默认协议使用单一长连接进行通信，这可以避免每次调用都新建 TCP 连接，从而降低延迟、提高调用的响应速度，但长连接的稳定性受环境、服务端等影响可能会断开，所以如何及时检测连接是否可用并进行重连（保活），对于通信框架而言是非常重要的话题。

<!--more-->

> 以下内容基于Dubbo 2.7.12版本



### 连接保活

#### 为什么需要保活

在了解 Dubbo 的方案前，我们可以先思考下有哪些因素会影响长连接的的稳定性：

1. 在网络通信中，客户端和服务端之间的数据传输通常会经过多个中间设备（如NAT网关、防火墙等）。这些设备为了节省资源或遵循安全策略，可能会在一段时间内没有数据传输时主动删除 TCP 会话信息；
2. 网络链路故障，或者进入不稳定或者拥堵的网络环境，可能会导致连接被关闭；
3. 服务端应用进程被关闭。



#### 如何保活

那为了确保连接的可用性，以及及时发现异常断线情况并进行重连，我们可以采取哪些措施呢？

提到保活，我们都知道 TCP keepalive机制，但这个其实是操作系统实现的一个功能，在开启这个机制后，当长连接无数据交互一定时间间隔时，连接的一方会向对端发送保活探测包，在连接正常时，对端将对此作出回应。

那既然如此，是不是只要开启 keepalive 就行了呢？

其实不够，因为 keepalive 只是在网络层进行保活，假使网络本身没问题，但是由于服务端进程假死等异常情况时，此时 keepalive 就无法探测到了，在这种情况下，往往需要应用层去实现心跳机制，按照一定的时间间隔，发送心跳包到对端，以此确认连接对端是否可达。而相对 TCP 的 keepalive，实现业务心跳：

1. 不仅可以检测连接是否正常，也可以检测对端进程是否正常
2. 具有更大的灵活性，可以自己控制检测的间隔，检测的方式等；
3. 另外，心跳包还可以附带业务信息，定时在服务端和客户端之间同步。

但在应用层实现心跳也需要考虑资源占用情况，心跳检测过于频繁会增加系统的负担，而间隔太长又可能会导致不可用的连接不能尽早的被发现，这是一个需要权衡的问题。

本文将一起来学习下，在 Dubbo 中，连接是如何保活的。



### Dubbo 的心跳机制

#### 服务消费端

ExchangeClient 是 Dubbo 对服务消费端信息交换层的抽象，而 HeaderExchangeClient 是其默认实现，我们直接来到其构造方法：

```Java
public HeaderExchangeClient(Client client, boolean startTimer) {
    Assert.notNull(client, "Client can't be null");
    // 客户端实现
    this.client = client;
    this.channel = new HeaderExchangeChannel(client);

    if (startTimer) {
        URL url = client.getUrl();
        // 开启重连定时任务
        startReconnectTask(url);
        // 开启心跳定时任务
        startHeartBeatTask(url);
    }
}
```



startReconnectTask 开启了一个重连定时任务：

```Java
private void startReconnectTask(URL url) {
    // 可以通过配置是不重连，默认重连
    if (shouldReconnect(url)) {
        AbstractTimerTask.ChannelProvider cp = () -> Collections.singletonList(HeaderExchangeClient.this);
        // 获取空闲的超时时间，默认为心跳 * 3，即心跳达到3次不回复则进行重连
        int idleTimeout = getIdleTimeout(url);
        // 计算任务的执行频率，实际是将配置的超时时间除以3后得到的值
        long heartbeatTimeoutTick = calculateLeastDuration(idleTimeout);
        this.reconnectTimerTask = new ReconnectTimerTask(cp, heartbeatTimeoutTick, idleTimeout);
        IDLE_CHECK_TIMER.newTimeout(reconnectTimerTask, heartbeatTimeoutTick, TimeUnit.MILLISECONDS);
    }
}
```

重连定时任务 ReconnectTimerTask#doTask：

```Java
protected void doTask(Channel channel) {
    try {
        // 获取上一次收到回复的时间
        Long lastRead = lastRead(channel);
        Long now = now();

        // Rely on reconnect timer to reconnect when AbstractClient.doConnect fails to init the connection
        // 如果连接断开了，直接重连
        if (!channel.isConnected()) {
            try {
                logger.info("Initial connection to " + channel);
                ((Client) channel).reconnect();
            } catch (Exception e) {
                logger.error("Fail to connect to " + channel, e);
            }
        // check pong at client
        } else if (lastRead != null && now - lastRead > idleTimeout) {
            // 如果空闲时间超过指定时间，则进行重连
            logger.warn("Reconnect to channel " + channel + ", because heartbeat read idle time out: "
                    + idleTimeout + "ms");
            try {
                ((Client) channel).reconnect();
            } catch (Exception e) {
                logger.error(channel + "reconnect failed during idle time.", e);
            }
        }
    } catch (Throwable t) {
        logger.warn("Exception when reconnect to remote channel " + channel.getRemoteAddress(), t);
    }
}
```



startHeartBeatTask 开启了一个心跳定时任务：

```Java
private void startHeartBeatTask(URL url) {
    // 客户端是否能够处理空闲事件
    if (!client.canHandleIdle()) {
        AbstractTimerTask.ChannelProvider cp = () -> Collections.singletonList(HeaderExchangeClient.this);
        // 获取心跳间隔时间，默认为 60s
        int heartbeat = getHeartbeat(url);
        // 心跳间隔时间
        long heartbeatTick = calculateLeastDuration(heartbeat);
        this.heartBeatTimerTask = new HeartbeatTimerTask(cp, heartbeatTick, heartbeat);
        IDLE_CHECK_TIMER.newTimeout(heartBeatTimerTask, heartbeatTick, TimeUnit.MILLISECONDS);
    }
}
```

在获取心跳任务的执行时间间隔时，这里跟重连任务一样，同时是将时间除以3得到，这是为什么呢？这其实是 Dubbo 缩短了检测间隔时间，增大了及时发现死链的概率。

心跳定时任务 HeartbeatTimerTask#doTask：

```Java
protected void doTask(Channel channel) {
    try {
        // 上一次读取响应时间 
        Long lastRead = lastRead(channel);
        // 上一次发出请求时间
        Long lastWrite = lastWrite(channel);
        // 超过心跳间隔则进行发起心跳请求
        if ((lastRead != null && now() - lastRead > heartbeat)
                || (lastWrite != null && now() - lastWrite > heartbeat)) {
            Request req = new Request();
            req.setVersion(Version.getProtocolVersion());
            req.setTwoWay(true);
            req.setEvent(HEARTBEAT_EVENT);
            channel.send(req);
            if (logger.isDebugEnabled()) {
                logger.debug("Send heartbeat to remote channel " + channel.getRemoteAddress()
                        + ", cause: The channel has no data-transmission exceeds a heartbeat period: "
                        + heartbeat + "ms");
            }
        }
    } catch (Throwable t) {
        logger.warn("Exception when heartbeat to remote channel " + channel.getRemoteAddress(), t);
    }
}
```

另外，我们还注意到 ，在 startHeartBeatTask 开启心跳任务前，有个前提条件的判断：

```Java
client.canHandleIdle()
```

查看实现得知，这是 Netty 对空闲连接的检测提供了天然支持，在使用 Netty 时，可以通过 IdleStateHandler 很方便的实现空闲连接检测。在 NettyClientHandler 中，提供了对空闲事件的处理：

```Java
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
    // send heartbeat when read idle.
    if (evt instanceof IdleStateEvent) {
        try {
            NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
            if (logger.isDebugEnabled()) {
                logger.debug("IdleStateEvent triggered, send heartbeat to channel " + channel);
            }
            // 发送心跳请求
            Request req = new Request();
            req.setVersion(Version.getProtocolVersion());
            req.setTwoWay(true);
            req.setEvent(HEARTBEAT_EVENT);
            channel.send(req);
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.channel());
        }
    } else {
        super.userEventTriggered(ctx, evt);
    }
}
```

有了 IdleStateHandler 的支持，Dubbo 可以减少一个心跳任务的定时器，从而降低资源消耗。



#### 服务提供端

在服务提供端，HeaderExchangeServer 是 Dubbo 信息交互层的默认实现在其构造函数中：

```Java
public HeaderExchangeServer(RemotingServer server) {
    Assert.notNull(server, "server == null");
    this.server = server;
    // 开启空闲检测任务
    startIdleCheckTask(getUrl());
}
```

会开启一个空闲检测任务：

```Java
private void start客户端是否能够处理空闲事件IdleCheckTask(URL url) {
    // 服务端是否能够处理空闲事件
    if (!server.canHandleIdle()) {
        AbstractTimerTask.ChannelProvider cp = () -> unmodifiableCollection(HeaderExchangeServer.this.getChannels());
        // 获取空闲超时时间，默认心跳间隔*3
        int idleTimeout = getIdleTimeout(url);
        // 计算执行时间间隔
        long idleTimeoutTick = calculateLeastDuration(idleTimeout);
        // 开启了一个关闭连接定时任务
        CloseTimerTask closeTimerTask = new CloseTimerTask(cp, idleTimeoutTick, idleTimeout);
        this.closeTimerTask = closeTimerTask;

        // init task and start timer.
        IDLE_CHECK_TIMER.newTimeout(closeTimerTask, idleTimeoutTick, TimeUnit.MILLISECONDS);
    }
}
```

关闭连接定时任务 CloseTimerTask#doTask：

```Java
protected void doTask(Channel channel) {
    try {
        // 上一次读取响应时间 
        Long lastRead = lastRead(channel);
        // 上一次发出请求时间
        Long lastWrite = lastWrite(channel);
        Long now = now();
        // check ping & pong at server
        // 超过空闲超时时间，则直接断开连接
        if ((lastRead != null && now - lastRead > idleTimeout)
                || (lastWrite != null && now - lastWrite > idleTimeout)) {
            logger.warn("Close channel " + channel + ", because idleCheck timeout: "
                    + idleTimeout + "ms");
            channel.close();
        }
    } catch (Throwable t) {
        logger.warn("Exception when close remote channel " + channel.getRemoteAddress(), t);
    }
}
```

可以看到，在服务提供端这边，如果在指定的空闲超时时间内没有发生读写，会直接关闭连接。

而对于 NettyServer 的空闲处理：

```Java
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
    // server will close channel when server don't receive any heartbeat from client util timeout.
    if (evt instanceof IdleStateEvent) {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
        try {
            logger.info("IdleStateEvent triggered, close channel " + channel);
            // 关闭连接
            channel.close();
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.channel());
        }
    }
    super.userEventTriggered(ctx, evt);
}
```

也是对连接直接关闭。



### 总结

在 Dubbo 服务消费端，会启动两个定时任务，IdleStateHandler主要用于定时发送心跳请求，而 ReconnectTimerTask 用于3次心跳未回复之后进行重连的逻辑；而在服务提供端，只启动了一个 CloseTimerTask，用于检测空闲超时时间关闭连接。另外对于 Netty 的实现客户端/服务端，则使用了 IdleStateHandler 代替了其中的 IdleStateHandler 及 CloseTimerTask。

