# ES的基本概念
## ES存储架构图
![](./img/物理存储架构图.jpg)

## 基本名词解释
- 文档（document）  
  - ES中的数据使用JSON的形式存储，每一个JSON数据称为文档（对应Mysql的每一行数据）  
  - 因为是用采用JSON形式存储，所以ES的文档支持动态的给文档增加新的字段（不需要像Mysql需要先写ddl语句去修改表结构）
  - ES默认给文档中的每个字段都建立索引，如果想节约空间，可以手动来设置索引  
  - 文档字段如果是字符串类型，ES默认不支持字符串类型字段的聚合操作，因为会消耗大量内存。所以ES会自动给字符串类型的字段增加一个keyword类型的子文档，keyword类型的字段可以进行聚合
  - 文档的type属性：在5.0之前索引中的文档可以支持多个不同的type，6.0之后只支持type=_doc。5.0之前支持设置不同的type是为了可以将一个索引当成库使用，不同的type作为不同的表来使用，后来ES官方认为这是一个不好的设计，因为索引已经是一个比较小的单位，将不相关的文档放在同一个索引中不合适，因此ES官方后续偏向于废弃type属性，但是目前还有保留
- 文档映射(mapping)  
  - mapping中记录了一个ES索引中的文档字段名称和字段值的类型，以及这个字段要使用到的分词器、过滤器等
  - 当插入一个新的文档时，ES会自动识别字段值的类型并添加到文档映射中，字段类型在被记录到文档映射中后，就不能更改了，只能通过重建索引(reindex)的方式来进行修改  
- 段(segment)  
  段一部分文档的集合
- 索引(index)  
  索引在ES中就是一张表，文档就是存储在索引中，ES支持给索引起一个别名，可以将多个索引视为一个索引
- 分片(shared)  
  - 一个分片中会存储一个索引的部分文档，索引在被创建时可以指定需要几个分片，索引中的每一个文档会根据文档的id hash到不同的分片中（类似于分库分表），hash算法是 hash(id) % 分片数量，因此索引在创建后分片的数量就不可以修改，否则同一个文档会因为分片数量变化导致hash值改变而出现在分片上查找不到的问题
  - 分片又分为主分片与副本分片，ES服务正常的情况下只会使用主分片，副本分片是作为高可用的一个保障，同一个分片的主分片与副本分片是不会存放在同一个节点上，当一个主分片所在的节点出现故障时，其他节点上这个分片的副本分片就会升级为主分片来使用
  - 由于一个索引的分片可以被存储到不同的节点上，这样的分布式存储有时候会为搜索、聚合带来一部分的问题（大部分有解决方法），如果我们在创建索引时，指定索引的分片数量是1，那么所有的数据都会存储在同一个分片上，可以规避一些分布式存储带来的问题，但是这样做也会失去分布式存储的优势
- 节点(node)  
一个ES节点就是一台服务器，当一个接收到某个文档的增删改请求时，如果这个文档经过hash后算出的分片不在这个节点上，这个节点会将这个请求转发给存储这个分片的节点
- 集群(cluster)  
一个ES集群中由多个es节点组成，客户端在访问集群时，可以通过一个代理服务器将集群当成一个服务使用


# ES的基础使用
客户端通过restfulAPI来对服务端进行操作 
## 增加
- 指定文档id  
```txt
put /index/type/id
{
    "name":"张三",
    "age":"18"
}
```
- 自动生成文档id
```txt
{
    "name":"李四",
    "age":"20"
}
```
- bulk 批量增加
  ```txt
  POST /index/type/_bulk
  { "index":{ "_id": "8a78dhkujg" } } }
  { "name":"john doe","age":25 }
  { "index":{} }
  { "name":"mary smith","age":32 }
  ```  
  在index中可以指定增加文档的id，也可以指定要插入的索引、type
## 更新
- 更新整个文档  
    ```txt 
    put /index/type/id
    {
        "name":"张三",
        "age":"19"
    }
    ```
    这种方式会直接将原来的文档修改成请求中的数据
- 部分更新
    ```txt
    post /index/type/id/_update
    {
        "doc":{
            "age":"20",
            "sex":"男"
        }
    }
    ```
    这种方式只会修改请求中涉及的字段，如果文档中不存在请求中的字段，则会添加字段到文档中



## 删除
```txt
delete /index/type/id
```
## 查询
### 精确查询(term)
```txt
post /index/_search
{
    "query":{
        "term":{
            "fieldName":fieldValue
        }
    }
}
```
term查询会查找出匹配字段fieldName中等于fieldValue的文档  

- 注意：term查询是匹配经过分析器分析后的词项  
  如果fieldValue这个字段有使用分词器等操作（ES默认会对字符串类型的字段进行分词），那么term查询则匹配的是分词后的结果  
  - 如，文档字段name = 李四，默认分词会将中文拆分成单独的字，也就是 [李、四]，此时term查询 name = 李，可以查询出此文档，而term查询name = 李四查不到这个文档

  如果想对字符串字段做精确查询，可以设置字段为禁用分词器(not_analyze)或者使用keyword类型的字段来存储字符串
