---
layout: post
title: "solr之Slop查询分析"
description: "solr的slop查询代码分析"
category: essay
tags: []
---



在查询如：title:"市场 价格"~5的时候，意思是title包括市场和价格并且市场和价格之前的距离不能大于5.如果查询title:"市场 价格 市场"~5，说明市场后面有价格，价格后面还要有市场，否则没有结果。</br></br>
主要代码如下：</br></br>
{% highlight objc %}
protected void search(List<AtomicReaderContext> leaves, Weight weight, Collector collector) throws IOException {
    // TODO: should we make this threaded...?  the Collector could be sync'd?
    // always use single thread:
    for (AtomicReaderContext ctx : leaves) { // search each subreader
        collector.setNextReader(ctx);
        Scorer scorer = weight.scorer(ctx, !collector.acceptsDocsOutOfOrder(), true, ctx.reader().getLiveDocs());
        if (scorer != null) {
            scorer.score(collector);
        }
    }
}
{% endhighlight %}</br>
进入PhraseWeight.score()方法：
{% highlight objc %}
public Scorer scorer(AtomicReaderContext context, boolean scoreDocsInOrder,boolean topScorer, Bits acceptDocs) throws IOException {
    assert !terms.isEmpty();
    final AtomicReader reader = context.reader();
    final Bits liveDocs = acceptDocs;
    PostingsAndFreq[] postingsFreqs = new PostingsAndFreq[terms.size()];
    final Terms fieldTerms = reader.terms(field);
    if (fieldTerms == null) {
        return null;
    }
    // Reuse single TermsEnum below:
    final TermsEnum te = fieldTerms.iterator(null);
    for (int i = 0; i < terms.size(); i++) {
        final Term t = terms.get(i);
        //这里的state是根据该term是否在索引中存在生成的
        final TermState state = states[i].get(context.ord);
        //如果该term在索引中不存在，则会return null，说明在该种查询语法中每个term都必须出现，即使AND的关系
        if (state == null) { /* term doesnt exist in this segment */
            assert termNotInReader(reader, t): "no termstate found but term exists in reader";
            return null;
        }
        te.seekExact(t.bytes(), state);//这里是具体的在索引文件里进行搜索的逻辑
        DocsAndPositionsEnum postingsEnum = te.docsAndPositions(liveDocs, null, DocsEnum.FLAG_NONE);
        // PhraseQuery on a field that did not index
        // positions.
        if (postingsEnum == null) {
            assert te.seekExact(t.bytes(), false) : "termstate found but no term exists in reader";
            // term does exist, but has no positions
            throw new IllegalStateException("field \"" + t.field() + "\" was indexed without position data; cannot run PhraseQuery (term=" + t.text() + ")");
        }
        postingsFreqs[i] = new PostingsAndFreq(postingsEnum, te.docFreq(), positions.get(i).intValue(), t);
    }
    // sort by increasing docFreq order
    if (slop == 0) {
        ArrayUtil.mergeSort(postingsFreqs);
    }
    //如果slop为0，即查询语句中的~后面的数字，则返回ExactPhraseScorer，否则返回SloppyPhraseScorer
    if (slop == 0) {  // optimize exact case
        ExactPhraseScorer s = new ExactPhraseScorer(this, postingsFreqs, similarity.exactSimScorer(stats, context));
        if (s.noDocs) {
            return null;
        } else {
            return s;
        }
    } else {
        return new SloppyPhraseScorer(this, postingsFreqs, slop, similarity.sloppySimScorer(stats, context));
    }
}
{% endhighlight %}</br>
接着进入Scorer.score()方法中：
{% highlight objc %}
/** Scores and collects all matching documents.
   * @param collector The collector to which all matching documents are passed.
   */
public void score(Collector collector) throws IOException {
    collector.setScorer(this);
    int doc;
    while ((doc = nextDoc()) != NO_MORE_DOCS) {
        collector.collect(doc);
    }
}
在这里主要是nextDoc()方法，进入SloppyPhraseScorer.nextDoc()方法：
public int nextDoc() throws IOException {
    return advance(max.doc);
}
public int advance(int target) throws IOException {
    sloppyFreq = 0.0f;
    if (!advanceMin(target)) {
        return NO_MORE_DOCS;
    } 
    boolean restart=false;
    while (sloppyFreq == 0.0f) {
        while (min.doc < max.doc || restart) {
            restart = false;
            if (!advanceMin(max.doc)) {
                return NO_MORE_DOCS;
            }
        }
        // found a doc with all of the terms
        sloppyFreq = phraseFreq(); // check for phrase
        restart = true;
    }
    // found a match
    return max.doc;
}
private float phraseFreq() throws IOException {
    if (!initPhrasePositions()) {
        return 0.0f;
    }
    float freq = 0.0f;
    numMatches = 0;
    PhrasePositions pp = pq.pop();
    System.out.println(pp+"---"+end);
    int matchLength = end - pp.position;
    int next = pq.top().position;
    while (advancePP(pp)) {
        if (hasRpts && !advanceRpts(pp)) {
            break; // pps exhausted
        }
        if (pp.position > next) { // done minimizing current match-length
            if (matchLength <= slop) {
                  freq += docScorer.computeSlopFactor(matchLength); // score match
                  numMatches++;
            } 
            pq.add(pp);
            pp = pq.pop();
            next = pq.top().position;
            matchLength = end - pp.position;
        } else {
            int matchLength2 = end - pp.position;
            if (matchLength2 < matchLength) {
                matchLength = matchLength2;
            }
        }
    }
    if (matchLength <= slop) {
        freq += docScorer.computeSlopFactor(matchLength); // score match
        numMatches++;
    }  
    return freq;
}
{% endhighlight %}
在这个方法可以看到matchLength和slop进行比较了。pq的heap（父类的属性）里面存放的是每个term所在field中的位置和该field属于哪个文档的信息：如heap[0]=d:0 o:0 p:7 c:0表示offset为0的term在dicid为0的文档中的当前查询field中的第7个位置，在该field中该term剩余count数量为0，即freq的大小,heap[1]=d:0 o:1 p:11 c:0。
