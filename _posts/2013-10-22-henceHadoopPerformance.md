---
layout: post
title: "MapReduce: 提高MapReduce性能的七点建议(转)"
category: 
- hadoop
- 调优
tags: []
---





Cloudera提供给客户的服务内容之一就是调整和优化MapReduce job执行性能。MapReduce和HDFS组成一个复杂的分布式系统，并且它们运行着各式各样用户的代码，这样导致没有一个快速有效的规则来实现优化代码性能的目的。在我看来，调整cluster或job的运行更像一个医生对待病人一样，找出关键的“症状”，对于不同的症状有不同的诊断和处理方式。</br></br> 
在医学领域，没有什么可以代替一位经验丰富的医生；在复杂的分布式系统上，这个道理依然正确—有经验的用户和操作者在面对很多常见问题上都会有“第六感”。我曾经为Cloudera不同行业的客户解决过问题，他们面对的工作量、数据集和cluster硬件有很大区别，因此我在这方面积累了很多的经验，并且想把这些经验分享给诸位。 </br></br>
在这篇blog里，我会高亮那些提高MapReduce性能的建议。前面的一些建议是面向整个cluster的，这可能会对cluster操作者和开发者有帮助。后面一部分建议是为那些用Java编写MapReduce job的开发者而提出。在每一个建议中，我列出一些“症状”或是“诊断测试”来说明一些针对这些问题的改进措施，可能会对你有所帮助。 </br></br>
请注意，这些建议中包含很多我以往从各种不同场景下总结出来的直观经验。它们可能不太适用于你所面对的特殊的工作量、数据集或cluster，如果你想使用它，就需要测试使用前和使用后它在你的cluster环境中的表现。对于这些建议，我会展示一些对比性的数据，数据产生的环境是一个4个节点的cluster来运行40GB的Wordcount job。应用了我以下所提到的这些建议后，这个job中的每个map task大概运行33秒，job总共执行了差不多8分30秒。 </br></br>
##第一点  正确地配置你的Cluster 
###诊断结果/症状： 
1. Linux top命令的结果显示slave节点在所有map和reduce </br>slot都有task运行时依然很空闲。 </br>
2. top命令显示内核的进程，如RAID(mdX_raid*)或pdflush占去大量的CPU时间。 </br>
3. Linux的平均负载通常是系统CPU数量的2倍。 </br>
4. 即使系统正在运行job，Linux平均负载总是保持在系统CPU数量的一半的状态。 </br>
5. 一些节点上的swap利用率超过几MB </br>
优化你的MapReduce性能的第一步是确保你整个cluster的配置文件被调整过。对于新手，请参考这里关于配置参数的一篇blog：配置参数。 除了这些配置参数 ，在你想修改job参数以期提高性能时，你应该参照下我这里的一些你应该注意的项： </br></br>
1.  确保你正在DFS和MapReduce中使用的存储mount被设置了noatime选项。这项如果设置就不会启动对磁盘访问时间的记录，会显著提高IO的性能。 </br>
2. 避免在TaskTracker和DataNode的机器上执行RAID和LVM操作，这通常会降低性能 </br>
3. 在这两个参数mapred.local.dir和dfs.data.dir 配置的值应当是分布在各个磁盘上目录，这样可以充分利用节点的IO读写能力。运行 Linux sysstat包下的iostat -dx 5命令可以让每个磁盘都显示它的利用率。 </br>
4. 你应该有一个聪明的监控系统来监控磁盘设备的健康状态。MapReduce job的设计是可容忍磁盘失败，但磁盘的异常会导致一些task重复执行而使性能下降。如果你发现在某个TaskTracker被很多job中列入黑名单，那么它就可能有问题。 </br>
5. 使用像Ganglia这样的工具监控并绘出swap和网络的利用率图。如果你从监控的图看出机器正在使用swap内存，那么减少mapred.child.java.opts属性所表示的内存分配。</br></br>