### 多个精确值的包含查询(terms)
```txt
post /index/_search
{
    "query":{
        "terms":{
            "fieldName":[fieldValue1,fieldValue2]
        }
    }
}
```
terms类似于 Mysql中的in，会将包含在数组内的文档都查出来
### 匹配查询(match)
```txt
post /index/_search
{
    "query":{
        "match":{
            "fieldName":fieldValue1
        }
    }
}
```
match会对字段分词后的结果进行匹配，查询条件中的值如果在分此后的内容中有出现，则会按照相关度排序后到结果集中
- 如match查询 name = 李五，索引中存在name=李五、name=李四的两个文档，并对name字段使用了默认的分词器将中文拆分为单独的字，此时的match查询会把这两条数据都查询出来，李五这条文档会排在李四之前，并且文档的相关度评分会比李四高  
- 如果字段没有经过分词器处理，那么match可以等价于term
### 范围查询(range)
```txt
post /index/_search
{
    "query":{
        "range":{
            "fieldName":{
                "gte":0,
                "lte":100
            }
        }
    }
}
```
range查询可以用于针对某字段做范围查询
- gt: > 大于（greater than）
- lt: < 小于（less than）
- gte: >= 大于或等于（greater than or equal to）
- lte: <= 小于或等于（less than or equal to）  

如果字段是日期类型，那么range也可以支持这样的查询
```txt
post /index/_search
{
    "query":{
        "range":{
            "fieldName":{
                "gte":"2022-2-10 00:00:00",
                "lte":"2022-2-11 23:59:59"
            }
        }
    }
}
```
### 分页(from+size)
```txt
post /index/_search
{
    "query":{
        "match_all":{}
    },
    "from":5,
    "size":10
}
```
- size表示要获取的结果集的大小
- from则是从匹配到的查询结果的第几条开始获取  

对于一些带有查询条件的聚合请求，只需要聚合后的结果，而不需要文档的内容，可以将size设置为0，减少文档数据的网络传输

### 字段是否存在查询(exists)
```txt
post /index/_search
{
    "query":{
        "exists":{
            "field":"fieldName"
        }
    }
}
```
### 组合查询(bool)
```txt
post /index/_search
{
	"query": {
		"bool": {
			"must": [
				{
				    "exists":{
				        "field":"age"
				    }
				},
				{
				    "range":{
				        "age":{
				            "gte":18
				        }
				    }
				},
				{
				    "match":{
				        "name":"李五"
				    }
				}
			],
			"must_not":[
			    {
			        "exists":{
			            "field":"sex"
			        }
			    }
			],
            "should":[
                {
                    "exists":{
                        "field":"addrss"
                    }
                }
            ]
		}
	}
}
```
bool提供了复杂查询的支持，可以同时使用多个查询条件来匹配数据
- must表示必须满足内部的查询条件
- must_not表示必须不满足内部的查询条件
- should是有满足条件则在结果集的排名更靠前（可以提高文档的相关度评分）
  
上面三个查询内部可以再嵌套bool查询以满足更复杂的查询
### 评分查询与过滤查询
ES中的查询分为两种：评分查询、过滤查询

#### 相关度评分
ES默认使用的查询方式就是评分搜索，也就是在查询文档时，除了文档需要满足查询条件外，还需要去给这个文档和搜索条件之间的关联度进行评分，评分后还要对结果集进行排序，评分结果会放入到_score字段，这个结果就是相关度评分  

#### 两者性能差距
过滤查询（Filtering queries）只是简单的检查包含或者排除，这就使得计算起来非常快。考虑到至少有一个过滤查询（filtering query）的结果是 “稀少的”（很少匹配的文档），并且经常使用不评分查询（non-scoring queries），结果会被缓存到内存中以便快速读取，所以有各种各样的手段来优化查询结果。

相反，评分查询（scoring queries）不仅仅要找出匹配的文档，还要计算每个匹配文档的相关性，计算相关性使得它们比不评分查询费力的多。同时，查询结果并不缓存。
#### 如何使用过滤查询
在查询请求中指定当前查询使用过滤上文下，es就会使用过滤查询来执行此次查询
```txt
post /index/_search
{
    "query" : {
        "constant_score" : { 
            "filter" : {
                "term" : { 
                    "field" : fieldValue
                }
            }
        }
    }
}
```
## 聚合
### 聚合相关名词解释
- 桶  
  桶就是满足特定条件的文档的集合（分组），类似于mysql中的group by sex  
  桶可以再嵌套一个桶，例如按照性别分桶后再按照年龄分桶  
  - 分桶注意事项（terms）  
    当需要根据某个字段分桶时，ES默认不允许对字符串类型的字段进行分桶，主要原因是ES担心字符串的字段非常大，会消耗大量内存与计算资源，因此如果要对字符串类型字段进行分桶，可以改为keyword类型。  
    如果不想调整为keywor类型，需要开启该字段的fielddata，ES会将这些字符串数据缓存到内存中，提高查询效率，开启了fielddata后，需要注意ES的内存消耗，fielddata内的数据在加载后只有根据最近最少使用策略来淘汰，也就是说一旦数据加入到fielddata后，供查询和聚合使用的内存就固定会减少这些数据大小，可以通过indices.fielddata.cache.size来限制fielddata的可以占用的内存空间大小
- 指标  
  桶将文档分到有意义的集合中，指标则是通过一些数学计算得到我们需要的文档计算结果  
  如计算男性用户有多少个人，类似于mysql中的 count(id)
### 总数
在ES中获取文档的总数有两种方法
- count API  
     ```txt
    post /index/_count
    {
	"query":{
	    "match_all":{}
	    }
    }
     ``` 
    这种方式可以查询出符合条件的文档总数
