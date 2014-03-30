---
layout: post
title: "solr之Slop查询分析"
description: "solr的slop查询代码分析"
category: essay
tags: []
---



在查询如：title:"市场 价格"~5的时候，意思是title包括市场和价格并且市场和价格之前的距离不能大于5.如果查询title:"市场 价格 市场"~5，说明市场后面有价格，价格后面还要有市场，否则没有结果。</br></br>
主要代码如下：</br></br>
protected void search(List<AtomicReaderContext> leaves, Weight weight, Collector collector)</br>
      throws IOException {</br>
    // TODO: should we make this threaded...?  the Collector could be sync'd?
    // always use single thread:</br>
    for (AtomicReaderContext ctx : leaves) { // search each subreader</br>
      collector.setNextReader(ctx);</br>
      Scorer scorer = weight.scorer(ctx, !collector.acceptsDocsOutOfOrder(), true, ctx.reader().getLiveDocs());</br>
      if (scorer != null) {</br>
        scorer.score(collector);</br>
      }</br>
    }</br>
}</br></br>
进入PhraseWeight.score()方法：</br></br>
public Scorer scorer(AtomicReaderContext context, boolean scoreDocsInOrder,
        boolean topScorer, Bits acceptDocs) throws IOException {</br>
      assert !terms.isEmpty();</br>
      final AtomicReader reader = context.reader();</br>
      final Bits liveDocs = acceptDocs;</br>
      PostingsAndFreq[] postingsFreqs = new PostingsAndFreq[terms.size()];</br>
      final Terms fieldTerms = reader.terms(field);</br>
      if (fieldTerms == null) {</br>
        return null;</br>
      }</br>
      // Reuse single TermsEnum below:</br>
      final TermsEnum te = fieldTerms.iterator(null);</br>
      for (int i = 0; i < terms.size(); i++) {</br>
        final Term t = terms.get(i);</br>
          //这里的state是根据该term是否在索引中存在生成的</br>
        final TermState state = states[i].get(context.ord);</br>
          //如果该term在索引中不存在，则会return null，说明在该种查询语法中每个term都必须出现，即使AND的关系</br>
        if (state == null) { /* term doesnt exist in this segment */</br>
          assert termNotInReader(reader, t): "no termstate found but term exists in reader";</br>
          return null;</br>
        }</br>
        te.seekExact(t.bytes(), state);//这里是具体的在索引文件里进行搜索的逻辑</br>
        DocsAndPositionsEnum postingsEnum = te.docsAndPositions(liveDocs, null, DocsEnum.FLAG_NONE);</br>
        // PhraseQuery on a field that did not index
        // positions.</br>
        if (postingsEnum == null) {</br>
          assert te.seekExact(t.bytes(), false) : "termstate found but no term exists in reader";</br>
          // term does exist, but has no positions</br>
          throw new IllegalStateException("field \"" + t.field() + "\" was indexed without position data; cannot run PhraseQuery (term=" + t.text() + ")");</br>
        }</br>
        postingsFreqs[i] = new PostingsAndFreq(postingsEnum, te.docFreq(), positions.get(i).intValue(), t);</br>
      }</br>
      // sort by increasing docFreq order</br>
      if (slop == 0) {</br>
        ArrayUtil.mergeSort(postingsFreqs);</br>
      }</br>
     //如果slop为0，即查询语句中的~后面的数字，则返回ExactPhraseScorer，否则返回SloppyPhraseScorer</br>
      if (slop == 0) {  // optimize exact case</br>
        ExactPhraseScorer s = new ExactPhraseScorer(this, postingsFreqs, similarity.exactSimScorer(stats, context));</br>
        if (s.noDocs) {</br>
          return null;</br>
        } else {</br>
          return s;</br>
        }</br>
      } else {
        return
          new SloppyPhraseScorer(this, postingsFreqs, slop, similarity.sloppySimScorer(stats, context));</br>
      }</br>
    }</br></br>
接着进入Scorer.score()方法中：</br></br>
/** Scores and collects all matching documents.</br>
   * @param collector The collector to which all matching documents are passed.</br>
   */</br>
  public void score(Collector collector) throws IOException {</br>
    collector.setScorer(this);</br>
    int doc;</br>
    while ((doc = nextDoc()) != NO_MORE_DOCS) {</br>
      collector.collect(doc);</br>
    }</br>
  }</br></br>
在这里主要是nextDoc()方法，进入SloppyPhraseScorer.nextDoc()方法：</br></br>
public int nextDoc() throws IOException {</br>
    return advance(max.doc);</br>
  }</br></br>
public int advance(int target) throws IOException {</br>
    sloppyFreq = 0.0f;</br>
    if (!advanceMin(target)) {</br>
      return NO_MORE_DOCS;</br>
    }       </br>
    boolean restart=false;</br>
    while (sloppyFreq == 0.0f) {</br>
      while (min.doc < max.doc || restart) {</br>
        restart = false;</br>
        if (!advanceMin(max.doc)) {</br>
          return NO_MORE_DOCS;</br>
        }       </br>
      }</br>
      // found a doc with all of the terms</br>
      sloppyFreq = phraseFreq(); // check for phrase</br>
      restart = true;</br>
    }</br>
    // found a match</br>
    return max.doc;</br>
  }</br></br>
private float phraseFreq() throws IOException {</br>
    if (!initPhrasePositions()) {</br>
      return 0.0f;</br>
    }</br>
    float freq = 0.0f;</br>
    numMatches = 0;</br>
    PhrasePositions pp = pq.pop();</br>
    System.out.println(pp+"---"+end);</br>
    int matchLength = end - pp.position;</br>
    int next = pq.top().position;</br>
    while (advancePP(pp)) {</br>
      if (hasRpts && !advanceRpts(pp)) {</br>
        break; // pps exhausted</br>
      }</br>
      if (pp.position > next) { // done minimizing current match-length</br>
        if (matchLength <= slop) {</br>
          freq += docScorer.computeSlopFactor(matchLength); // score match</br>
          numMatches++;</br>
        }     </br>
        pq.add(pp);</br>
        pp = pq.pop();</br>
        next = pq.top().position;</br>
        matchLength = end - pp.position;</br>
      } else {</br>
        int matchLength2 = end - pp.position;</br>
        if (matchLength2 < matchLength) {</br>
          matchLength = matchLength2;</br>
        }</br>
      }</br>
    }</br>
    if (matchLength <= slop) {</br>
      freq += docScorer.computeSlopFactor(matchLength); // score match</br>
      numMatches++;</br>
    }   </br>
    return freq;</br>
  }</br></br>
在这个方法可以看到matchLength和slop进行比较了。pq的heap（父类的属性）里面存放的是每个term所在field中的位置和该field属于哪个文档的信息：如heap[0]=d:0 o:0 p:7 c:0表示offset为0的term在dicid为0的文档中的当前查询field中的第7个位置，在该field中该term剩余count数量为0，即freq的大小,heap[1]=d:0 o:1 p:11 c:0。
