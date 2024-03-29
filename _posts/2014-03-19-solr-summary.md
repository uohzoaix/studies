---
layout: post
title: "solr使用总结"
description: "对solr的各种使用进行总结"
category: 
- solr
tags: []
---



用solr用了蛮久，将网站上的查询数据库环节几乎全部换成查solr索引，可想而知速度的提升是很高的，研究期间也碰到了很多的问题。这里将这些问题和相应处理方法记录下来：</br>
（1）项目搭建：</br>
     	solr下载下来后将其中的一个war包解压缩放入tomcat中，你也可以自己新建一个项目将解压缩之后的相应代码拷贝到WebContent或WebRoot下，接着发布你的项目，修改web.xml文件<env-entry-value>值，该值对应于你的eclipse中的项目地址，如D:/workspace/.metadata/.plugins/org.eclipse.wst.server.core/tmp6/wtpwebapps/solr/home,其实就是的项目发布之后的地址，记住一定是要到home目录为止，因为home目录里面才是你的solr的各个core（core会在下面讲到，其实就是solr查询的地方）。启动后，访问http://127.0.0.1:8080/solr/（根据你实际项目而定）就可以进去solr的控制台了（控制台不细讲，有些需要注意的我会讲到）。</br>
（2）建立索引（db-data-config.xml文件）：</br>
	建立索引时，我们需要配置home目录下各个目录中conf下的db-data-config.xml文件，该文件就是solr建立索引时需要读取的，这个文件首先是配置dataSource,solr是根据你配置的dataSource项去读取数据库，包括type,driver,user,password,url,出于安全考虑，我们可以将user，password，url不在此配置，而将其放入一个properties或xml文件，这时候我们就需要修改DataImporter.readFromXml()方法了，加入读取properties或xml文件的逻辑即可。</br>
	接下来就是配置document项，索引的表现形式就是以document的形式体现的，下面的entity配置项对应的name属性就是你在solr的控制面板上做索引时候Entity的值，我们还需要为每个entity配置pk值（一般是表的主键），它代表你每条索引的唯一性，修改和删除都是以这个pk值来确定删除和修改哪条索引的。下面就是各种query的配置了，solr提供了很多种的操作索引方式，可以重做索引，可以增量索引，可以定时删除索引等，这些都以各种不同的query方式体现。常见的有query，deltaQuery，deletedPkQuery，deltaImportQuery,query很简单，就是简单的sql查询语句，solr第一次做索引的时候会执行这个sql语句将查出来的数据全部做成索引，deltaQuery和deltaImportQuery都是你在增量索引（下文会细讲）的时候用到的，如果你需要定时删除某些索引，你可以使用deletedPkQuery，这个配置的sql语句如：select pkID from xxx where xxx,需要注意的是pkID必须是你在配置entity时的pk值。</br>
	配置完这些之后，现在你只是提供了一些数据库字段，并没有告诉solr到底需要将哪些字段做进索引中，这时你就需要配置<field>，如<field column=”pkID” name=”pkID”>，其中column对应你数据库的字段名，name对应于solr中的字段名（这个在schema.xml文件用到），最好保持一致。在配置filed中，可是使用Transformer来改变你原来数据库字段值的表现形式或者新建一组由原数据库字段组成的新字段。常见的Transformer有RegexTransformer(在我们的项目中用到)，DateFormatTransformer，NumberFormatTransformer，HtmlStripTransformer这些Transformer由字面意思就可以知道它们是干什么的，具体的用法可以见：http://wiki.apache.org/solr/DataImportHandler#Transformer，这里主要讲下RegexTransformer，RegexTransformer是通过正则表达式来生成一个新的字段或修改旧字段，比如你想将一个字段的某个部分抽取出来作为一个单独的字段来查（我们项目里有这样的需求），你就可以将值用正则表达式表示出来并将你需要抽取出来的部分用括号括起来作为一个group，这时候这个group就会被做进索引中。如你想将一个时间字段的小时抽取出来作为单独的字段处理就可以这样写：<field column=”hour” name=”hour” sourceColName=”sourceDate” regex=”\d{4}[-]\d{2}[-]\d{2}\s*(\d{2})[:]\s*\d{2}[:]\s*\d{2}”/>,这时(\d{2})那个部分就作为单独字段做进索引中了。（注意，要使用Transformer,必须在entity配置中添加属性transformer=”XXXTransformer”）。</br>