- 分桶后返回的doc_count(terms)
    ```txt
    post /index/_search
    {
	    "size": 0,
	    "aggs": {
		    "ageBucket": {
			    "terms": {
				    "field": "age.keyword",
                    "size":1000
			    }
		    }
	    }
    }
    ```
    terms分桶会会返回这个桶内的文档数量,terms中的size表示要对多少个匹配查询条件进行分桶，如果要计算总数量，size需要设置的比总数大（不推荐使用此方式来求总数）
### 求和(sum)
```txt
post /index/_search
{
    "size":0,
    "aggs":{
        "sumAgg":{
            "sum":{
                "field":"score"
            }
        }
    }
}
```
### 平均(avg)
```txt
post /index/_search
{
    "size":0,
    "aggs":{
        "avgAgg":{
            "avg":{
                "field":"score"
            }
        }
    }
}
```
### 最大、最小（max、min）
```txt
post /index/_search
//最大
{
	"size":0,
	"aggs":{
	    "maxScore":{
	        "max":{
	            "field":"score"
	        }
	    }
	}
}
//最小
{
	"size":0,
	"aggs":{
	    "maxScore":{
	        "min":{
	            "field":"score"
	        }
	    }
	}
}
```
### 统计(stats)
stats聚合会一起返回某个字段的文档总数、最小值、最大值、平均值、总和
post /index/_search
```txt
{
	"size": 0,
	"aggs": {
		"maxScore": {
			"stats": {
				"field": "score"
			}
		}
	}
}
```
### 直方图（histogram、date_histogram）
有时候我们想要按照字段的某个范围区间来分割桶，此时就可以采用histogram、如果字段是时间类信息则采用date_histogram  

比如我们想看考试成绩在0-20，20-40，40-60，60-80，80-100的人数分别有多少，这就是按照20分的范围来分割出不同的桶，人数就是桶的doc_count

```txt
post /index/_search
{
	"size": 0,
	"aggs": {
		"his": {
			"histogram": {
				"field": "score",
				"interval": 20,
				"extended_bounds": {
					"min": 0,
					"max": 100
				}
			}
		}
	}
}
```
默认的histogram聚合最小数值的桶只会根据已经存在的文档来展示，也就是说如果学生成绩都是80-100，那么前面的0-20、20-40之类的桶就不会展示，因此需要使用extended_bounds参数
- extended_bounds：显示直方图的最小值和最大值范围
### 扩展
ES中还有非常多的聚合方式，这里讲解了几种常用的聚合，后续聚合看官网文档 https://www.elastic.co/guide/en/elasticsearch/reference/7.17/search-aggregations.html
# ES中的数据结构
## 倒排索引(inverted index)
在了解倒排索引之前，先了解一下普通的索引：普通的索引通常是根据ID快速的查找出内容，比如根据用户ID查找出用户的姓名。而倒排索引则和普通的索引相反，是根据用户的姓名快速的查找出用户的ID，和普通的索引查询的方向相反，因此取名倒排索引。  

es中，倒排索引的作用是用于搜索，比如我们要根据某个单词来搜索出对应的文档内容，此时就是根据倒排索引进行搜索  
倒排索引是一个列表结构，每个列表项有两个属性
- 单词项（Terms）：单词项是指将内容经过分词后得到的每个单词名，每个单词项在倒排索引中是唯一的
- 倒排列表（DOC_ID）：记录有出现上面单词项的文档ID

如果此时有两个文档，文档的content内容是：  
1. The quick brown fox jumped over the lazy dog
2. Quick brown foxes leap over lazy dogs in summer  
那么此时给content字段创建的倒排索引的数据结构如下图  
![倒排索引](./img/倒排索引.jpg)  

这时候，如果要搜索`quick brown`，那么倒排索引就可以很快的找出含有quick和brown的两个文档
![](./img/倒排索引2.jpg)  
并且因为倒排索引中的列结构一致，可以很快的发i西安Doc_1中quick和brown都含有，这样可以很容易的确定Doc_1的相关度会比Doc_2高，Doc_1优先展示
- ps：叫分词索引会比叫倒排索引更容易理解

## 词典(Doc values)
上面的倒排索引主要作用是用于搜索，根据某个值查找出文档。但是对于聚合而言，倒排索引就很难使用，比如需要根据某个字段分桶，如果使用倒排索引，需要先把单词项整合成完整的内容，再根据完整的内容进行分桶，这一部分又需要遍历所有的整合后的内容进行去重，性能非常差，因此需要一个新的数据结构来满足聚合功能，也就是词典  
词典也是一种列表结构，每个列表项也有两个属性
- 文档ID(Doc)：记录文档ID
- 单词项(Terms)：记录文档拆分后的每个单词  
  
词典的存储结构和倒排索引正好相反，还是用上面的例子：
如果此时有两个文档，文档的content内容是：  
1. The quick brown fox jumped over the lazy dog
2. Quick brown foxes leap over lazy dogs in summer  
  
那么content字段建立的词典如下图：
![](./img/词典.jpg)  
这样，如果要对字段进行分桶时，只需要获取所有的Terms，然后求交集即可。并且对于sum、max、min、histogram等都可以高效的完成。  
- 另外，词典不仅仅只使用于聚合，任何需要查找某个文档包含的值的操作都必须使用它，还包括排序，访问字段值的脚本，父子关系处理等等

# ES文档更新过程
上面了解了ES内的数据结构后，再了解一下ES文档的更新  

