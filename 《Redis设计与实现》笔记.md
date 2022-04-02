[toc]
# 《Redis设计与实现》
## 第二章.简单动态字符串
### SDS
SDS:Redis中用来替代C字符串的数据类型。  
定义:
```
struct sdshdr
{
    //已使用字节长度
    int len;
    //未使用字节数量
    int free;
    //用于保护字符串的字节数组
    char buf[];
}
```

### SDS与C字符串
#### SDS可使用一部分<string.h>库函数
SDS中数据放在buf中,同时其末尾为'\0'，所以我们可以直接使用C函数:  

```
printf("%s",s->buf);
```
也因此，可使用部分sting.h库中函数
#### len 时间复杂度O(1)
SDS用len记录存放数据长度，故获取其长度时间复杂度为O(1);  
#### SDS杜绝缓冲区溢出
SDS杜绝缓冲区溢出:
```
C字符串使用strcat将两个字符串拼接时，需要先为第一个参数字符串分配足够空间，否则可能导致缓冲区溢出(buffer overflow).  
而SDS杜绝了这种可能:SDS API需要对SDS进行修改时，API检查SDS的内存空间，如果不足会自动帮其扩展至修改所需大小。
SDS用于拼接的API:sdscat
```
#### free 减少内存重分配次数
1.空间预分配:减少重分配次数  
对一个SDS进行修改并需要对其进行空间扩展时，会为其分配额外未使用空间
```
若len将小于1MB,分配等同len大小的未使用空间

若len大于等于1MB,分配1MB的未使用空间

如len为30MB:
分配:30MB+1MB+1byte (1byte:'\0')
```
2.惰性空间释放  
缩短字符串长度时，不立即内存重分配，而是保留至free长度未释放空间，避免了内存重分配且为可能的增长操作提供优化。 
(有需要时，可真正释放SDS未使用空间)
#### SDS二进制安全
SDS的API二进制安全，所以SDS API以处理二进制的方式处理SDS存放在buf数组中的数据，而不会对其进行限制、过滤，所以SDS的buf属性也称字节数组。

#### SDS API

|函数|作用|时间复杂度|
| --- | --- | --- |
sdsnew|创建一个包含给定 C 字符串的 SDS 。|O(N) ， N 为给定 C 字符串的长度。
sdsempty|创建一个不包含任何内容的空 SDS 。|O(1)
sdsfree|释放给定的 SDS 。|O(1)
sdslen|返回 SDS 的已使用空间字节数。|这个值可以通过读取 SDS 的 len 属性来直接获得， 复杂度为 O(1) 。
sdsavail|返回 SDS 的未使用空间字节数。|这个值可以通过读取 SDS 的 free 属性来直接获得， 复杂度为 O(1) 。
sdsdup|创建一个给定 SDS 的副本（copy）。|O(N) ， N 为给定 SDS 的长度。
sdsclear|清空 SDS 保存的字符串内容。|因为惰性空间释放策略，复杂度为 O(1) 。
sdscat|将给定 C 字符串拼接到 SDS 字符串的末尾。|O(N) ， N 为被拼接 C 字符串的长度。
sdscatsds|将给定 SDS 字符串拼接到另一个 SDS 字符串的末尾。|O(N) ， N 为被拼接 SDS 字符串的长度。
sdscpy|将给定的 C 字符串复制到 SDS 里面， 覆盖 SDS 原有的字符串。|O(N) ， N 为被复制 C 字符串的长度。
sdsgrowzero|用空字符将 SDS 扩展至给定长度。|O(N) ， N 为扩展新增的字节数。
sdsrange|保留 SDS 给定区间内的数据， 不在区间内的数据会被覆盖或清除。|O(N) ， N 为被保留数据的字节数。
sdstrim|接受一个 SDS 和一个 C 字符串作为参数， 从 SDS 左右两端分别移除所有在 C 字符串中出现过的字符。|O(M*N) ， M 为 SDS 的长度， N 为给定 C 字符串的长度。
sdscmp|对比两个 SDS 字符串是否相同。|O(N) ， N 为两个 SDS 中较短的那个 SDS 的长度。

## 第三章.链表
### 链表结构
链表节点使用一个adlist.h/listNode结构来表示:
```
typedef struct listNode{
    //前置节点
    struct listNode *prev;
    
    //后置节点
    struct listNode *next;
    
    //节点的值
    void *value;
}listNode;
```