（3）增量索引：</br>
     	前文说过增量索引是由deltaQuery和deltaImportQuery来实现，solr需要使用定时器来定时执行deltaQuery和deltaImportQuery，solr使用的定时器默认是java的Timer类，但是该类的特点是任务会累加（加大了系统的负载），所以你如果不想让任务累加你可以使用java同步类库中的ScheduledExecutorService来实现。solr使用一个listener来管理定时器，所以你需要在web.xml中配置该listener（ApplicationListener）。deltaQuery的作用是让solr去读取数据库取出哪些行是需要做进索引的，往往这里你的sql语句需要的条件是where updateTime>’${dataimporter.last_index_time}’,这里的dataimporter.last_index_time是solr自己的一个时间，它是你在每次做完索引的那个时间，所以我们很容易的知道增量索引是根据你数据库里的某个时间字段大于上次做完索引的时间来实现增量的。deltaQuery是查出你需要增量索引的的pkID，对于每个pkID都会去执行deltaImportQuery来完成索引的建立，这时候deltaImportQuery中需要的条件是where pkID=’${dataimporter.delta.pkID}’, dataimporter.delta.pkID中的pkID必须和deltaQuery查出来的pkID保持一致。那么solr如何知道我要隔多久去增量呢？这时你可以在home/conf/dataimport.properties文件中配置，其中syncCores是你需要增量的core的名字，它必须和你在solr.xml文件中的<core name=””>项的name值一样，params可以设置为/dataimport?command=delta-import&clean=false&commit=true,interval就是你每次增量的时间间隔，solr默认是以分钟为单位的，如果你需要加大频率，你可以修改ApplicationListener类。如果你需要查看增量索引是否在工作，你可以点击solr控制面板上各个core的dataimport项，如果显示绿色的indexing completed.added/updated:xx(大于0)documents……就说明在工作了。</br>
	在增量索引过程中，因为dataimporter.last_index_time是精确到秒的，所以如果你在一秒之内增量索引完成且这一秒内你又修改了某行数据，那么这一行数据在下一次增量索引中是不会做进去的，因为是updateTime>dataimporter.last_index_time，我们可以改为updateTime>=dataimporter.last_index_time，但是这样就会操作很多重复数据，所以我们可以将updateTime加上一秒，这样updateTime>dataimporter.last_index_time就永远成立了，就不会有数据丢失的情况出现。</br>
注意：</br>
	1.    因为数据库中的updateTime是在应用当中修改的，一般会是应用服务器的当前时间，所以这时候如果你的应用服务器（即你的项目所在服务器）的时间和solr服务器的时间不同步的话那么索引也会出现丢失或重复的现象，如果solr服务器时间大于应用服务器时间则会丢失，如果应用服务器时间大于solr服务器时间则会重复。</br>
	2.    若增量索引的sql语句比较复杂，这时候查询该sql语句会比较消耗数据库资源，很有可能会将正在查询的表进行锁表操作，那么这个时候如果其他的sql语句查询该表时就会一致等待获得锁，这样显然是不对的，所以你必须对增量的sql语句进行优化使得查询尽快释放锁。</br>
     	在增量索引中，由于要多次写deltaQuery和deltaImportQuery，这不免会带来很多重复性的工作，solr还提供了另外一种方法实现增量索引的方法，就是只提供一个query语句就可以实现full-import和delta-import两个效果：将query语句改为select * from table where ‘${dataimporter.request.clean}!=’false’ or updatetime>’${dataimporter.last_index_time}’，这时候如果你是full-import，那么dataimport.request.clean的值默认是true的（在solrconfig.xml文件配置，如果你不确定是否为true，则可以在程序中显式的将clean设为true，query.setParam(“clean”,true)），所以第一个条件是成立的，这时就不需要判断第二个条件了；如果是delta-import，那么由于dataimport.request.clean=false(在home/conf/dataimport.properties文件中的params配置：params=/dataimport?command=delta-import&clean=false&commit=true)，这时第一个条件就不成立了，所以就去判断第二个条件，这时就相当于是增量查找了。
     	很多人就会说这种方法不是更简洁更方便吗？确实是，我一开始也是这么觉得，但是细想如果你的表数据特别多，select * from xxx where xxx这个效率是很低的，会导致你的solr服务器一致在执行这个sql语句，然而之前的做法是先用deltaQuery：select id from xxx where xxx再select * from xxx where id=${dataimporter.delta.id}实现的，先把id全查出来再根据id去查记录这种方式是不是会更好呢？所以简洁不一定就是好方法。</br>