## 倒排索引的不变性
倒排索引有一个特点：`不变性`，倒排索引一经创建就不能再更改
不变性的好处如下：
- 不需要锁。如果你从来不更新索引，你就不需要担心多进程同时修改数据的问题。
- 一旦索引被读入内核的文件系统缓存，便会留在哪里，由于其不变性。只要文件系统缓存中还有足够的空间，那么大部分读请求会直接请求内存，而不会命中磁盘。这提供了很大的性能提升（不变性的主要原因）。
- 其它缓存(像filter缓存)，在索引的生命周期内始终有效。它们不需要在每次数据改变时被重建，因为数据不会变化。
- 写入单个大的倒排索引允许数据被压缩，减少磁盘 I/O 和 需要被缓存到内存的索引的使用量。  
  
## 针对不变性的更新改进
- 不变性的弊端  
  不变性的弊端也很明显，如果对文档的某个字段建立了倒排索引，当增加文档、更新文档时，只能将倒排索引先删除，再重新根据所有文档建立倒排索引。当文档数量多时，更新的性能将会非常差。 
 
ES对倒排索引的更新性能改进方法：将所有的文档分`段(segment)`存储，倒排索引也分段建立。ES中的倒排索引不是针对所有的文档建立，而是针对一小部分的文档建立一个倒排索引，这一小部分文档就称为段，当新增、更新文档时，只需要重新建立这个文档所在段的倒排索引即可

## 文档的详细更新（新增）过程
![](./img/es的更新.jpg)
上面了解了`段`这个概念，已经为什么要有段的存在后，再来说一下文档的更新过程
- refresh  
  1.在向ES写入数据时，ES没有直接将更新写入到磁盘中，而是先记录到In memory buffer和Translog日志（日志用于防止ES重启更新丢失）中，此时的修改记录在in memory buffer中，在in memory buffer中的数据不会被ES搜索到  

  2.refresh就是新建一个segement，将ES记在in memory buffer中的文档写入到segement中，此时在segment中的文档就可以被搜索，使用refresh的原因是为了提高ES每次in memory buffer → segment的数据量，先记录到内存中的segment原因是 磁盘写入速度慢，先记录到内存中，那么数据可以被更快的搜索到

  3.默认的refresh间隔是1s，refresh比较消耗资源，官方建议等es自己的固定时间进行refresh，而不是手动触发refresh，es的get请求默认会先触发refresh再查询（如果在文档大量更新的情况下发送get请求，可能会因为refresh导致get请求超时）
- flush  
  es每隔一段时间，会把内存中的segment持久化到磁盘中，这个过程叫flush
- 段合并  
  由于refush流程每秒会创建一个新的段，这样会导致短时间内的段数量暴增。 每一个段都会消耗文件句柄、内存和cpu运行周期。并且每个搜索请求都必须轮流检查每个段；所以段越多，搜索也就越慢。  
  Elasticsearch通过在后台进行段合并来解决这个问题。小的段被合并到大的段，然后这些大的段再被合并到更大的段。  
  段合并时还会顺带着删除旧的已经被标记为删除的文档（ES中发出删除文档的请求后，并不会立刻删除，而是给文档打上一个要删除的标记），因为被标记的文档在段合并时不会被合并到新的大的段中，而原来的小段在合并后就会自动删除。


# ES的分布式搜索过程
## 分布式的查询
ES的查询过程分为两阶段：
### 1.Query阶段
在第一步中，首先客户端向ES集群中的某个节点发送查询请求，接收到这个请求的节点称为协调节点，协调节点接收到查询请求后，协调节点会创建一个大小为from + size的优先队列，然后将请求同步转发给其他的节点一起查询，每个节点（包括协调节点）会创建一个from + size的本地优先队列，然后将满足查询条件的文档ID和排序需要用的字段添加到本地优先队列中，然后将队列发回给协调节点，协调节点对所有返回的数据进行整合排序
### 2.Featch阶段
在对所有的数据整合排序后，协调节点会决定是否要去获取文档的所有内容（比如根据from+size的参数，舍弃前80个文档），决定后协调节点就会对其他节点发送muti get来获取文档的所有内容，将文档从ID丰富成客户端需要的文档内容后，协调节点就会将数据返回给护短端

## 分布式的聚合过程

# ES的深入使用
## es脚本（script）
es的script可以让我们像写程序代码一样来查询、修改es中的数据
- 脚本使用的语言是es自己开发的painless，类似于java，支持lambda表达式  
  
下面举两个简单的例子来了解script的使用
### 通过script进行query查询
比如查询名字长度是2的用户：
```txt
post /index/_search
{
	"query": {
		"script": {
			"script": {
				"source": "doc['name.keyword'].value.length() == 2",
				"lang": "painless"
			}
		}
	}
}
```
脚本说明：
- source：source中写要查询的脚本语句
  - doc：表示es中的整个文档对象
  - []：中括号中放要操作的文档字段的路径
  - value：value是脚本语言painless中的一个关键字，表示字段的值
  - length():调用String对象的length方法，获取字符串长度
  - == 2：匹配条件,source最终要返回true Or false
