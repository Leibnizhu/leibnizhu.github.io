---
title: Lucene7获取所有有效文档
date: 2018-05-15 16:38:41
tags:
- lucene
- scala
image: mikuwing.jpg
---
最近在写一个`Vert.X+Lucene`的搜索引擎,为Vert.X中国论坛提供的(详见[Github](https://github.com/Leibnizhu/vertXearch)),在尝试更新文档索引的时候,遇到如何获取所有有效文档的问题(未被删除索引的). 官方没有直接查询的API,在Google和Stack Overflow搜了之后没有满意的效果,网上的资源很多都是Lucene 6甚至更老的3,4版本的,有些API已经过时.  
经过深入查阅官方API文档之后,终于找到了解决方案.  
Lucene删除索引之后,通过`IndexReader.document()`还是可以查到的,因此当有文档被删除后,不能直接用这个来查; 而`MultiFields.getLiveDocs()`在没有删除文档时返回`null`,有删除过文档时返回一个`Bits`对象,这里面可以通过`get()`方法,获取每个documentID是否有效(未被删除).  
因此可以这样写:  
```scala
  private val indexDirectory = FSDirectory.open(Paths.get(indexDirectoryPath))
  private var reader: DirectoryReader = DirectoryReader.open(indexDirectory)
  var indexSearcher = new IndexSearcher(reader)

  /**
    * 获取所有有效文档
    *
    * @return
    */
  def getAllDocuments: List[Document] = {
    //获取有哪些存活的文档
    val liveDocs = MultiFields.getLiveDocs(reader)
    if (liveDocs != null)
    //liveDocs非null时有删除过文件,遍历所有文档ID,liveDocs.get为true的话就是存活的,要过滤存活的文档对象
      (0 until reader.maxDoc()).filter(liveDocs.get).map(reader.document(_)).toList
    else
    //没有删除过文件的时候liveDocs为null,此时只能直接通过IndexSearcher去查询
      topDocsToDocumentList(indexSearcher.search(new MatchAllDocsQuery, Integer.MAX_VALUE))
  }
  
  def topDocsToDocumentList(topDocs: TopDocs): List[Document] = topDocs.scoreDocs.map(this.getDocument).toList
```