（4）schema.xml文件解析：</br>
     	该文件必须指定索引的唯一字段名<uniqueKey>它往往是你的数据库表的主键，solr是以该值来判断修改和删除的是那一条索引的。你还可以指定<defaultSearchField>它的意思是如果你在搜索时不指定具体字段名时默认就以这里配置的字段名去搜索，这里需要注意的是：如果你指定了搜索字段，但是搜索关键字中有空格时，比如搜索all:aaa bbb，这时solr会把它变为all:aaa text:bbb，这里的text是在solrconfig.xml（下文详细讲）中的<lst name=”defaults”>中的<str name=”df”>配置的值，所以你需要修改这里的值或者将搜索变为all:(aaa bbb),这样搜索就对了。配置<solrQueryParser>是表示在你不指定OR或AND的情况下默认是以这里的配置来搜索。
     	该文件包含了很多的字段类型配置，您可以为你的solr索引字段指定int，string，date等类型，还可以为您的字段指定具体的分词方式，在中文环境下你可以配置IK分词，如果您需要使用相似性查找的话，你需要对IK进行修改（IK默认不提供相似性查询功能），见：IKTokenizerFactory类，这时如果你需要使用相似性你需要配置SynonymFilterFactory并指定synonymous.txt文件的位置，如果你需要使用停止词你需要配置StopFilterFactory并指定stopwords.txt文件的位置，如果你需要使用空格进行分词，那么你可以直接使用solr自带的text_ws即可。</br>
	在你需要搜索部分手机号码的情况下，你可以使用通配符来完成，在查询量比较小的情况下可以使用此方法，但是在查询量比较大的时候你最好不要使用通配符，因为会造成内存回收不了而致使服务器宕机。这时候你可以使用EdgeNGramFilterFactory来完成对同一个手机号码分成多个部分，如13888888888，在minGramSize=6,maxGramSize=11的情况下该手机号码会包含138888，1388888,13888888,138888888,1388888888,13888888888这几个索引，但是这种情况你只能搜索以xxx开头的情况，这时候你可以使用ReversedWildcardFilterFactory+EdgeNGramFilterFactory来完成搜索以xxx开头和以xxx结尾的情况，但是搜索中间的部分（比如搜索8888）还是不能实现（暂时还没有找到solr自带的解决方案），这时你可以修改分词工具，对于数字类型的你可以实现这样的算法：将13888分词为：13,138,1388,13888，38,388,3888,88,888这样的形式，这样怎么搜都能搜出来了，其实也就和solr的通配符的搜索形式一样了。</br>
	如果你搜索时想搜索多个字段匹配搜索内容时你可以配置一个<field name=”all” type=”xxx” indexed=”true” stored=”false”>，在其下加上多个<copyField source=”xxx” dest=”all”>即可，这样搜索all的时候就会搜索copyField里的字段了。</br>
	如果你想让你的搜索结果不按某个字段排序，而是每次刷新结果的排序都是变的（solr默认是以得分排序的，如果结果不变那么最终显示的排序也是不变的），就是类似sqlserver里的order by newid()或mysql里的order by rand()或oracle中的order by dbms_random.value，这时你可以在该文件中加上<dynamicField name=”random_*” type=”random” indexed=”false” stored=”false”/>，但是你必须保证<field name=”random”>的存在，这时你在程序中需要生成一个随机数，然后query.setSortField(“random_”+随机数,SolrQuery.ORDER.desc(这里desc或asc已经无所谓了))。</br>
注意：在该文件中不允许有重复名字的<field>配置出现，否则solr启动就会报错。需要保证有<field name=”text”>的存在，否则solr查询时候会出错。</br>
最佳实践：</br>
	1.    当你只想搜索该字段但是不想让该字段显示那么你可以将该字段的stored设为false，这样可以节省你的硬盘空间，特别是对于大字段来说.</br>
	2.    当你的某些字段不需要搜索而只需要显示出来，你可以将该字段的indexed设为false，这样可以减少你做索引的时间。</br>
	3.    将不需要的copyField删除。</br>
	4.    尽量使用StreamingUpdateSolrServer来搜索索引。</br>
	5.    尽量在服务器模式下运行JVM，并且将日志级别设高以防止每次请求都会打印日志。</br>
	6.    若你需要在多个core中使用synonymous.txt和stopwords.txt时，你可以将这两个文件放在一个公共目录中，然后让每个core去引用这两个文件。</br>