- lang :表示脚本使用的语言，默认采用painless，该字段可以不写
### 通过script查询自定义返回值
script还支持查询自定义的返回值
比如我们要查询有考试过的用户，既有存在score字段的用户，并在返回结果中直接表示出来
```txt
post /test_user/_search
{
	"script_fields": {
		"eaxmed": {
			"script": {
				"source": "if(doc['score'].size() == params['defalut']) {return 0;}else return 1;",
				"params": {
					"defalut": 0
				}
			}
		},
		"user": {
			"script": {
				"source": "doc['name.keyword'].value"
			}
		}
	}
}
```
脚本说明：
- script_fields：表明要使用脚本语言来自定义生成返回字段
  - eaxmed：自定义返回的字段名
  - source: 字段返回值语句，里面的内容是如有文档存在score字段则返回0，否则返回1
  - params：source语句中的参数对象，可以在source语句中调用paras中的字段。例子中就使用了params中的defalut字段
### 使用script进行更新  
例如给名为张三的用户考试成绩加两分：
```txt
/test_user/_update_by_query
{
	"script":{
	    "source":"ctx._source.score += 2"
	},
	"query":{
	    "term":{
	        "name.keyword":"张三"
	    }
	}
}
```
脚本说明：
- script:脚本语句json体
  - soruce:如何更新的脚本语句
  - ctx:更新语句对象的上下文，ctx是简写,全名是contxt
  - _source：代表要更新的文档
  整个脚本就是将score字段在原值的基础上加2
## 分析器
ES中的分析器的作用就是对存储到es索引中的文本进行分析、转换，提高全文检索的相关度，在创建索引时可以指定哪些字段要使用分析器  
分析器的工作流程：
1. 首先，将一块文本分成适合于倒排索引的独立的 词条 ，
2. 之后，将这些词条统一化为标准格式以提高它们的“可搜索性”  

分析器由三个部分组成：
- 字符过滤器(char_filter)：  
  首先，字符串按顺序通过每个字符过滤器。他们的任务是在分词前整理字符串。比如一个字符过滤器可以用来去掉HTML，或者将 & 转化成 and。
- 分词器(tokenizer)：  
  之后，字符串被分词器分为单个的词条。比如按照空格将字符串切分成单词
- Token 过滤器(filter)  
  最后，每个被切分后的词条经过Token过滤器，这个过程可以改变词条。比如将词条从大写都转为小写
### ES内置的分析器
为了方便ES的使用，ES内置了几种分析器  
以  `"Set the shape to semi-transparent by calling set_trans(5)"`为例，经过各种分析器后会得到什么结果
- 标准分析器：  
  标准分析器是Elasticsearch默认使用的分析器。它是分析各种语言文本最常用的选择。它根据 Unicode联盟定义的单词边界划分文本。删除绝大部分标点。最后，将词条小写。  
  - 分析后的结果为：`set, the, shape, to, semi, transparent, by, calling, set_trans, 5`
- 简单分析器：在任何不是字母的地方分隔文本，将词条小写
  - 分析后的结果为：`set, the, shape, to, semi, transparent, by, calling, set, trans`
