---
layout: post
title: "Google Glass（文中简称GG）小觑"
category: 
- Google Glass
tags: []
---



GG的原理可以参看http://www.zhihu.com/question/20276179该文，硬件嵌入式方面的东西就不讲了，我也不懂，咱就来看看GG里的一些功能是如何实现的吧。</br></br>
GG 有拍照，记录你在某个时刻做了什么事，可以将你的一些照片、视频等信息和他人分享已达到社交功能，记录您的位置信息（我觉得这个可以和Google Map结合，以后妈妈再也不会担心我迷路了），对您订阅的东西进行各种不同操作等等功能。这些功能和用户之间的交互都通过Mirror类来实现，我猜测 Mirror就是眼镜上面的那个小块投影仪（也符合Mirror的中文意思），Mirror类中有对 Timeline,Contact,Attachments,Locations,Subscriptions（这些都作为Mirror的内部类存在）的 各种增删改查操作，这些增删改查操作都通过REST服务进行，大致的流程是：首先眼镜将用户的请求包装为一个json格式的数据，通过内置的 httpclient请求google服务器，而后眼镜将返回的json数据解析后返回给用户。比如用户想向Timeline中添加一个item，该 item的内容为“hello world”，我猜测这时候用户说一声“insert“眼镜就会提交一个insert的请求给 AbstractGoogleClientRequest.execute()方法，该方法首先去执行executeUnparsed()方法，接下来就 是正常的httpclient处理了，google提供了多种不同的 client：UrlFetchTransport,ApacheHttpTransport,NetHttpTransport（这几种的区别可查看 HttpTransport源码，推荐使用NetHttpTransport），要执行请求必须得有请求对象在这里就是HttpRequest类，该类里 封装了HttpTransport，所以可以使用该类进行请求。这时候如果你的item中有媒体文件那么会使用MediaHttpUploader类将你 的媒体文件上传到google服务器上，如果没有媒体文件那么就自然的去执行上面获得的HttpRequest.execute()方法，这个方法有将近 250行代码，但是其实很简单，就是将请求后返回的信息封装为HttpResponse对象，最终 AbstractGoogleClientRequest会将该对象封装为json格式数据返回给用户。一个向Timeline插入一个item的过程就 结束了，其实很简单，并没有想的那么复杂。</br></br>
讲了添加，删除修改查找也是一样的，因为每个item都会有一个id 标识（google自动添加的），所以我们可以根据这个id来进行删除和修改，值得注意的是google提供了两种修改的方式：PATCH和PUT，这里 来讲一下REST中的增删改查吧，如果你看过RESTful webservices就可以清楚的知道REST中的各种操作和我们熟知的数据库增删改查是不一样的，REST是面向http进行的，它的POST命令对 应于数据库中的insert，GET对应于select，PUT对应于update，DELETE对应于delete，PATCH对应于update，看 到这里肯定会有疑惑，怎么会有两个不同的update，google讲了，你如果想修改一个对象其中的几个属性，使用PATCH效率会高很多，因为PUT 是将整个对象都修改。查找的时候你可以指定itemID或者不指定，指定的话就是返回特定的item，不指定的话就是返回你的Timeline中的所有 item。</br></br>
对于联系人（Contact），位置信息（Locations），订阅信息（Subscriptions），附件（Attachments）的各种操作和Timeline是一样的，这里就不再讲了。</br></br>
讲 到了联系人，我们可以联想是不是GG还提供了分享功能达到社交的作用呢？确实是的，GG这样做了，比如你的一个item的inReplyTo属性不为空， 那么你的这个item就是inReplyTo的值的回复内容，recipients属性表示你的这个item还有谁分享了，有了这些是不是就可以实现社交 的功能呢？我想是的。</br></br>
关于item的更多属性可以参看TimelineItem类，其中speakableText属性很屌的，我猜这个属性的内容是眼镜可以为你朗读的；etag属性（查看我之前的一条关于etagere的状态）自然就是http端cache的作用了。</br></br>
上 面说到GG会为你朗读，那么如何触发这一事件呢？每一个item都会有一个item menu的菜单栏，这个菜单栏里各种操作，比如分享该item，删除该item，读取该item内容，播放该item的附件等，所以我猜用户如果说 “read aloud“，GG就会去读取item的speakableText内容。当然你也可以让你的联系人赞（pin）与不赞你的item，这个太萌了吧。</br></br>
GG的这种变现形式让我想到了ios和android版的Path应用，该应用也是基于时间轴的概念为用户记录发生了的事情，当然人人网也是有的（有个开启我的时间轴这个功能）。</br></br>
好了，就讲这么多，其中有很多我猜的部分，因为限于硬件方面文档的匮乏，我不得不去猜。