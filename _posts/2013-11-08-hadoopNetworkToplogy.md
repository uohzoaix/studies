---
layout: post
title: "聊聊hadoop的NetworkTopology"
category: 
- hadoop
tags: []
---






NetworkTopology是网络拓扑的意思，这在网络架构中运用很多，机器的各种部署等等。hadoop的DataNode也是分布在各个节点设置各个不同的机架上的，所以数据的读取必然会联系到网络的拓扑结构。NetworkTopology类很简单，就是对一棵树进行各种操作，下面主要分析一下该类中InnerNode内部类的 getLeaf方法，代码如下：
{% highlight objc %}
/** get <i>leafIndex</i> leaf of this subtree 
     * if it is not in the <i>excludedNode</i>*/
protected Node getLeaf(int leafIndex, Node excludedNode) {
    int count=0;
    // check if the excluded node a leaf
    boolean isLeaf = excludedNode == null || !(excludedNode instanceof InnerNode);
    // calculate the total number of excluded leaf nodes
    int numOfExcludedLeaves = isLeaf ? 1 : ((InnerNode)excludedNode).getNumOfLeaves();
    if (isLeafParent()) { // children are leaves 
        if (isLeaf) { // excluded node is a leaf node
            //excludeNode为null时，excludedIndex=-1，否则excludedIndex>=0
            int excludedIndex = children.indexOf(excludedNode);
            if (excludedIndex != -1 && leafIndex >= 0) {
                // excluded node is one of the children so adjust the leaf index
                //excludedNode!=null，这时如果所要搜索的节点下标小于excludedIndex则不作处理，否则跨过该excludeNode
                leafIndex = leafIndex>=excludedIndex ? leafIndex+1 : leafIndex;
            }
        }
        //excludedIndex=-1会直接进入这里表示excludedNode=null，这时直接从该InnerNode中获取leafIndex即可
        //这里需要检查leafIndex是否大于该InnerNode的子节点数量
        // range check
        if (leafIndex<0 || leafIndex>=this.getNumOfChildren()) {
            return null;
        }
        return children.get(leafIndex);
    }  else {
        //如果该InnerNode下的子节点含有子节点则遍历InnerNode的子节点
        for(int i=0; i<children.size(); i++) {
            InnerNode child = (InnerNode)children.get(i);
            if (excludedNode == null || excludedNode != child) {
                // not the excludedNode
                int numOfLeaves = child.getNumOfLeaves();//该子节点的叶子节点数量
                if (excludedNode != null && child.isAncestor(excludedNode)) {
                    //该子节点是excludeNode的祖先节点则需要将excludeNode的叶子节点排除
                    numOfLeaves -= numOfExcludedLeaves;
                }
                //如果先前遍历过的子节点的所有叶子节点和该子节点的叶子节点数量之和大于所要寻找的下标则在该子节点上递归调用getLeaf方法，否则累加叶子节点数量
                if (count+numOfLeaves > leafIndex) {
                    // the leaf is in the child subtree
                    return child.getLeaf(leafIndex-count, excludedNode);
                } else {
                    // go to the next child
                    count = count+numOfLeaves;
                }
            } else { // it is the excluededNode
                // skip it and set the excludedNode to be null
                excludedNode = null;
            }
        }
        return null;
    } 
}
{% endhighlight %}
该类的两个参数说明：</br>
leafIndex就是我们需要从这个InnerNode下获取的那个节点的下标，excludeNode是我们指定的不参与该搜索的那个节点，当然你也可以将该参数传递null即所搜全部节点。</br></br>
代码一开始检查excludeNode是否是叶子节点，这里叶子节点的判定条件是excludeNode为null或者excludeNode不是内部节点，InnerNode是NetworkTopology的一个内部类，代表一个机架上的一个交换机或者路由，所以它肯定是有子节点的，否则这个交换机或者路由的存在是没有必要的。</br></br>
接下来numOfExcludedLeaves表示excludeNode包含的子节点数量，如果excludeNode本身就是一个子节点那么该值自然为1，如果不是子节点则该值为它本身的子节点数量。</br></br>
接下来的isLeafParent()方法判断所搜索的这个InnerNode下的节点是否都为叶子节点。
