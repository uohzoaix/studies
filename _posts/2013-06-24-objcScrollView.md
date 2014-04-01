---
layout: post
title: "objc UIScrollView和UIPageControl使用"
category: 
- objc
tags: []
---














很多的app有内容图片等滚动功能，即使用手指滑动屏幕图片或者文章即可翻页浏览，这种功能就需要这两个组件，要实现这 样的功能，首先就需要布局这两个组件在屏幕上的位置，一般情况下浏览的内容即UIScrollView在上面，而滚动条（在苹果中一般为一个个小圆点，当 前页对应的小圆点会被点亮）则位于屏幕下方，所以位置的布局往往就是对屏幕坐标的计算，通过计算内容的高度来确定UIScrollView和 UIPageControl的x,y坐标值和它们各自的宽高。位置确定完之后需要对UIScrollView的各种属性进行设值，必须设置的属性有 pagingEnabled（开启分页），contentSize（滚动的平面面积），delegate（代理，以后文章补充），具体的值可以参考网上的 例子进行了解（这篇文章主要内容不是如何实现翻页，而是记录一些网上例子没有的或者网上例子错误的一些功能），接着需要设置UIPageControl的 一些属性，比如numberOfPages（有几页），currentPage（默认显示第几页）等，在这里网上的例子currentPage基本上都设 置为0即第一页，可是我想在开启app后显示中间的那一页，我简单的使用 pageControl.currentPage=floor(numberOfPages/2+0.5);并将中间那页的内容页加到了 UIScrollView中，可是运行后看到的是一个白板没有任何东西，这时我把每一页的内容都加到UIScrollView中，这时我看到的是第一页的 内容，为什么显示的是第一页的内容？我可以确定问题的源头肯定是在UIScrollView中，和UIPageControl肯定是没有关系的，因为 UIScrollView负责显示，而UIPageControl只负责翻页，即UIPageControl发出一个指令让UIScrollView显示 具体的哪一页的信息，进入UIScrollView的源码，还好它的属性不多，一个一个找，发现一个Offset结尾的属性contentOffset即 内容偏移，后面的注视“default CGPointZero”让我更加的坚信应该就是这个属性决定了ScrollView的初始位移，于是将ScrollView的初始位移设置成中间那张图 片的位置：
{% highlight objc %}
CGPoint pt = CGPointMake(pageControl.currentPage*320, 0);
[scrollView setContentOffset:pt];
{% endhighlight %}
重新运行之后默认显示的就是中间的那张图片了，并且左右滑动也可以进行切换了。</br></br>
到 这里还没完，我还想实现那种自动滑动的效果，类似图片广告的那种效果，自然而然就会想到使用定时器（web页面也基本上使用什么 interval，timeout等实现的），网上搜定时器相关的东西，只需要使用：
{% highlight objc %}
[NSTimer scheduledTimerWithTimeInterval:2 target:self selector:@selector(noStopping) userInfo:nil repeats:YES];
{% endhighlight %}
就 可实现定时处理noStopping的功能，需要自己实现的是noStopping方法，在我们这个例子中，是这样的一种情况：当 pageControl.currentPage为图片数目即滑到了最后一张图片时，我们需要将pageControl.currentPage的值设为 0即让它显示第一张图片，这时我们还是要重新设置setContentOffset改变ScrollView的位移。这样就实现了定时无限滑动的功能了：
{% highlight objc %}
-(void)noStopping{
    BACK=(pageControl.currentPage==pageControl.numberOfPages-1)?YES:NO;//是否需要回滚
    ++pageControl.currentPage;//在不需要回滚时page+1
    pageControl.currentPage= BACK?0:pageControl.currentPage;
    [scrollView setContentOffset:CGPointMake(pageControl.currentPage*scrollView.frame.size.width, 0) animated:YES];
}
{% endhighlight %}