（5）solrconfig.xml文件详解：</br>
     	该文件配置了许多你查询索引时的一些功能。<luceneMatchVersion>必须和你当前使用的solr版本一致。<dataDir>配置了你索引生成的目录。<directoryFactory>配置项指明你使用的那种索引保存方式，默认是NRTCachingDirectoryFactory(near-real-time,即时的)。<autoCommit>配置了是否使用solr的自动提交功能，因为solr的commit方法是很耗时间和资源的，你不能在每次频繁做完索引之后都调用commit方法，这时候你可以配置autoCommit来让solr自己commit，其中maxTime表示过多久commit一次，openSearcher默认为false，不需要改变，maxDocs表示等收集了多少条索引之后commit一次，maxTime和maxDocs只要有一个达到条件了就会commit，在solr4以后的版本中你需要将<autoSoftCommit>配置取消注释，否则自动提交不起作用，在solr4以前的版本中不需要此动作。在solr中提供了多种缓存方式，有filterCache:当查询参数有ids时，solr会去filterCache中取；queryResultCache:如果没有提供ids则会去queryResultCache中取；documentCache:在代码中使用doc(i)时使用；fieldValueCache:在facet查询时使用。这些缓存配置可以使用默认值，也可以按你的应用需求进行修改。autoWarmCount的意思是在每次查询时有多少条旧索引被放入结果列表中。最后你需要加上<requestHandler name=”/dataimport”>项让solr知道做索引应该读取哪个配置文件。</br>
延伸：</br>
	在一般情况下我们做索引都是从数据库读取数据进行dataimport的，但是在没有数据库的情况下，数据只是按固定格式存放在xml文件或txt等文件中的话，并且在真正做索引之前需要对每条document进行修改，这时就需要使用到solr的update请求了，即在solrconfig.xml文件配置的<requestHandler name="/update" class=”solr.UpdateRequestHandler”>，这里的class可以为BinaryUpdateRequestHandler,CSVRequestHandler,JsonUpdateRequestHandler,XmlUpdateRequestHandler,XsltUpdateRequestHandler，这些handler根据名字可以猜测出它们的作用对象是什么，但是这些类都被@Deprecated了，所以你只需要按照默认配置就行。配置好class之后你还需要指定在update请求过程中使用哪个请求处理链，在solrconfig.xml文件中默认配置了<updateRequestProcessorChain name=”dedupe”>这个处理链，在处理链中必须以<processor class=”solr.LogUpdateProcessorFactory”/><processor class=”solr.RunUpdateProcessorFactory”/>结束，否则在运行过程中会出错，由名字可以看出这两个processor是修改log和执行update的（所以可见它们是非常重要的）。而除了这两个在它们之前可以配置其他的processor，processor也可以是我们自定义的（在使用update请求时，一般都是因为有了外部的数据文件才使用，而有了自己的文件自然就需要自己定义processor来处理文件），下面主要是自定义processor的过程：</br>
	①    自定义processor需要继承UpdateRequestProcessorFactory。</br>
	②    重载getInstance(SolrQueryRequest req,SolrQueryResponse resp,UpdateRequestProcessor next)方法，该方法内部只需return new xxxUpdateProcessor(next);当然你可以在其中加入其他的业务逻辑（比如初始化变量工作，当然初始化工作可以通过重载init()方法来实现）。</br>
	③    上述方法返回的是xxxUpdateProcessor，该类也是我们自己定义的，它需要继承UpdateRequestProcessor并提供一个public xxxUpdateProcessor(UpdateRequestProcessor next)构造方法，该方法只需提供super(next);即可。还需要重载processAdd(AddUpdateCommand cmd)方法，该方法就是update索引的主要方法，其中cmd中的solrDoc就是在真正做索引之前由SolrServer.add(SolrInputDocument)之后的那个SolrInputDocument，所以你可以在这时修改SolrInputDocument以达到我们的需求。修改完之后只需调用super.processAdd(cmd)即可完成。</br>
问题：什么情况下需要用到自定义UpdateRequestProcessor？</br>
     	在需要根据指定的值生成另外的值并且需要将生成的值也做进索引则可以使用它来实现。在不需要另外生成值的情况下就不需要使用自定义的processor了，但是内容不是存在数据库而是存在各类文件中时，只需要提供<processor class=”solr.LogUpdateProcessorFactory”/><processor class=”solr.RunUpdateProcessorFactory”/>，并且解析各类文件将每行数据封装为一个SolrInputDocument后SolrServer.add(solrInputDocument)即可。</br>
