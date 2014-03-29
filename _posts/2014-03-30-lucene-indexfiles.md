---
layout: post
title: "lucene索引文件解析"
description: "lucene的各个索引文件形成解析"
category: essay
tags: []
---



fdt：docBase表示每个块开始的docID；numBufferedDocs表示文档数量；接下来是每个文档的字段数量，如果每个文档的字段数量都相同则首先会存储一个0然后存储字段数量，否则会计算最大的那个文档的字段数量所需要的bits并存储，接着存储每一个文档的字段数量；接着存储每个文档所有的字段的字符数之和，如果各个文档的字段字符数之和都一样，则首先也是存储一个0然后存储字符数，否则也计算最大的那个字符数所需要的bits并存储接着存储每一个文档的字符数之和。</br></br>
fdx：每次做索引的时候blockChunks都会自增加1，当blockChunks加到最大（1024）时，就会将该值写入该文件，并且将该值reset为0，在reset的时候会将blockDocs也置为0，该值表示该块里所有的文档数量，如果没有到1024则每次都会把该值写入该文件，所以该文件的格式为1 2 3…1024 1 2 3…1024 1…，如果blockChunks值为1表示第一次做索引，这时avgChunkSize设为0，该值表示每一个chunk中的平均文档数量，否则为Math.round((float) (blockDocs - docBaseDeltas[blockChunks - 1]) / (blockChunks - 1))其中docBaseDeltas存储的是每个chunk中的文档数量，比如第一次做索引有2个doc，第二次做索引有3个doc，第三次做索引有4个doc，那么docBaseDeltas[]={2,3,4},blockDocs为2+3+4=9，这时avgChunkSize=round(（9-4）/2)=3。接下来将totalDocs-blockDocs的值写入该文件，该值在第一个块即blockChunks<1024时都为0，在blockChunks>1024时大于0 ，如前1024次做索引总共的文档数量为1024，在第1025次时由于blockDocs经过rest置为0后，所以totalDocs-blockDocs=1025-1=1024。接着将上述的avgChunkDocs写入该文件。接着将各文档之间的差值的最大值所需的bits和个文档之间的差值写入该文件。接着将各个文档之间的startPointer的差值写入该文件。最后写入一个0表示该文件写入结束。</br></br>
doc：首先写入CODEC_MAGIC(1071082519)表示codec头的开始，接着写入所用的codec如Lucene41PostingsWriterDoc然后写入CURRENT_VERSION如0。接着写入PackedInts.VERSION_CURRENT如1，接着依次写入32个字节。</br></br>
pos：首先分别写入CODEC_MAGIC,所用codec，CURRENT_VERISON。接着写入各个term之间的位置差值</br></br>
pay：首先分别写入CODEC_MAGIC,所用codec，CURRENT_VERISON。</br></br>
tip：首先写入FILE_FORMAT_NAME如FST和VERSION_CURRENT如3，接着写入0或1表示是否packed</br></br>

(未完待续)