##第二点  使用LZO压缩 
###诊断结果/症状： 
1. 对 job的中间结果数据使用压缩是很好的想法。 </br>
2. MapReduce job的输出数据大小是不可忽略的。 </br>
3. 在job运行时，通过linux top 和 iostat命令可以看出slave节点的iowait利用率很高。 </br>
几乎每个Hadoop job都可以通过对map task输出的中间数据做LZO压缩获得较好的空间效益。尽管LZO压缩会增加一些CPU的负载，但在shuffle过程中会减少磁盘IO的数据量，总体上总是可以节省时间的。 </br></br>
当一个job需要输出大量数据时，应用LZO压缩可以提高输出端的输出性能。这是因为默认情况下每个文件的输出都会保存3个幅本，1GB的输出文件你将要保存3GB的磁盘数据，当采用压缩后当然更能节省空间并提高性能。 </br></br>
为了使LZO压缩有效，请设置参数mapred.compress.map.output值为true。 </br></br>

###基准测试： 
在我的cluster里，Wordcount例子中不使用LZO压缩的话，job的运行时间只是稍微增加。但FILE_BYTES_WRITTEN计数器却从3.5GB增长到9.2GB，这表示压缩会减少62%的磁盘IO。在我的cluster里，每个数据节点上磁盘数量对task数量的比例很高，但Wordcount job并没有在整个cluster中共享，所以cluster中IO不是瓶颈，磁盘IO增长不会有什么大的问题。但对于磁盘因很多并发活动而受限的环境来说，磁盘IO减少60%可以大幅提高job的执行速度。 </br></br>

##第三点  调整map和reduce task的数量到合适的值 
###诊断结果/症状： 
1. 每个map或reduce task的完成时间少于30到40秒。 </br>
2. 大型的job不能完全利用cluster中所有空闲的slot。 </br>
3. 大多数map或reduce task被调度执行了，但有一到两个task还在准备状态，在其它task完成之后才单独执行 。</br></br>
调整job中map和reduce task的数量是一件很重要且常常被忽略的事情。下面是我在设置这些参数时的一些直观经验：</br> 
1. 如果每个task的执行时间少于30到40秒，就减少task的数量。Task的创建与调度一般耗费几秒的时间，如果task完成的很快，我们就是在浪费时间。同时，设置JVM重用也可以解决这个问题。 </br>
2. 如果一个job的输入数据大于1TB，我们就增加block size到256或者512，这样可以减少task的数量。你可以使用这个命令去修改已存在文件的block size: hadoop distcp -Ddfs.block.size=$[256*1024*1024] /path/to/inputdata  /path/to/inputdata-with/largeblocks。在执行完这个命令后，你就可以删除原始的输入文件了(/path/to/inputdata)。 </br>
3. 只要每个task运行至少30到40秒，那么就增加map task的数量，增加到整个cluster上map slot总数的几倍。如果你的cluster中有100个map slot，那就避免运行一个有101个map task的job — 如果运行的话，前100个map同时执行，第101个task会在reduce执行之前单独运行。这个建议对于小型cluste和小型job是很重要的。</br>
4. 不要调度太多的reduce task — 对于大多数job来说，我们推荐reduce task的数量应当等于或是略小于cluster中reduce slot的数量。 
###基准测试： 
为了让Wordcount job有很多的task运行，我设置了如下的参数：Dmapred.max.split.size=$[16*1024*1024]。以前默认会产生360个map task，现在就会有2640个。当完成这个设置之后，每个task执行耗费9秒，并且在JobTracker的Cluster Summar视图中可以观看到，正在运行的map task数量在0到24之间浮动。job在17分52秒之后结束，比原来的执行要慢两倍多。 </br></br>

##第四点  为job添加一个Combiner 
###诊断结果/症状： 
1. job在执行分类的聚合时，REDUCE_INPUT_GROUPS计数器远小于REDUCE_INPUT_RECORDS计数器。 </br>
2. job执行一个大的shuffle任务(例如，map的输出数据每个节点就是好几个GB)。 </br>
3. 从job计数器中看出，SPILLED_RECORDS远大于MAP_OUTPUT_RECORDS。 </br>
如果你的算法涉及到一些分类的聚合，那么你就可以使用Combiner来完成数据到达reduce端之前的初始聚合工作。MapReduce框架很明智地运用Combiner来减少写入磁盘以及通过网络传输到reduce端的数据量。

