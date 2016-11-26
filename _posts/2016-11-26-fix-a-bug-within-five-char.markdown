---
layout: post
title:  "改 5 个字符修了一个 bug"
date:   2016-11-26 21:31:26 +0800
categories: blog
---
公司决定赌一把，把所有项目组撤销合并到《石器时代》里，毕竟腾讯发行，希望很大。石器的代码源自新仙剑，是 c# 和 c++ 编写的。当时我在上家公司做平台的时候，奎斌写了服务器的第一版，依然怀念每天一起去华师大吃饭扯皮的日子。

之后几经辗转，换了几批人，这些满是补丁的代码成了石器的服务端基础代码。代码年久失修，所以前段时间他们决定用 netty 重写网关，我加入的时候这个新版网关正在测试。测试反馈了一个 bug，压力测试环境中，机器人全部杀掉之后，有部分机器人没有下线。我相对清闲，又是公司第一个用 netty 写服务端的，于是去解决这个 bug。从表现看，正常下线和不正常下线的机器人都有，比较符合并发 bug 的表现，于是去翻代码，特别注意有并发的部分。

机器人下线的代码是这么写的。

{% highlight java %}
public void channelInactive(ChannelHandlerContext ctx) throws Exception {
    DisconnectMessage disconnect = new DisconnectMessage();
    //... other code
    queue.offerMessage(disconnect);
}
{% endhighlight %}

queue 部分的代码是这样的

{% highlight java %}
private ReadWriteLock rwLock = new ReetrantReadWriteLock();

public boolean offerMessage(Message msg) {
    Lock l = rwLock.readLock(); // bug is here
    l.lock();
    try {
        queue.offer(msg);
    } finally {
        l.unlock();
    }
}

public Queue<Message> getMessageQueue() {
    if(queue.isEmpty()) {
        return null;
    }
    Queue<Message> rtn = new LinkedList<>();
    Lock l = rwLock.writeLock();
    l.lock();
    try {
        rtn.addAll(queue);
        queue.clear();
    } finally {
        l.unlock();
    }
    return rtn;
}

....
{% endhighlight %}

bug 找到了，跟预期一直，并发问题。写这段代码的人不熟悉 java 的锁又没有仔细看文档，写出了 bug 代码，简单修改下，把 `readLock` 改成 `writeLock` 就可以了。修改 5 个字符，改掉一个 bug，感觉还是很爽的。