- 空格分析器：空格分析器在空格的地方划分文本
  - 分析后的结果为：`Set, the, shape, to, semi-transparent, by, calling, set_trans(5)
- 语言分析器：  
  特定语言分析器可用于很多语言。它们可以考虑指定语言的特点。例如， 英语分析器附带了一组英语无用词（常用单词，例如 and 或者 the ，它们对相关性没有多少影响），它们会被删除。由于理解英语语法的规则，这个分词器可以提取英语单词的 词干。  
  - 英语 分词器会产生下面的词条：  
    set, shape, semi, transpar, call, set_tran, 5
    注意看 transparent、 calling 和 set_trans 已经变为词根格式。
### 自定义分析器
我们也可以自己创建一个分析器，以存储一个html页面作业检索字段为例：  
这个分析器要实现以下两个功能：
1. 使用 html清除 字符过滤器移除HTML部分
2. 使用一个自定义的 映射 字符过滤器把 & 替换为 "and",还想把"hi"转成"hello"
3. 使用标准分词器分词
4. 将所有单词转换为小写,使用 小写 词过滤器处理
5. 使用自定义 停止 词过滤器移除自定义的停止词列表中包含的词
完整的创建索引语句为：  
```txt
put /testan
{
	"settings": {
		"analysis": {
			"analyzer": {
				"my_analyzer": {
					"type": "custom",
					"char_filter": [
						"html_strip",
						"my_char_filter"
					],
					"tokenizer": "standard",
					"filter": [
						"lowercase",
						"my_stopwords"
					]
				}
			},
			"char_filter": {
				"my_char_filter": {
					"type": "mapping",
					"mappings": [
						"&=>and",
						"hi=>hello"
					]
				}
			},
			"filter": {
				"my_stopwords": {
					"type": "stop",
					"stopwords": [
						"the",
						"a",
						"an"
					]
				}
			}
		},
        "mapping":{
            "properties": {
		    "content": {
			    "type": "text",
			    "analyzer": "my_analyzer"
		        }
	        }
        }
	}
}
```
语句说明：
1. 建立char_filter:my_char_filter，创建了mapping过滤器来转换&和hi，然
2. 创建filter:my_stopwords，将the a an从分词后的结果中剔除
3. 创建analyzer:my_analyzer,将刚刚创建的my_char_filter、my_stopwords、以及es自带的char_filter:html_script、tokenizer：standard组合起来成为一个新的分析器  

测试分析器：
```txt
post /index/_analyze
{
	"analyzer": "my_analyzer",
	"text": "hi <i>Nihao</i> an a apple , the APPLE IS GOOD"
}
```
使用上面的API即可测试刚刚创建的分析器，会返回经过分析器后的字符串结果

## 高级检索
### 多词查询（match）
当我们使用match时，有时候想精确的查找由多个单词组成的词，但是由于match的机制，当查询多词时，会将只有包含一部分单词的文档也检索出来，因为match会对要查询的词进行分词，然后转换为多个term查询。这时候就可以使用多词查询来进行精确匹配
- 精确的多词匹配
精确即多个词项一定要在字段中出现才匹配
```txt
post /index/_search
{
	"query": {
		"match": {
			"text": {
				"query": "question2 answer",
				"operator": "and"
			}
		}
	}
}
```
- 精准度匹配
```txt
post /index/_search
{
	"query": {
		"match": {
			"text": {
				"query": "question2 answer ga",
				"minimum_should_match": "75%"
			}
		}
	}
}
```
minimum_should_match：查询精度，匹配的词项数会向下取整，当精度是75%、要匹配的词项是3个时，只要文档的有同时出现两个词即可匹配
## 扩展
es中的查询方式非常多，比如跨字段匹配、上下文匹配、自动不全搜索等等，这边还是不做详细说明，具体到es官方文档查看  
https://www.elastic.co/guide/en/elasticsearch/reference/7.17/query-dsl.html
# 相关性
相关性是ES检索中的重要因素，相关性高的文档会优先展示，那么ES是如何评估一次检索中的文档相关性？ES自5.0后默认使用的打分算法是`Okapi BM25`,
算法的具体内容不深究，只说明一下影响这个算法的最终打分的主要因素由以下几点：
- 词频：  
  词频表示要搜索的词在文档中出现的频度是多少？频度越高，权重越高 。 文档中出现5次要搜索的词比只出现1次的高
- 逆向文档频率：  
  逆向文档表示要检索的词在所有文档里出现的频率是多少？频次越高，权重 越低 。如果要检索的词中包含一些常用词如 and 或 the，这些词对这个文档的相关度贡献很少，因为它们在多数文档中都会出现；而要检索的词中包含一些常见的词如 elastic 或 hippopotamus在所有文档中少见，如果文档中正好存在这种词语，那么这个词对文档的相关度贡献就比较高
- 字段长度归一值：  
  字段长度归一值表示要检索的字段整体长度是多少？字段越短，字段的权重 越高 。如果词出现在类似标题 title 这样的字段，要比它出现在内容 body 这样的字段中的相关度更高

# ES的数据建模
和关系型数据库不同：ES数据建模推荐使用反范式。  
主要原因是于全文检索这一功能来说当关联多张表时性能很差、几乎不可用，且ES还是分布式的，跨服务器关联的成本更大。因此在建模时尽量将适合冗余的数据都放到同一个模型（索引结构）中，并且ES支持对单文档的事务。
- 反范式的优缺点：  
  优点：对于查询而言，不需要关联再查询其他表，查询速度快  
  缺点：对于更新而言，由于ES的更新机制，每次都需要重建整个文档，如果文档大，更新速度慢  
## 关联关系的处理
首先明确一点：不建议在ES数据建模中使用关联关系，而是优先使用反范式的方式建模。  
如果文档中不同文档之间确实有关联关系，那么该如何处理

### 嵌套对象（Nested）
嵌套对象是es中的一种特殊的字段类型，可以使用嵌套对象将文档的关联都放到文档中的一个字段中，这样就可以不用再进行一次关联查询
- 嵌套对象的使用方式：  
  嵌套对象的使用比较简单，在创建索引映射时指定嵌套对象所在的著文档的字段类型是nested即可
- 嵌套对象与数组字段的区别：  
  嵌套对象和数组对象的表现形式很像，都是一个数组
  - 数组对象在存储时es会对其进行扁平化处理，也就是将对象的字段都打平做一个数组字段存储，如一个"博客的评论"数组对象，在ES内部的存储会变成
    ```txt
    "comments.name":    [ alice, john, smith, white ],
    "comments.comment": [ article, great, like, more, please, this ],
    "comments.age":     [ 28, 31 ],
    "comments.stars":   [ 4, 5 ],
    "comments.date":    [ 2014-09-01, 2014-10-22 ]
    ```  
    这样带来的问题就是由于被扁平化处理，就无法精准的查询数组对象中的某个对象，比如精准的查询出博客中有评论name是alice、age是28，此时可能会返回name是alice、age是29，name是jon、age是28的博客  

  - 嵌套对象则会单独存储到一份隐藏的文档中，并将这份文档存储到主文档中，不会经过扁平化处理，这时才可以精准的匹配文档中的数组对象  
  
- 虽然嵌套对象是单独存储一份文档，但是这份文档存储在主文档中，无法直接获取，对文档的嵌套对象的更新然后需要重建整个文档，所以嵌套对象并不适用于文档有需要关联对象且关联对象更新频繁的场景
- 嵌套对象不能单独做为搜索返回结果的主体，返回结果的主体是主文档
## 父子文档(Join)
### 父子文档介绍
父子文档和嵌套对象的区别就是父文档与子文档分开，子文档有自己的存储空间，并且可以直接获取，父子文档仍存储在同一个索引中。  

Elasticsearch 维护了一个父文档和子文档的映射关系，所以父-子文档关联查询操作非常快。但是这个映射也对父-子文档关系有个限制条件：。

- 父子文档和嵌套对象相比的优势：
  - 更新父文档时，不会重新索引子文档
  - 创建，修改或删除子文档时，不会影响父文档或其他子文档。这一点在这种场景下尤其有用：子文档数量较多，并且子文档创建和修改的频率高时
  - 子文档可以作为搜索结果独立返回
- 父子文档的限制：
  - 父文档和其所有子文档，都必须要存储在同一个分片中，因此在get、更新、存储子文档时需要显示声明要存储的分片编号为父文档所在分片
  - 一个索引中只能有一个父子文档join类型字段
  - 通过父文档关联查询子文档的速度会比嵌套对象的查询慢5-10倍（虽然慢很多，但还是能接收）  
- 父子文档适用场景：  
  父子文档最适用于子文档数远多余父文档数的场景
### 父子文档使用方式
1. 首先需要在创建索引时指定字段的类型为join类型，join类似用来表达父子文档的关联方式
    ```txt
    put /index
    {
    "mappings": {
        "properties": {
        "my_id": {
            "type": "keyword"
        },
        "my_join_field": { 
            "type": "join",
            "relations": {
            "question": "answer" 
            }
        }
        }
    }
    }
    
    ```
    在建立索引时指定某个字段的类型是join，并在relations中表明关联方式，示例中的代码表示1个question会对应多个answer，question就是父文档，answer就是子文档
    - 一个父文档可以对应多种子文档、并且子文档可以有自己对应的子文档（也即是孙子文档），但是es并不建议使用层数过多的关联  
    对应的索引创建方式为：  
        ```txt
        PUT my-index-000001
        {
        "mappings": {
            "properties": {
            "my_join_field": {
                "type": "join",
                "relations": {
                //以数组的形式声明多个子文档
                "question": ["answer", "comment"],  
                //子文档再声明自己的子文档
                "answer": "vote" 
                }
            }
            }
        }
        }

        ```
2. 增加父文档  
    ```txt
    PUT index/_doc/1
    {
        "my_id": "1",
        "text": "This is a question",
        "my_join_field": {
            "name": "question" 
        }
    }

    PUT index/_doc/2
    {
        "my_id": "2",
        "text": "This is another question",
        "my_join_field": {
            "name": "question"
        }
    }
    ```
    在刚刚建立的join类型字段的name字段中声明当前文档在父子文档中是父还是子
    - 对于父文档的join类型字段可以简写成
        ```txt
        {
            "my_id": "2",
            "text": "This is another question",
            "my_join_field": "question"
        }
        ```
3. 建立子文档
    ```txt
    PUT my-index-000001/_doc/3?routing=1
    {
    "my_id": "3",
    "text": "This is an answer",
    "my_join_field": {
        "name": "answer", 
        "parent": "1" 
        }
    }
    
    PUT my-index-000001/_doc/3?routing=1
    {
    "my_id": "4",
    "text": "This is another answer",
    "my_join_field": {
        "name": "answer",
        "parent": "1"
        }
    }
    ```
    子文档新增时，在join类型的字段中填入子文档的标识，并且子文档需要通过parant字段指定父文档的id。因为父子文档必须在在同一个分配请求中需要使用routing = 父文档id 让子文档分配到父文档所在的分片中  
      
4. 到这就完成了父子文档的构建，下面看一下如何进行父子文档的关联查询  
- 通过子文档查询父文档（has_child）  
  ```txt
    post index/_search
    {
        "query":{
            "has_child":{
                "type":"answer",
                "query":{
                    "term":{
                        "my_id":"5"
                    }
                }
            }
        }
    }
  ```
  上面的例子表示查询子文档id=5的父文档
- 通过父文档查询子文档(has_parent)  
  ```txt
  post /index/_search
  {
    "query":{
        "has_parent":{
            "parent_type":"question",
            "query":{
                "term":{
                    "my_id":"1"
                }
            }
        }
    }
    }
    ```
    上面的例子就是查询父文档id=1的所有子文档


# ES集群监控管理
es中提供了一些用于查询ES集群、索引运行情况的API
## 查看集群健康(cluster health)
```txt
get _cluster/health