（6）竞价排名查询：</br>
     	在solrconfig.xml文件的<requestHandler name=”/elevate”>配置中的<lst name=”defaults”>下加上<bool name=”enableElevation”>true</bool><bool name=”forceElevation”>true</bool>以开启竞价功能，然后在conf目录下的elevate.xml文件中加上：
{% highlight objc %}
	<query text=”xxx:xxx”>
		<doc id=”1”/>
		<doc id=”2”/>
	</query>
  {% endhighlight %}
	这样当搜索xxx:xxx（如all:你好）时id为1和2的doc就会显示在前面（这里的id必须是schema.xml文件中配置的<uniqueKey>值）。</br>
	elevate.xml文件可以配置多个竞价规则。</br>
（7）spellcheck查询：</br>
     	有时候我们希望在搜索的关键词拼写错误的时候能让solr给出正确的关键词提示，这时就可以使用solr的spellcheck功能，启用该功能步骤如下：</br>
	1.    找到solrconfig.xml文件中的<requestHandler name=”/spell”>，修改spellcheck.dictionary的值，它的值可以为在<searchComponent name=”spellcheck”>中各个spellchecker中的name属性值。如default，wordbreak，jarowinkler，file等等，由于我们的搜索关键词的专业性，这时就需要另外维护一份文件来生成符合我们需求的索引，所以我们这里使用的是file形式的spellchecker，修改sourceLocation为spellings.txt文件的位置，spellings.txt文件就是每一个拼写正确的关键词。这里我们制定spellcheck.dictionary的值为file，在使用该功能时必须将spellcheck,spellckeck.build分别设为on和true，否则solr不会启用该功能，其他的一些参数可以参看http://wiki.apache.org/solr/SpellCheckComponent进行了解，一般使用默认值即可，需要注意的是spellcheck.count这个属性，它表示最终返回多少个拼写正确的关键词。</br>
	2.    配置好之后，在客户端搜索的时候向后台发送一个/spell查询请求即可，如http://localhost:8080/solr/?qt=/spell&q=关键词。这时如果你的关键词拼写错误，solr就会从spellings.txt文件找到和该关键词类似的并且拼写正确的那些词并返回给QueryResponse，这时可以通过QueryResponse的getSpellCheckResponse方法获取到具体的拼写正确的词，代码如下：
  {% highlight objc %}
SolrQuery checkQuery=new SolrQuery();
String checkQuery=SolrServerUtil.getSpellQuery(keyWord);
QueryResponse spellrsp = solrServer.query(checkQuery); // 提交查询条件
SpellCheckResponse checkRsp=spellrsp.getSpellCheckResponse();
List<String> checks=null;
if(checkRsp!=null){
    List<Suggestion> suggestions=checkRsp.getSuggestions();
    if(suggestions.size()>0&&null!=suggestions){
        checks=new ArrayList<String>();
        for(Suggestion suggestion:suggestions){
            checks=suggestion.getAlternatives();
        }
    }
}
return checks;
{% endhighlight %}
     	返回的checks数组就是拼写正确的词的一个列表。</br>
