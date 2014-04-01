---
layout: post
title: "objc UINavigationController使用心得"
category: 
- objc
tags: []
---
















在objc开发当中，第一个主要的工作就是画图，这个过程包括图片的渲染，元素位置的调整，各种效果的填充，这些工作只要花点时间就很容易完成的， 没什么难度；还有一个重要的工作就是UINavigationController的运用，在各种APP中都会存在多个视图之间的切换，这种切换要说容易 其实真的是很简单的，你几乎可以用一行代码来搞定，就是使用它的popViewControllerAnimated方法，因为 UINavigationController会保存你从打开应用之后看到过的所有ViewController，比如你要在当前视图返回之前的视图你就 可以调用这个方法，UINavigationController知道到底要显示哪个ViewController，这对于一步一梯式的APP工作模式来 说已经足够了，但是就是有些APP会存在一步多梯的情况，我就遇到过，没遇到过也就不会有这篇文章的出现，所谓一步多梯就是点一下返回按钮我要跨越中间一 个或多个的ViewController来显示指定的ViewController，这里来举一个实际例子：在微博详细页面点击添加评论并对微博评论完之 后点击确定这时显示的ViewController应该是刚才这条微博的详细页面，但是这时再点击返回的话如果使用 popViewControllerAnimated这时会显示刚才那个评论页面，这显然不是我要的效果，我想要的应该是到首页或者其他的页面反正不会是 评论页面，图示如下：</br></br>
weiboListViewController—点击某条微博— >weiboDetailViewController—点击评论—>addCommentViewController—点击保存— >弹出对话框:评论成功—点击确定—> weiboDetailViewController，好了，到这里我们来看看UINavigationController中是些什么东西，它们是 weiboListViewController，weiboDetailViewController，addCommentViewController，weiboDetailViewController， 这时点击返回的话使用popViewControllerAnimated方法就会显示addCommentViewController，而我是想显示 weiboListViewController，这样是不是一目了然了。</br></br>
要避免这种情况就要在 weiboDetailViewController进行一些特殊处理了，当点击weiboDetailViewController的返回键时我们通过 UINavigationController的viewControllers属性获取所有ViewController，这时我们循环这些 ViewController判断当前循环的ViewController是否为weiboListViewController如果是我们将 viewControllers属性置为NULL(这步很重要，否则将报错)，然后将当前循环的ViewController push到UINavigationController中即可。</br></br>
讲了这么多其实都是废话，这种方法仅仅是我在没有发现简便的方法之前实现 的，后来发现API中有一个popToViewController方法，顾名思义，它就是为这种情况而生的。为什么要写出上面那堆废话，就是为了让我们 在写程序的时候多想想，什么问题都会有办法解决的。</br></br>
当然，还可以换一种角度思考，为什么不能在微博详细页面点击评论弹出一个对话框让用户输 入进行评论，评论完将该对话框删除呢？这样UINavigationController中就会只存在weiboListViewController和 weiboDetailViewController就不会存在那种问题了。据我实验这是完全可行的，而且这种效果我觉得会更加的人性化：让用户做更少的 事来达到更多的目的。