###基准测试： 
我删去Wordcount例子中对setCombinerClass方法的调用。仅这个修改就让map task的平均运行时间由33秒增长到48秒，shuffle的数据量也从1GB提高到1.4GB。整个job的运行时间由原来的8分30秒变成15分42秒，差不多慢了两倍。这次测试过程中开启了map输出结果的压缩功能，如果没有开启这个压缩功能的话，那么Combiner的影响就会变得更加明显。 </br></br>

##第五点  为你的数据使用最合适和简洁的Writable类型 
###诊断/症状： 
1. Text 对象在非文本或混合数据中使用。 </br>
2. 大部分的输出值很小的时候使用IntWritable 或 LongWritable对象。 </br>
当一个开发者是初次编写MapReduce，或是从开发Hadoop Streaming转到Java MapReduce，他们会经常在不必要的时候使用Text 对象。尽管Text对象使用起来很方便，但它在由数值转换到文本或是由UTF8字符串转换到文本时都是低效的，且会消耗大量的CPU时间。当处理那些非文本的数据时，可以使用二进制的Writable类型，如IntWritable， FloatWritable等。 </br></br>
除了避免文件转换的消耗外，二进制Writable类型作为中间结果时会占用更少的空间。当磁盘IO和网络传输成为大型job所遇到的瓶颈时，减少些中间结果的大小可以获得更好的性能。在处理整形数值时，有时使用VIntWritable或VLongWritable类型可能会更快些—这些实现了变长整形编码的类型在序列化小数值时会更节省空间。例如，整数4会被序列化成单字节，而整数10000会被序列化成两个字节。这些变长类型用在统计等任务时更加有效，在这些任务中我们只要确保大部分的记录都是一个很小的值，这样值就可以匹配一或两个字节。 </br></br>
如果Hadoop自带的Writable类型不能满足你的需求，你可以开发自己的Writable类型。这应该是挺简单的，可能会在处理文本方面更快些。如果你编写了自己的Writable类型，请务必提供一个RawComparator类—你可以以内置的Writable类型做为例子。 

###基准测试： 
对于Wordcount例子，我修改了它在map计数时的中间变量，由IntWritable改为Text。并且在reduce统计最终和时使用Integer.parseString(value.toString)来转换出真正的数值。这个版本比原始版本要慢近10%—整个job完成差不多超过9分钟，且每个map task要运行36秒，比之前的33秒要慢。尽量看起来整形转换还是挺快的，但这不说明什么情况。在正常情况下，我曾经看到过选用合适的Writable类型可以有2到3倍的性能提升的例子。 

##第六点  重用Writable类型 
###诊断/症状： 
1. 在mapred.child.java.opts参数上增加-verbose:gc -XX:+PriintGCDetails，然后查看一些task的日志。如果垃圾回收频繁工作且消耗一些时间，你需要注意那些无用的对象。</br> 
2. 在你的代码中搜索"new Text" 或"new IntWritable"。如果它们出现在一个内部循环或是map/reduce方法的内部时，这条建议可能会很有用。 </br>
3. 这条建议在task内存受限的情况下特别有用。 </br>
很多MapReduce用户常犯的一个错误是，在一个map/reduce方法中为每个输出都创建Writable对象。例如，你的Wordcout mapper方法可能这样写： 
{% highlight objc %}
public void map(...) {  
	…  
	for (String word : words) {  
 		output.collect(new Text(word), new IntWritable(1));  
	}  
 }  
{% endhighlight %}
这样会导致程序分配出成千上万个短周期的对象。Java垃圾收集器就要为此做很多的工作。更有效的写法是： 
{% highlight objc %}
class MyMapper … {  
 	Text wordText = new Text();  
	IntWritable one = new IntWritable(1);  
 	public void map(...) {  
 		for (String word: words) {  
			wordText.set(word);  
 			output.collect(wordText, one);  
 		}  
 	}  
 }  
{% endhighlight %}
###基准测试： 
当我以上面的描述修改了Wordcount例子后，起初我发现job运行时与修改之前没有任何不同。这是因为在我的cluster中默认为每个task都分配一个1GB的堆大小 ，所以垃圾回收机制没有启动。当我重新设置参数，为每个task只分配200MB的堆时，没有重用Writable对象的这个版本执行出现了很严重的减缓 —job的执行时间由以前的大概8分30秒变成现在的超过17分钟。原始的那个重用Writable的版本，在设置更小的堆时还是保持相同的执行速度。因此重用Writable是一个很简单的问题修正，我推荐大家总是这样做。它可能不会在每个job的执行中获得很好的性能，但当你的task有内存限制时就会有相当大的区别。 </br></br>