（8）最相似查询（MoreLikeThis）功能：</br>
     	在该功能中需要将mlt参数设置为true以启用该功能，并且需要设置相似功能用在哪一个字段上，通过设置mlt.fl可以指定相应的字段使用该功能。mlt.mintf表示搜索的关键词在某条索引中的出现次数小于该值的将忽略，mlt.mindf表示搜索的关键词在整个文档索引中出现次数小于该值的将忽略。mlt.count表示需要返回多少条相似记录。查询时可以：http://localhost:8080/solr/select?q=关键词&mlt=true&mlt.fl=title&mlt.mindf=1&mlt.mintf=1&fl=*,score，这时solr会返回title中和关键词相似的那些记录。获取最终记录的代码如下：
{% highlight objc %}
SolrQuery mltQuery=new SolrQuery();
String mltQuery=SolrServerUtil.mlt(keyWord);
QueryResponse mltrsp = solrServer.query(mltQuery); // 提交查询条件
SolrDocumentList docs=mltrsp.getResults();
SimpleOrderedMap<Object> mlts=(SimpleOrderedMap<Object>)mltrsp.getResponse().get("moreLikeThis");
Map<String, String> suggests=null;
if(docs.size()>0){
    for (SolrDocument doc : docs) {
        String sellID = doc.getFieldValue("sellID").toString();
        SolrDocumentList mltdocs = (SolrDocumentList)mlts.get(sellID);
     	  if(mltdocs.size()>0){
            suggests=new HashMap<String, String>();
        	for (SolrDocument mltdoc : mltdocs) {
        	   suggests.put(mltdoc.getFieldValue("sellID").toString(),mltdoc.getFieldValue("title").toString());
            }
        }
    }
}
return suggests;
{% endhighlight %}
（9）facet查询：</br>
     	Facet查询只针对indexed=true的字段而不是stored=true，facet查询其实就是sql里的group by功能，说到group by，我们就必须为group by提供所要group的字段，比如group by corpID，在solr中可以通过query.addFacetField(“corpID”)实现，在使用facet时必须将facet功能开启（默认关闭）：query.setFacet(true)，其他的一些参数设置如：query.setRows(0)设置返回结果条数，在facet中需要设置为0，query.setFacetSort(“count”)设置返回的结果列表按数量高低排序，query.setFacetLimit(pageSize)设置每次返回的分组数量可用于分页，query.set(FacetParams.FACET_OFFSET,(page==null?0:page-1)*pageSize)设置分组起始位置。如果你需要显示分组后总共有多少组，则需要使用query.setFacetLimit(Integer.MAX_VALUE)来不限制返回的分组数，这样就可以通过SolrServer.query(query).getFacetField(“corpID”).getValues().size()来取得总共的分组数量。但是在分页时我们需要如下操作来存储结果：
{% highlight objc %}	
QueryResponse resultRsp=solrServer.query(query);
List<Count> countList = resultRsp.getFacetField("corpID").getValues();
Map<Sell, Integer> rmap = new LinkedHashMap<Sell, Integer>();
for (Count resultcount : countList) {
    if (resultcount.getCount() > 0) {//查出该组中的一个供应产品以显示在页面上
        Sell sell = expand(其他参数…, resultcount.getName()).get(0);
        rmap.put(sell, (int) resultcount.getCount());
    }
}
return rmap;
{% endhighlight %}
	值得注意的是expand方法，该方法的目的是从每组结果中获取第一个对象以显示在页面上，resultCount.getName()就是获取该组的corpID值，再去使用该corpID到solr中去查出该corpID的所有信息并get(0)获取第一个对象，resultCount.getCount()是获取该组的数量。通过这样操作之后就可以实现分组效果并在每组下显示诸如“该组共有多少个供求信息”的提示信息了。
	到这里还没完，我们还需要点击提示信息的时候显示出该组的全部供求信息，当然这个就比较简单了，直接拿corpID到solr中去查并封装结果列表返回即可。</br>
solr的fieldType初始化流程：</br>
Lucene41PostingReader类中的docDeltaBuffer属性用来存储每个term在所有doc中的偏移量，比如“价格”这个词在doc1，3，4，7，8，9这几个文档中出现，那么该属性的值为0，2，1，3，1，1.该值由方法readVIntBlock获得：
{% highlight objc %}
static void readVIntBlock(IndexInput docIn, int[] docBuffer, int[] freqBuffer, int num, boolean indexHasFreq) throws IOException {
    if (indexHasFreq) {
        for(int i=0;i<num;i++) {
            final int code = docIn.readVInt();
            docBuffer[i] = code >>> 1;
            if ((code & 1) != 0) {
                freqBuffer[i] = 1;
            } else {
                freqBuffer[i] = docIn.readVInt();
            }
        }
    } else {
        for(int i=0;i<num;i++) {
            docBuffer[i] = docIn.readVInt();
        }
    }
}
{% endhighlight %}
这里的docIn参数指示的是.doc文件，该文件存储的是term存在于哪些doc中，其格式如：
价格：1 3 4 7 8 9，这里的num就为6，每次docIn.readVInt()就会读取到1，3，4，7，8，9.</br>

solr spatial:</br>
latlontype:http://localhost:9080/solr/factory_article/select?q=*:*&fq={!geofilt}&sfield=geohash&pt=11.6485,5.3358&d=500&&score=recipDistance&sort=geodist%28%29%20asc&fl=*,score,_dist_:geodist%28%29</br>
solr.SpatialRecursivePrefixTreeFieldType:http://localhost:9080/solr/factory_article/select?&fl=*,score&sort=score%20asc&q={!%20score=distance}geohash:%22Intersects%28Circle%2811.6485,5.3358%20d=100%29%29%22</br>