返回信息：
{
   "cluster_name": "elasticsearch_zach",
   "status": "green",
   "timed_out": false,
   "number_of_nodes": 1,
   "number_of_data_nodes": 1,
   "active_primary_shards": 10,
   "active_shards": 10,
   "relocating_shards": 0,
   "initializing_shards": 0,
   "unassigned_shards": 0
}
```
- status：  
  返回信息中最重要的字段就是`status`,这代表集群当前的运行状况
    - green：  
    所有的主分片和副本分片都已分配。你的集群是 100% 可用的。
    - yellow：  
    所有的主分片已经分片了，但至少还有一个副本是缺失的。不会有数据丢失，所以搜索结果依然是完整的。不过，你的高可用性在某种程度上被弱化。如果 更多的 分片消失，你就会丢数据了。把 yellow 想象成一个需要及时调查的警告。（如果在建立索引时，如果索引只有主分片，副本分配的数量是0，那么集群一定会处于yellow状态）
    - red：  
    至少一个主分片（以及它的全部副本）都在缺失中。这意味着你在缺少数据：搜索只能返回部分数据，而分配到这个分片上的写入请求会返回一个异常。  
## 索引、分片级别的健康状态
- 索引级
    ```txt
    GET _cluster/health?level=indices
    ```
    通过在API后面指定level是索引级，那么返回结果会详细的显示每个索引是green还是yellow、red
- 分片级  
  ```txt
  GET _cluster/health?level=shards
  ```
## 节点的状态(node stats)
通常一个集群出问题了，还需要定位到具体是哪个节点出现问题，这时候可以使用查看节点统计值API
```txt
GET _nodes/stats
```
这个API会返回集群中每个节点的基本信息
```txt
{
   "cluster_name": "elasticsearch_zach",
   "nodes": {
      "UNr6ZMf5Qk-YCPA_L18BOQ": {
         "timestamp": 1408474151742,
         "name": "Zach",
         "transport_address": "inet[zacharys-air/192.168.1.131:9300]",
         "host": "zacharys-air",
         "ip": [
            "inet[zacharys-air/192.168.1.131:9300]",
            "NONE"
         ],
...
```
### 索引状态
上面的节点状态API还会返回节点中每个索引的统计状态
```txt
"indexing": {
           "index_total": 803441,
           "index_time_in_millis": 367654,
           "index_current": 99,
           "delete_total": 0,
           "delete_time_in_millis": 0,
           "delete_current": 0
        },
        "get": {
           "total": 6,
           "time_in_millis": 2,
           "exists_total": 5,
           "exists_time_in_millis": 2,
           "missing_total": 1,
           "missing_time_in_millis": 0,
           "current": 0
        },
        "search": {
           "open_contexts": 0,
           "query_total": 123,
           "query_time_in_millis": 531,
           "query_current": 0,
           "fetch_total": 3,
           "fetch_time_in_millis": 55,
           "fetch_current": 0
        },
        "merges": {
           "current": 0,
           "current_docs": 0,
           "current_size_in_bytes": 0,
           "total": 1128,
           "total_time_in_millis": 21338523,
           "total_docs": 7241313,
           "total_size_in_bytes": 5724869463
        },
```
这部分信息对于了解索引的工作情况非常有帮助。  

- indexing：  
  显示已经索引了多少文档。这个值是一个累加计数器。在文档被删除的时候，数值不会下降。还要注意的是，在发生内部 索引 操作的时候，这个值也会增加，比如说文档更新。还列出了索引操作耗费的时间，正在索引的文档数量，以及删除操作的类似统计值。

- get：  
  显示通过 ID 获取文档的接口相关的统计值。包括对单个文档的 GET 和 HEAD 请求。
- search：  
  描述正在查询中的搜索（ open_contexts ）数量、查询的总数量、以及自节点启动以来在查询上消耗的总时间。用 query_time_in_millis / query_total 计算的比值，可以用来粗略的评价查询有多高效。比值越大，每个查询花费的时间越多。

- fetch：  
  统计值展示了查询处理的后一半流程（query-then-fetch 里的 fetch ）。如果 fetch 耗时比 query 还多，说明磁盘较慢，或者获取了太多文档，或者可能搜索请求设置了太大的分页（比如， size: 10000 ）。

- merges：  
  包括了 Lucene 段合并相关的信息。它会告诉你目前在运行几个合并，合并涉及的文档数量，正在合并的段的总大小，以及在合并操作上消耗的总时间。

```txt
       "filter_cache": {
           "memory_size_in_bytes": 48,
           "evictions": 0
        },
        "fielddata": {
           "memory_size_in_bytes": 0,
           "evictions": 0
        },
        "segments": {
           "count": 319,
           "memory_in_bytes": 65812120
        },
```
- filter_cache：  
  展示了已缓存的过滤器位集合所用的内存数量，以及过滤器(fliter)被驱逐出内存的次数。驱逐次数过多，可能表示需要加大过滤器的缓存，或者你的过滤器不太适合缓存（比如它们因为高基数而在大量产生，就像是缓存一个 now 时间表达式）。  

  不过，驱逐数是一个很难评定的指标。过滤器是在每个段的基础上缓存的，而从一个小的段里驱逐过滤器，代价比从一个大的段里要廉价的多。有可能你有很大的驱逐数，但是它们都发生在小段上，也就意味着这些对查询性能只有很小的影响。  

  把驱逐数指标作为一个粗略的参考。如果你看到数字很大，检查一下你的过滤器，确保他们都是正常缓存的。不断驱逐着的过滤器，哪怕都发生在很小的段上，效果也比正确缓存住了的过滤器差很多。

- field_data：  
 显示 fielddata 使用的内存，用以聚合、排序等等。这里也有一个驱逐计数。和 filter_cache 不同的是，这里的驱逐计数是很有用的：这个数应该或者至少是接近于 0。因为 fielddata 不是缓存，任何驱逐都消耗巨大，应该避免掉。如果你在这里看到驱逐数，你需要重新评估你的内存情况，fielddata 限制，请求语句。
- segments：  
 会展示这个节点目前正在使用中的 Lucene 段的数量。这是一个重要的数字。大多数索引会有大概 50–150 个段，哪怕它们存有 TB 级别的数十亿条文档。段数量过大表明合并出现了问题（比如，合并速度跟不上段的创建）。注意这个统计值是节点上所有索引的汇聚总数。记住这点。  
   
   - memory：  
     统计值展示了 Lucene 段自己用掉的内存大小。这里包括底层数据结构，比如倒排表，字典，和布隆过滤器等。太大的段数量会增加这些数据结构带来的开销，这个内存使用量就是一个方便用来衡量开销的度量值。
# 总结
这篇笔记总结了ElasticSearch的大部分使用知识点：ES的架构、搜索、聚合、ES、搜索、更新的执行过程、数据建模、监控等，更详细的内容还需要从官方文档获取。  
后续可能还会再出一篇经常与ElasticSearch一同使用的组件：Kibana、logstash等的笔记