可以使用adlist.h/list来持有链表:
```
typedef struct list{
    //表头节点
    listNode *head;
    
    //表尾节点
    listNode *tail;
    
    //链表所包含的节点数量
    unsigned long len;
    
    //节点值复制函数 用于复制链表节点保存值
    void *(*dup)(void *ptr);
    
    //节点值释放函数 用于释放链表节点所保存的值
    void (*free) (void *ptr);
    
    //节点值对比函数 用于对比链表节点所保存值和另一个输入值是否相等
    int (*match)(void *ptr,void *key);
}list;
```

### Redis链表特性   
**双端**(获取某节点前置后置节点复杂度O(1))    
**无环**(表头prev表尾next指向NULL)    
**带表头表尾指针**(对表头表尾节点获取复杂度O(1))  
**带链表长度计数器**(获取链表长度复杂度O(1))  
**多态**:(使用void*指针保存节点值，且可通过dup、free、match为节点值设置类型特定函数，故可保存不同类型值)

### Redis API

函数|作用|时间复杂度|
| --- | --- | --- |
listSetDupMethod|将给定的函数设置为链表的节点值复制函数|O(1) |
listGetDupMethod|返回链表当前正在使用的节点值复制函数|复制函数可以通过链表的 dup 属性直接获得， O(1)|
listSetFreeMethod|将给定的函数设置为链表的节点值释放函数|O(1) |
listGetFree|返回链表当前正在使用的节点值释放函数|释放函数可以通过链表的 free 属性直接获得， O(1)|
listSetMatchMethod|将给定的函数设置为链表的节点值对比函数|O(1)|
listGetMatchMethod|返回链表当前正在使用的节点值对比函数|对比函数可以通过链表的 match 属性直接获得， O(1)|
listLength|返回链表的长度（包含了多少个节点）|链表长度可以通过链表的 len 属性直接获得， O(1) |
listFirst|返回链表的表头节点|表头节点可以通过链表的 head 属性直接获得， O(1) |
listLast|返回链表的表尾节点|表尾节点可以通过链表的 tail 属性直接获得， O(1) |
listPrevNode|返回给定节点的前置节点|前置节点可以通过节点的 prev 属性直接获得， O(1) |
listNextNode|返回给定节点的后置节点|后置节点可以通过节点的 next 属性直接获得， O(1) |
listNodeValue|返回给定节点目前正在保存的值|节点值可以通过节点的 value 属性直接获得， O(1) |
listCreate|创建一个不包含任何节点的新链表|O(1)|
listAddNodeHead|将一个包含给定值的新节点添加到给定链表的表头|O(1)|
listAddNodeTail|将一个包含给定值的新节点添加到给定链表的表尾|O(1)|
listInsertNode|将一个包含给定值的新节点添加到给定节点的之前或者之后|O(1)|
listSearchKey|查找并返回链表中包含给定值的节点|O(N) ， N 为链表长度|
listIndex|返回链表在给定索引上的节点|O(N) ， N 为链表长度|
listDelNode|从链表中删除给定节点|O(1) |
listRotate|将链表的表尾节点弹出，然后将被弹出的节点插入到链表的表头， 成为新的表头节点|O(1)|
listDup|复制一个给定链表的副本|O(N) ， N 为链表长度|
listRelease|释放给定链表，以及链表中的所有节点|O(N) ， N 为链表长度|


## 第8章.对象
### 简述
5种主要数据结构:  
简单动态字符串(SDS)、双端链表、字典、压缩列表、整数集合。  

Redis基于这些数据结构创建了一个对象系统，包含:字符串对象、列表对象、哈希对象、集合对象和有序对象这五种类型对象，没种对象用到至少一种上述主要数据结构  

### 对象结构:

```
typedef strict redisObject{
    //类型
    unsigned type:4;
    
    //编码
    unsigned encoding:4;
    
    //指向底层实现数据结构的指针
    void *ptr;
}robj;
```

称呼一个键为"...键":指的是其值为"...对象"
类型常量|对象的名称|创建方式|type属性值|type命令输出|
| --- | --- | --- |---|---|
REDIS_STRING|字符串对象|SET ... ...|REDIS_STRING|"string"|
REDIS_LIST|列表对象|RPUSH ... ...|REDIS_LIST|"list"|
REDIS_HASH|哈希对象|HMSET ... ...|	REDIS_HASH|	"hash"|
REDIS_SET|集合对象|SADD ... ...|REDIS_SET|"set"|
REDIS_ZSET|有序集合对象|ZADD ... ...|REDIS_ZSET|"zset"|

### 编码、底层实现
