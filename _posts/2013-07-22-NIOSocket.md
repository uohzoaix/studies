---
layout: post
title: "再聊NIO之socket"
category: 
- NIO
- socket
tags: []
---














上个星期完成了类似微信聊天的聊天系统，服务器端由我一人实现，android和apple客户端分别由另两位同事完成，其实要完成一个聊天系统是非常简 单的，但是由于很多的原因（下文会讲）我们不得不使用一些成熟的框架来实现，在讨论使用何种socket框架实现时，我本想使用netty，但他们两个对 netty都不熟，对xsocket都熟，而我又是第一次听说过xsocket，经过权衡最后决定使用xsocket实现。</br></br>
在网上找了些例子熟悉 之后就开始搞了，这篇文章讲的不是如何实现这个聊天系统（下文我会专门来讲讲这个聊天系统的实现细节），而是xsocket这个东西。今天到公司就开始看 xsocket的源码，使用的还是java NIO的东西，NIO这个似仙女的东西看再多遍也不为过，讲多少遍也不觉得烦。先来讲讲NIO的主要步骤的实现吧。</br></br>
在NIO中主要用到的几个类有 Selector，SelectionKey，SocketChannel，ByteBuffer，要使用socket进行通信我们首先必须将 socket通道连接上服务器，但是再NIO中不需要你手动去连接，连接的这个操作是由NIO进行的，我们要做的就是通知NIO要去做连接这件事情，所以 从此可以看出NIO是基于反应器模式的，而真正重要的就是如何做到能够有事请就发出通知的这个过程。首先要做的就是获取一个Selector选择器，这个 选择器会在以后去寻找它感兴趣的操作，什么叫感兴趣的操作，如何定义感兴趣的操作呢？这个时候可以通过SocketChannel的register方法 向Selector注册这个Selector感兴趣的动作，所谓感兴趣意思就是说这个动作会在之后的操作被Selector扫描到并且执行，这个 register方法返回的是SelectionKey对象，这个时候如果SocketChannel去执行某个操作，比如执行connect方法，在执 行的过程中这个上文的选择器Selector循环执行select方法，这个方法的作用是让操作系统在上下文中寻找该选择器感兴趣的操作，如果找到例如某 个channel向Seletor注册了OP_CONNECT事件并且这个channel正在执行connect操作，这个时候select方法就会返回 一个大于0的整数，代表有多少个感兴趣的事件发生，这个时候我们可以将寻找到的SelectionKey放入一个集合中，之后可遍历该集合并可在实际情况 中进行相关方法的回调，回调可通过SelectionKey的attachment方法进行将某个类附加到该SelectionKey上。</br></br>
对于NIO的write，read方法，流程也是一样的，通过注册感兴趣的操作并通知它去完成操作。只不过在write和read中需要使用ByteBuffer来存取数据。</br></br>
现在市面上的NIO框架都会存在这些最基本的NIO操作，不一样的是对回调方法和线程池等的使用。只要掌握了这些基本操作使用任何一个NIO框架都是手到擒来的。</br></br>
下篇文章会专门来讲一下我们这个聊天系统的实现细节，其实主要的东西都已经讲了，需要讲的只是一些实现聊天系统应该注意的事项和较复杂的业务功能。