##第七点  使用简易的剖析方式查看task的运行 
这是我在查看MapReduce job性能问题时常用的一个小技巧。那些不希望这些做的人就会反对说这样是行不通的，但是事实是摆在面前。 </br></br>
为了实现简易的剖析，可以当job中一些task运行很慢时，用ssh工具连接上task所在的那台task tracker机器。执行5到10次这个简单的命令 sudo killall -QUIT java(每次执行间隔几秒)。别担心，不要被命令的名字吓着，它不会导致任何东西退出。然后使用JobTracker的界面跳转到那台机器上某个task的stdout 文件上，或者查看正在运行的机器上/var/log/hadoop/userlogs/目录中那个task的stdout文件。你就可以看到当你执行那段命令时，命令发送到JVM的SIGQUIT信号而产生的栈追踪信息的dump文件。([译]在JobTracker的界面上有Cluster Summary的表格，进入Nodes链接，选中你执行上面命令的server，在界面的最下方有Local Logs,点击LOG进入，然后选择userlogs目录，这里可以看到以server执行过的jobID命名的几个目录，不管进入哪个目录都可以看到很多task的列表，每个task的log中有个stdout文件，如果这个文件不为空，那么这个文件就是作者所说的栈信息文件) </br></br>
解析处理这个输出文件需要一点点以经验，这里我介绍下平时是怎样处理的： </br>
对于栈信息中的每个线程，很快地查找你的java包的名字(假如是com.mycompany.mrjobs)。如果你当前线程的栈信息中没有找到任何与你的代码有关的信息，那么跳到另外的线程再看。</br></br> 
如果你在某些栈信息中看到你查找的代码，很快地查阅并大概记下它在做什么事。假如你看到一些与NumberFormat相关的信息，那么此时你需要记下它，暂时不需要关注它是代码的哪些行。 </br></br>转到日志中的下一个dump，然后也花一些时间做类似的事情然后记下些你关注的内容。 在查阅了4到5个栈信息后，你可能会意识到在每次查阅时都会有一些似曾相识的东西。如果这些你意识到的问题是阻碍你的程序变快的原因，那么你可能就找到了程序真正的问题。假如你取到10个线程的栈信息，然后从5个里面看到过NumberFormat类似的信息，那么可能意味着你将50%的CPU浪费在数据格式转换的事情上了。 </br></br>
当然，这没有你使用真正的分析程序那么科学。但我发现这是一种有效的方法，可以在不需要引入其它工具的时候发现那些明显的CPU瓶颈。更重要的是，这是一种让你会变的更强的技术，你会在实践中知道一个正常的和有问题的dump是啥样子。通过这项技术我发现了一些通常出现在性能调优方面的误解，列出在下面。 </br>
1. NumberFormat 相当慢，尽量避免使用它。 </br>
2. String.split—不管是编码或是解码UTF8的字符串都是慢的超出你的想像— 参照上面提到的建议，使用合适的Writable类型。 </br>
3. 使用StringBuffer.append来连接字符串 </br>