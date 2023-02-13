---
title: redis五大数据类型
tags:
  - Redis
typora-root-url: ../../themes/butterfly/source
date: 2020-10-19 22:21:00
description: 总结一下redis五中数据类型
cover: /blogImg/Redis对象系统模型.png
categories: Redis
---

Redis 是一个高性能的分布式内存型数据库，在国内外各大互联网公司中都有着广泛的使用，即使是一些非互联网公司中也有着非常重要的适用场景，所以对 Redis 的掌握也成为后端工程师必备的基础技能，在面试中，Redis早已成为老生常谈的话题，而在实际工作中，我们更是每时每刻都需要和 Redis 打交道。因此熟练的掌握Redis技术栈至关重要！

# Redis对象类型和编码

Redis中的每个对象都由一个redisObject结构表示,该结构中和保存数据有关的三个属性分别是type属性、 encoding属性和ptr属性:

  Redis使用对象来表示数据库中的键和值,每次当我们在Redis的数据库中新创建一个键值对时,我们至少会创建两个对象,一个对象用作键值对的健(键对象),另一个对象用作键值对的值(值对象)。

```c++
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
/*
 * Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
typedef struct redisObject {

    // 类型
    unsigned type:4;
    // 对齐位
    unsigned notused:2;
    // 编码方式
    unsigned encoding:4;
    // LRU 时间（相对于 server.lruclock）
    unsigned lru:22;
    // 引用计数
    int refcount;
    // 指向对象的值
    void *ptr;
} robj;
```

  其中Redis的键对象都是字符串对象，而Redis的值对象主要有字符串、哈希、列表、集合、有序集合几种。其分别对应的内部编码和底层数据结构如下图所示：

<img src="/blogImg/Redis对象系统模型.png" alt="Redis对象系统模型" style="zoom:50%;" />

Redis中的对象，大都是通过多种数据结构来实现的，为什么会这样设计呢？用一种固定的数据结构来实现，不是更加简单吗？

Redis这样设计有两个好处：

1. 可以自由改进内部编码，而对外的数据结构和命令没有影响，这样一旦开发出更优秀的内部编码，无需改动外部数据结构和命令，例如Redis3.2提供了quicklist，其结合了ziplist和linkedlist两者
	的优势，为列表类型提供了一种更为优秀的内部编码实现，而对外部用户来说基本感知不到。 这一点比较像程序设计中的分层架构。
2. 多种内部编码实现可以在不同场景下发挥各自的优势，从而优化对象在不同场景下的使用效率。例如ziplist比较节省内存，但是在列表元素比较多的情况下，性能会有所下降，这时候Redis会根据配置选项将列表类型的内部实现转换linkedlist。

# 字符串

  字符串对象是 Redis 中最基本的数据类型,也是我们工作中最常用的数据类型。redis中的键都是字符串对象，而且其他几种数据结构都是在字符串对象基础上构建的。字符串对象的值实际可以是字符串、数字、甚至是二进制，最大不能超过512MB 。

![Redis字符串](/blogImg/Redis字符串.jpg)

## 内部实现

  Redis字符串对象底层的数据结构实现主要是int和简单动态字符串SDS(这个字符串，和我们认识的C字符串不太一样，后面会对他进行详细介绍，其通过不同的编码方式映射到不同的数据结构。

![Redis字符串对象](/blogImg/Redis字符串对象.jpg)

字符串对象的内部编码有3种 ：`int`、`raw`和`embstr`。Redis会根据当前值的类型和长度来决定使用哪种编码来实现。

1. 如果一个字符串对象保存的是整数值,并且这个整数值可以用`long`类型来表示,那么字符串对象会将整数值保存在字符串对象结构的`ptr`属性里面(将`void*`转换成`1ong`),并将字符串对象的编码设置为`int`。

![int型字符串](/blogImg/int型字符串.jpg)

2. 如果字符串对象保存的是一个字符串值,并且这个字符串值的长度大于44字节（Redis3.2，之前是39字节）,那么字符串对象将使用一个简单动态字符串(SDS)来保存这个字符串值,并将对象的编码设置为`raw`。

![raw型字符串](/blogImg/raw型字符串.jpg)

3. 如果字符串对象保存的是一个字符串值,并且这个字符申值的长度小于等于44字节（Redis3.2，之前是39字节），那么字符串对象将使用一个简单动态字符串(SDS)来保存这个字符串值，并将对象的编码设置为`embstr`

![embstr](/blogImg/embstr.jpg)

**为什么会选择44作为两种编码的分界点？在3.2版本之前为什么是39？这两个值是怎么得出来的呢？**

**1） 计算RedisObject占用的字节大小**

```c++
struct RedisObject {
    int4 type; // 4bits
    int4 encoding; // 4bits
    int24 lru; // 24bits
    int32 refcount; // 4bytes = 32bits
    void *ptr; // 8bytes，64-bit system
}
```

- type: 不同的redis对象会有不同的数据类型(string、list、hash等)，type记录类型，会用到 **4bits** 。
- encoding：存储编码形式，用 **4bits** 。
- lru：用 **24bits** 记录对象的LRU信息。
- refcount：引用计数器，用到 **32bits** 。
- *ptr：指针指向对象的具体内容，需要 **64bits** 。

计算： 4 + 4 + 24 + 32 + 64 = 128bits =  **16bytes**

第一步就完成了，RedisObject对象头信息会占用 **16字节** 的大小，这个大小通常是固定不变的.

**2) sds占用字节大小计算**

**旧版本：**

```c++
struct SDS {
    unsigned int capacity; // 4byte
    unsigned int len; // 4byte
    byte[] content; // 内联数组，长度为 capacity
}
```

这里的 **unsigned int**  一个4字节，加起来是8字节.

内存分配器jemalloc分配的内存如果超出了64个字节就认为是一个大字符串，就会用到raw编码。

前面提到 SDS 结构体中的 content 的字符串是以字节\0结尾的字符串，之所以多出这样一个字节，是为了便于直接使用 glibc 的字符串处理函数，以及为了便于字符串的调试打印输出。所以我们还要减去1字节  **64byte - 16byte - 8byte - 1byte = 39byte**

**新版本：**

```c++
struct SDS {
    int8 capacity; // 1byte
    int8 len; // 1byte
    int8 flags; // 1byte
    byte[] content; // 内联数组，长度为 capacity
}
```

这里unsigned int 变成了uint8_t、uint16_t.的形式，还加了一个char flags标识，总共只用了3个字节的大小。相当于优化了sds的内存使用，相应的用于存储字符串的内存就会变大。

![emstr数据占用](/blogImg/emstr数据占用.png)

**64byte - 16byte -3byte -1byte = 44byte** 。

**总结：**

所以，redis 3.2版本之后embstr最大能容纳的字符串长度是44，之前是39。长度变化的原因是SDS中内存的优化。

`embstr`编码是专门用于保存短字符串的一种优化编码方式，我们可以看到`embstr`和`raw`编码都会使用`SDS`来保存值，但不同之处在于`embstr`会通过一次内存分配函数来分配一块连续的内存空间来保存`redisObject`和`SDS`。而`raw`编码会通过调用两次内存分配函数来分别分配两块空间来保存`redisObject`和`SDS`。Redis这样做会有很多好处。

- `embstr`编码将创建字符串对象所需的内存分配次数从raw编码的两次降低为一次
- 释放 `embstr`编码的字符串对象同样只需要调用一次内存释放函数
- 因为`embstr`编码的字符串对象的所有数据都保存在一块连续的内存里面可以更好的利用CPU缓存提升性能。

  Redis中根据数据类型和长度来使用不同的编码和数据结构存储存在于Redis中的每一种对象类型上。其这种小细节上的优化令我叹服不止，后续我们会看到Redis中到处都是这种内存与性能上的小细节优化！

## 常见命令

### 设置值

```shell
redis> set testKey testValue
OK
set key value [ex seconds] [px milliseconds] [nx|xx]
```

- `ex seconds`：为键设置秒级过期时间。

	如命令:`set username xiaoming ex 100`相当于执行下面两条命令

	```shell
	SET username xiaoming 
	EXPIRE username 100
	```

	`set key value [ex seconds]`操作是原子性的，相比连续执行上面两个命令，它更快。

- `px milliseconds`：为键设置毫秒级过期时间。

- `nx`：键必须不存在，才可以设置成功，用于添加。

	```shell
	//mykey 不存在
	redis> set mykey "Hello" nx
	(integer) 1
	//mykey 已经存在
	redis> set mykey "World" nx
	(integer) 0
	redis> GET mykey
	"Hello"
	redis>
	```

	由于`set key value nx`同样是原子性的操作，因此可以作为分布式锁的一种实现方案。

- `xx`：与nx相反，键必须存在，才可以设置成功，用于更新

以上几个命令的替代命令是`SETNX`, `SETEX`,`PSETEX`，但是由于`SET`命令加上选项已经可以完全取代`SETNX`, `SETEX`,`PSETEX`的功能，所以在将来的版本中，redis可能会不推荐使用并且最终抛弃这几个命令。

### 获取值

```shell
get key
```

返回`key`的`value`。如果key不存在，返回特殊值`nil`。如果`key`的`value`不是string，就返回错误，因为`GET`只处理string类型的`values`。

```shell 
redis> GET nokey
(nil)
redis> SET mykey "Hello World"
OK
redis> GET mykey
"Hello World"
```

### 计数

```shell
incr key
```

对存储在指定`key`的数值执行原子的加1操作。

如果指定的key不存在，那么在执行incr操作之前，会先将它的值设定为`0`。

如果指定的key中存储的值不是字符串类型（fix：）或者存储的字符串类型不能表示为一个整数，

那么执行这个命令时服务器会返回一个错误(eq:(error) ERR value is not an integer or out of range)。

这个操作仅限于64位的有符号整型数据。

**注意**: 由于redis并没有一个明确的类型来表示整型数据，所以这个操作是一个字符串操作。

执行这个操作的时候，key对应存储的字符串被解析为10进制的**64位有符号整型数据**。

事实上，Redis 内部采用整数形式（Integer representation）来存储对应的整数值，所以对该类字符串值实际上是用整数保存，也就不存在存储整数的字符串表示（String representation）所带来的额外消耗。

```shell
redis> SET mykey "1"
OK
redis> INCR mykey
(integer) 2
redis> GET mykey
"3"
redis>
```

除了`incr`命令， Redis提供了`decr（自减）` 、 `incrby（自增指定数字）` 、`decrby（自减指定数字）` 、 `incrbyfloat（自增浮点数）`

### 其他

| 命令                                                  | 描述                                                         | 时间复杂度            |
| :---------------------------------------------------- | :----------------------------------------------------------- |
| `set key value [ex seconds] [px milliseconds] [nxxx]` | 设置值                                                       | O(1)                  |      |
| `get key`                                             | 获取值                                                       | O(1)                  |      |
| `del key [key ...]`                                   | 删除key                                                      | O(N)(N是键的个数)     |      |
| `mset key [key value ...]`                            | 批量设置值                                                   | O(N)(N是键的个数)     |      |
| `mget key [key ...]`                                  | 批量获取值                                                   | O(N)(N是键的个数)     |      |
| `incr key`                                            | 将 key 中储存的数字值增一                                    | O(1)                  |      |
| `decr key`                                            | 将 key 中储存的数字值减一                                    | O(1)                  |      |
| `incrby key increment`                                | 将 key 所储存的值加上给定的增量值（increment）               | O(1)                  |
| `decrby key increment`                                | key 所储存的值减去给定的减量值（decrement）                  | O(1)                  |      |
| `incrbyfloat key increment`                           | 将 key 所储存的值加上给定的浮点增量值（increment）           | O(1)                  |
| `append key value`                                    | 如果 key 已经存在并且是一个字符串， APPEND 命令将指定的 value 追加到该 key 原来值（value）的末尾 | O(1)                  |      |
| `strlen key`                                          | 返回 key 所储存的字符串值的长度。                            | O(1)                  |      |
| `setrange key offset value`                           | 用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始 | O(1)                  |      |
| `getrange key start end`                              | 返回 key 中字符串值的子字符                                  | O(N)(N是字符串的长度) |

## 常用场景

  reids字符串的使用场景应该是最为广泛的，甚至有些对redis其它几种对象不太熟悉的人，基本所有场景都会使用字符串(序列化一下直接扔进去)。在众多的使用场景中总结一下大概分以下几种。

### 作为缓存层

Redis经常作为缓存层，来缓存一些热点数据。来加速读写性能从而降低后端的压力。一般在读取数据的时候会先从Redis中读取，如果Redis中没有，再从数据库中读取。在Redis作为缓存层使用的时候，必须注意一些问题，如：缓存穿透、雪崩以及缓存更新问题……

### 计数器\限速器\分布式系统ID

计数器\限速器\分布式ID等主要是利用Redis字符串自增自减的特性。

- 计数器：经常可以被用来做计数器，如微博的评论数、点赞数、分享数，抖音作品的收藏数，京东商品的销售量、评价数等。
- 限速器：如验证码接口访问频率限制，用户登陆时需要让用户输入手机验证码，从而确定是否是用户本人，但是为了短信接口不被频繁访问，会限制用户每分钟获取验证码的频率，例如一分钟不能超过5次。
- 分布式ID：由于Redis自增自减的操作是原子性的因此也经常在分布式系统中用来生成唯一的订单号、序列号等。

### 分布式系统共享session

通常在单体系统中，Web服务将会用户的Session信息（例如用户登录信息）保存在自己的服务器中。但是在分布式系统中，这样做会有问题。因为分布式系统通常有很多个服务，每个服务又会同时部署在多台机器上，通过负载均衡机制将将用户的访问均衡到不同服务器上。这个时候用户的请求可能分发到不同的服务器上，从而导致用户登录保存Session是在一台服务器上，而读取Session是在另一台服务器上因此会读不到Session。

  这种问题通常的做法是把Session存到一个公共的地方，让每个Web服务，都去这个公共的地方存取Session。而Redis就可以是这个公共的地方。(数据库、memecache等都可以各有优缺点)。

### 二进制存储

  由于Redis字符串可以存储二进制数据的特性，因此也可以用来存储一些二进制数据。如图片、 音频、 视频等。

# 哈希

哈希在很多编程语言中都有着很广泛的应用，而在Redis中也是如此，在redis中，哈希类型是指Redis键值对中的值本身又是一个键值对结构，形如`value=[{field1，value1}，...{fieldN，valueN}]`，其与Redis字符串对象的区别如下图所示:

![Redis-Hash](/blogImg/Redis-Hash.png)

## 内部编码

哈希类型的内部编码有两种：ziplist(压缩列表)，hashtable(哈希表)。只有当存储的数据量比较小的情况下，Redis 才使用压缩列表来实现字典类型。具体需要满足两个条件：

- 当哈希类型元素个数小于hash-max-ziplist-entries配置（默认512个）
- 所有值都小于hash-max-ziplist-value配置（默认64字节）
	`ziplist`使用更加紧凑的结构实现多个元素的连续存储，所以在节省内存方面比`hashtable`更加优秀。当哈希类型无法满足`ziplist`的条件时，Redis会使用`hashtable`作为哈希的内部实现，因为此时`ziplist`的读写效率会下降，而`hashtable`的读写时间复杂度为O（1）。

## 常见命令

| 命令                                            | 说明                                           | 时间复杂度                 |
| :---------------------------------------------- | :--------------------------------------------- | :------------------------- |
| HDEL key field                                  | 删除一个或多个Hash的field                      | O(N) N是被删除的字段数量。 |
| HEXISTS key field                               | 判断field是否存在于hash中                      | O(1)                       |
| HGET key field                                  | 获取hash中field的值                            | O(1)                       |
| HGETALL key                                     | 从hash中读取全部的域和值                       | O(N) N是Hash的长度         |
| HINCRBY key field increment                     | 将hash中指定域的值增加给定的数字               | O(1)                       |
| HINCRBYFLOAT key field increment                | 将hash中指定域的值增加给定的浮点数             | O(1)                       |
| HKEYS key                                       | 获取hash的所有字段                             | O(N) N是Hash的长度         |
| HLEN key                                        | 获取hash里所有字段的数量                       | O(1)                       |
| HMGET key field                                 | 获取hash里面指定字段的值                       | O(N) N是请求的字段数       |
| HMSET key field value                           | 设置hash字段值                                 | O(N) N是设置的字段数       |
| HSET key field value                            | 设置hash里面一个字段的值                       | O(1)                       |
| HSETNX key field value                          | 设置hash的一个字段，只有当这个字段不存在时有效 | O(1)                       |
| HSTRLEN key field                               | 获取hash里面指定field的长度                    | O(1)                       |
| HVALS key                                       | 获得hash的所有值                               | O(N) N是Hash的长度         |
| HSCAN key cursor [MATCH pattern\] [COUNT count] | 迭代hash里面的元素                             |                            |

## 适用场景

### 存储对象

 Redis哈希对象常常用来缓存一些对象信息，如用户信息、商品信息、配置信息等。

我们以用户信息为例,它在关系型数据库中的结构是这样的

| uid  | name  | age  |
| :--- | :---- | :--- |
| 1    | Tom   | 15   |
| 2    | Jerry | 13   |

而使用Redis Hash存储其结构如下图:

![Redis-Hash (1)](/blogImg/Redis-Hash结构.png)

相比较于使用Redis字符串存储，其有以下几个优缺点:

1. 原生字符串每个属性一个键。

	```
	set user:1:name Tom
	set user:1:age 15
	```

	优点：简单直观，每个属性都支持更新操作。
	缺点：占用过多的键，内存占用量较大，同时用户信息内聚性比较差，所以此种方案一般不会在生产环境使用。

2. 序列化字符串后，将用户信息序列化后用一个键保存

	```
	set user:1 serialize(userInfo)
	```

优点：简化编程，如果合理的使用序列化可以提高内存的使用效率。
缺点：序列化和反序列化有一定的开销，同时每次更新属性都需要把全部数据取出进行反序列化，更新后再序列化到Redis中。

1. 哈希类型：每个用户属性使用一对field-value，但是只用一个键保存。

	```
	hmset user:1 name Tom age 15
	```

	优点：简单直观，如果使用合理可以减少内存空间的使用。
	缺点：要控制哈希在ziplist和hashtable两种内部编码的转换，hashtable会消耗更多内存。

### 购物车

很多电商网站都会使用 cookie实现购物车，也就是将整个购物车都存储到 cookie里面。这种做法的一大**优点:**无须对数据库进行写入就可以实现购物车功能,这种方式大大提高了购物车的性能，而**缺点**则是程序需要重新解析和验证( validate) cookie,确保cookie的格式正确,并且包含的商品都是真正可购买的商品。cookie购物车还有一个**缺点**:因为浏览器每次发送请求都会连 cookie一起发送,所以如果购物车cookie的体积比较大,那么请求发送和处理的速度可能会有所降低。

  购物车的定义非常简单:我们以每个用户的用户ID(或者CookieId)作为Redis的Key,每个用户的购物车都是一个哈希表,这个哈希表存储了商品ID与商品订购数量之间的映射。在商品的订购数量出现变化时,我们操作Redis哈希对购物车进行更新:

如果用户订购某件商品的数量大于0,那么程序会将这件商品的ID以及用户订购该商品的数量添加到散列里面。

```
//用户1 商品1 数量1
127.0.0.1:6379> HSET uid:1 pid:1 1
(integer) 1 //返回值0代表改field在哈希表中不存在，为新增的field
```

如果用户购买的商品已经存在于散列里面,那么新的订购数量会覆盖已有的订购数量;

```
//用户1 商品1 数量5
127.0.0.1:6379> HSET uid:1 pid:1 5
(integer) 0 //返回值0代表改field在哈希表中已经存在
```

相反地,如果用户订购某件商品的数量不大于0,那么程序将从散列里面移除该条目。

```
//用户1 商品1
127.0.0.1:6379> HDEL uid:1 pid:2
(integer) 1
```

### 计数器

Redis 哈希表作为计数器的使用也非常广泛。它常常被用在记录网站每一天、一月、一年的访问数量。每一次访问，我们在对应的field上自增1

```
//记录我的
127.0.0.1:6379> HINCRBY MyBlog  202001 1
(integer) 1
127.0.0.1:6379> HINCRBY MyBlog  202001 1
(integer) 2
127.0.0.1:6379> HINCRBY MyBlog  202002 1
(integer) 1
127.0.0.1:6379> HINCRBY MyBlog  202002 1
(integer) 2
```

也经常被用在记录商品的好评数量，差评数量上

```
127.0.0.1:6379> HINCRBY pid:1  Good 1
(integer) 1
127.0.0.1:6379> HINCRBY pid:1  Good 1
(integer) 2
127.0.0.1:6379> HINCRBY pid:1  bad  1
(integer) 1
```

也可以实时记录当天的在线的人数。

```
//有人登陆
127.0.0.1:6379> HINCRBY MySite  20200310 1
(integer) 1
//有人登陆
127.0.0.1:6379> HINCRBY MySite  20200310 1
(integer) 2
//有人登出
127.0.0.1:6379> HINCRBY MySite  20200310 -1
(integer) 1
```

# 列表

  列表（list）类型是用来存储多个有序的字符串，列表中的每个字符串称为元素(element)，一个列表最多可以存储2^32-1个元素。在Redis中，可以对列表两端插入（push）和弹出（pop），还可以获取指定范围的元素列表、获取指定索引下标的元素等。列表是一种比较灵活的数据结构，它可以充当栈和队列的角色，在实际开发上有很多应用场景。

![List](/blogImg/List.png)

列表类型有两个特点：

1. 列表中的元素是有序的，这就意味着可以通过索引下标获取某个元素或者某个范围内的元素列表。
2. 列表中的元素可以是重复的.

## 内部实现

在Redis3.2版本以前列表类型的内部编码有两种。

- `ziplist（压缩列表）`：当列表的元素个数小于list-max-ziplist-entries配置（默认512个），同时列表中每个元素的值都小于list-max-ziplist-value配置时（默认64字节），Redis会选用ziplist来作为列表的内部实现来减少内存的使用。
- `linkedlist（链表）`：当列表类型无法满足ziplist的条件时，Redis会使用linkedlist作为列表的内部实现。

而在Redis3.2版本开始对列表数据结构进行了改造，使用 quicklist 代替了 ziplist 和 linkedlist.

## 常用命令

| 命令                                  | 说明                                                         | 时间复杂度 |
| :------------------------------------ | :----------------------------------------------------------- | :--------- |
| BLPOP key                             | 删除，并获得该列表中的第一元素，或阻塞，直到有一个可用       | O(1)       |
| BRPOP key                             | 删除，并获得该列表中的最后一个元素，或阻塞，直到有一个可用   | O(1)       |
| BRPOPLPUSH source destination timeout | 弹出一个列表的值，将它推到另一个列表，并返回它;或阻塞，直到有一个可用 | O(1)       |
| LINDEX key index                      | 获取一个元素，通过其索引列表                                 | O(N)       |
| LINSERT key BEFORE                    | AFTER pivot value在列表中的另一个元素之前或之后插入一个元素  | O(N)       |
| LLEN key                              | 获得队列(List)的长度                                         | O(1)       |
| LPOP key                              | 从队列的左边出队一个元素                                     | O(1)       |
| LPUSH key value                       | 从队列的左边入队一个或多个元素                               | O(1)       |
| LPUSHX key value                      | 当队列存在时，从队到左边入队一个元素                         | O(1)       |
| LRANGE key start stop                 | 从列表中获取指定返回的元素                                   | O(S+N)     |
| LREM key count value                  | 从列表中删除元素                                             | O(N)       |
| LSET key index value                  | 设置队列里面一个元素的值                                     | O(N)       |
| LTRIM key start stop                  | 修剪到指定范围内的清单                                       | O(N)       |
| RPOP key                              | 从队列的右边出队一个元                                       | O(1)       |
| RPOPLPUSH source destination          | 删除列表中的最后一个元素，将其追加到另一个列表               | O(1)       |
| RPUSH key value                       | 从队列的右边入队一个元素                                     | O(1)       |
| RPUSHX key value                      | 从队列的右边入队一个元素，仅队列存在时有效                   | O(1)       |

## 适用场景

### 消息队列

列表类型可以使用 rpush 实现先进先出的功能，同时又可以使用 lpop 轻松的弹出（查询并删除）第一个元素，所以列表类型可以用来实现消息队列

### 文章(商品等)列表

我们以博客站点为例，当用户和文章都越来越多时，为了加快程序的响应速度，我们可以把用户自己的文章存入到 List 中，因为 List 是有序的结构，所以这样又可以完美的实现分页功能，从而加速了程序的响应速度。

1. 每篇文章我们使用**哈希**结构存储，例如每篇文章有3个属性title、timestamp、content

	```
	hmset acticle:1 title xx timestamp 1476536196 content xxxx
	...
	hmset acticle:k title yy timestamp 1476512536 content yyyy
	...
	```

2. 向用户文章列表添加文章，user：{id}：articles作为用户文章列表的键：

	```
	lpush user:1:acticles article:1 article3
	...
	lpush
	...
	```

3. 分页获取用户文章列表，例如下面伪代码获取用户id=1的前10篇文章

	```
	articles = lrange user:1:articles 0 9
	for article in {articles}
	{
		hgetall {article}
	}
	```

**注意:**使用列表类型保存和获取文章列表会存在两个问题。

- 如果每次分页获取的文章个数较多，需要执行多次hgetall操作，此时可以考虑使用Pipeline批量获取，或者考虑将文章数据序列化为字符串类型，使用mget批量获取。
- 分页获取文章列表时，lrange命令在列表两端性能较好，但是如果列表较大，获取列表中间范围的元素性能会变差，此时可以考虑将列表做二级拆分，或者使用Redis3.2的quicklist内部编码实现，它结合ziplist和linkedlist的特点，获取列表中间范围的元素时也可以高效完成。

关于列表的使用场景可参考以下几个命令组合：

- lpush+lpop=Stack（栈）
- lpush+rpop=Queue（队列）
- lpush+ltrim=Capped Collection（有限集合）
- lpush+brpop=Message Queue（消息队列）

# 集合

集合类型 (Set) 是一个无序并唯一的键值集合。它的存储顺序不会按照插入的先后顺序进行存储。

集合类型和列表类型的区别如下：

- 列表可以存储重复元素，集合只能存储非重复元素；
- 列表是按照元素的先后顺序存储元素的，而集合则是无序方式存储元素的。

一个集合最多可以存储2^32-1个元素。Redis除了支持集合内的增删改查，同时还支持多个集合取交集、并集、差集，合理地使用好集合类型，能在实际开发中解决很多实际问题。

![集合](/blogImg/集合.png)

## 内部实现

集合类型的内部编码有两种：

- intset(整数集合):当集合中的元素都是整数且元素个数小于set-maxintset-entries配置（默认512个）时，Redis会选用intset来作为集合的内部实现，从而减少内存的使用。
- hashtable(哈希表):当集合类型无法满足intset的条件时，Redis会使用hashtable作为集合的内部实现。

## 常用命令

| 命令                                            | 说明                                           | 时间复杂度 |
| :---------------------------------------------- | :--------------------------------------------- | :--------- |
| SADD key member                                 | 添加一个或者多个元素到集合(set)里              | O(N)       |
| SCARD key                                       | 获取集合里面的元素数量                         | O(1)       |
| SDIFF key                                       | 获得队列不存在的元素                           | O(N)       |
| SDIFFSTORE destination key                      | 获得队列不存在的元素，并存储在一个关键的结果集 | O(N)       |
| SINTER key                                      | 获得两个集合的交集                             | O(N*M)     |
| SINTERSTORE destination key                     | 获得两个集合的交集，并存储在一个关键的结果集   | O(N*M)     |
| SISMEMBER key member                            | 确定一个给定的值是一个集合的成员               | O(1)       |
| SMEMBERS key                                    | 获取集合里面的所有元素                         | O(N)       |
| SMOVE source destination member                 | 移动集合里面的一个元素到另一个集合             | O(1)       |
| SPOP key                                        | 删除并获取一个集合里面的元素                   | O(1)       |
| SRANDMEMBER key                                 | 从集合里面随机获取一个元素                     |            |
| SREM key member                                 | 从集合里删除一个或多个元素                     | O(N)       |
| SUNION key                                      | 添加多个set元素                                | O(N)       |
| SUNIONSTORE destination key                     | 合并set元素，并将结果存入新的set里面           | O(N)       |
| SSCAN key cursor [MATCH pattern\] [COUNT count] | 迭代set里面的元素                              | O(1)       |

## 适用场景

通过上文，我们可以知道集合的主要几个特性，无序、不可重复、支持并交差等操作。因此集合类型比较适合用来数据去重和保障数据的唯一性，还可以用来统计多个集合的交集、错集和并集等，当我们存储的数据是无序并且需要去重的情况下，比较适合使用集合类型进行存储。

### 标签系统

集合类型比较典型的使用场景是标签（tag）。

1. 给用户添加标签。

	```
	sadd user:1:tags tag1 tag2 tag5
	sadd user:2:tags tag2 tag3 tag5
	...
	sadd user:k:tags tag1 tag2 tag4
	...
	```

1. 给标签添加用户

	```
	sadd tag1:users user:1 user:3
	sadd tag2:users user:1 user:2 user:3
	...
	sadd tagk:users user:1 user:2
	...
	```

2. 使用sinter命令，可以来计算用户共同感兴趣的标签

	```
	sinter user:1:tags user:2:tags
	```

这种标签系统在电商系统、社交系统、视频网站，图书网站，旅游网站等都有着广泛的应用。例如一个用户可能对娱乐、体育比较感兴趣，另一个用户可能对历史、新闻比较感兴趣，这些兴趣点就是标签。有了这些数据就可以得到喜欢同一个标签的人，以及用户的共同喜好的标签，这些数据对于用户体验以及增强用户黏度比较重要。例如一个社交系统可以根据用户的标签进行好友的推荐，已经用户感兴趣的新闻的推荐等，一个电子商务的网站会对不同标签的用户做不同类型的推荐，比如对数码产品比较感兴趣的人，在各个页面或者通过邮件的形式给他们推荐最新的数码产品，通常会为网站带来更多的利益。

### 抽奖系统

Redis集合的 SPOP(随机移除并返回集合中一个或多个元素) 和 SRANDMEMBER(随机返回集合中一个或多个元素) 命令可以帮助我们实现一个抽奖系统

如果允许重复中奖，可以使用SRANDMEMBER 命令

```
//添加抽奖名单
127.0.0.1:6379> SADD lucky:1 Tom
(integer) 1
127.0.0.1:6379> SADD lucky:1 Jerry
(integer) 1
127.0.0.1:6379> SADD lucky:1 John
(integer) 1
127.0.0.1:6379> SADD lucky:1 Marry
(integer) 1
127.0.0.1:6379> SADD lucky:1 Sean
(integer) 1
127.0.0.1:6379> SADD lucky:1 Lindy
(integer) 1
127.0.0.1:6379> SADD lucky:1 Echo
(integer) 1

//抽取三等奖3个
127.0.0.1:6379> SRANDMEMBER lucky:1 3
1) "John"
2) "Echo"
3) "Lindy"
//抽取二等奖2个
127.0.0.1:6379> SRANDMEMBER lucky:1 2
1) "Sean"
2) "Lindy"
//抽取一等奖1个
127.0.0.1:6379> SRANDMEMBER lucky:1 1
1) "Tom"
```

如果不允许重复中奖，可以使用 SPOP 命令

```
//添加抽奖名单
127.0.0.1:6379> SADD lucky:1 Tom
(integer) 1
127.0.0.1:6379> SADD lucky:1 Jerry
(integer) 1
127.0.0.1:6379> SADD lucky:1 John
(integer) 1
127.0.0.1:6379> SADD lucky:1 Marry
(integer) 1
127.0.0.1:6379> SADD lucky:1 Sean
(integer) 1
127.0.0.1:6379> SADD lucky:1 Lindy
(integer) 1
127.0.0.1:6379> SADD lucky:1 Echo
(integer) 1

//抽取三等奖3个
127.0.0.1:6379> SPOP lucky:1 3
1) "John"
2) "Echo"
3) "Lindy"
//抽取二等奖2个
127.0.0.1:6379> SPOP lucky:1 2
1) "Sean"
2) "Marry"
//抽取一等奖1个
127.0.0.1:6379> SPOP lucky:1 1
1) "Tom"
```

**注意:**

Redis 2.6版本开始SRANDMEMBER命令支持Count参数。

Redis 3.2版本开始SRANDMEMBER命令支持Count参数。

其余低版本一次只能获取一个随机元素。

# 有序集合

有序集合类型 (Sorted Set或ZSet) 相比于集合类型多了一个排序属性 score（分值），对于有序集合 ZSet 来说，每个存储元素相当于有两个值组成的，一个是有序结合的元素值，一个是排序值。有序集合保留了集合不能有重复成员的特性(分值可以重复)，但不同的是，有序集合中的元素可以排序。

![有序集合](/blogImg/有序集合.png)

## 内部实现

有序集合是由 ziplist (压缩列表) 或 skiplist (跳跃表)组成的。

当数据比较少时，有序集合使用的是 ziplist 存储的，有序集合使用 ziplist 格式存储必须满足以下两个条件：

- 有序集合保存的元素个数要小于 128 个；
- 有序集合保存的所有元素成员的长度都必须小于 64 字节。

如果不能满足以上两个条件中的任意一个，有序集合将会使用 skiplist 结构进行存储。

## 常见命令

| 命令                                                         | 说明                                                         | 时间复杂度         |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------- |
| BZPOPMAX key                                                 | 从一个或多个排序集中删除并返回得分最高的成员，或阻塞，直到其中一个可用为止 | O(log(N))          |
| BZPOPMIN key                                                 | 从一个或多个排序集中删除并返回得分最低的成员，或阻塞，直到其中一个可用为止 | O(log(N))          |
| ZADD key [NXXX\] [CH] [INCR] score member                    | 添加到有序set的一个或多个成员，或更新的分数，如果它已经存在  | O(log(N))          |
| ZCARD key                                                    | 获取一个排序的集合中的成员数量                               | O(1)               |
| ZCOUNT key min max                                           | 返回分数范围内的成员数量                                     | O(log(N))          |
| ZINCRBY key increment member                                 | 增量的一名成员在排序设置的评分                               | O(log(N))          |
| ZINTERSTORE                                                  | 相交多个排序集，导致排序的设置存储在一个新的关键             | O(N*K)+O(M*log(M)) |
| ZLEXCOUNT key min max                                        | 返回成员之间的成员数量                                       | O(log(N))          |
| ZPOPMAX key [count\]                                         | 删除并返回排序集中得分最高的成员                             | O(log(N)*M)        |
| ZPOPMIN key [count\]                                         | 删除并返回排序集中得分最低的成员                             | O(log(N)*M)        |
| ZRANGE key start stop [WITHSCORES\]                          | 根据指定的index返回，返回sorted set的成员列表                | O(log(N)+M)        |
| ZRANGEBYLEX key min max [LIMIT offset count\]                | 返回指定成员区间内的成员，按字典正序排列, 分数必须相同。     | O(log(N)+M)        |
| ZREVRANGEBYLEX key max min [LIMIT offset count\]             | 返回指定成员区间内的成员，按字典倒序排列, 分数必须相同       | O(log(N)+M)        |
| ZRANGEBYSCORE key min max [WITHSCORES\] [LIMIT offset count] | 返回有序集合中指定分数区间内的成员，分数由低到高排序。       | O(log(N)+M)        |
| ZRANK key member                                             | 确定在排序集合成员的索引                                     | O(log(N))          |
| ZREM key member [member …\]                                  | 从排序的集合中删除一个或多个成员                             | O(M*log(N))        |
| ZREMRANGEBYLEX key min max                                   | 删除名称按字典由低到高排序成员之间所有成员。                 | O(log(N)+M)        |
| ZREMRANGEBYRANK key start stop                               | 在排序设置的所有成员在给定的索引中删除                       | O(log(N)+M)        |
| ZREMRANGEBYSCORE key min max                                 | 删除一个排序的设置在给定的分数所有成员                       | O(log(N)+M)        |
| ZREVRANGE key start stop [WITHSCORES\]                       | 在排序的设置返回的成员范围，通过索引，下令从分数高到低       | O(log(N)+M)        |
| ZREVRANGEBYSCORE key max min [WITHSCORES\] [LIMIT offset count] | 返回有序集合中指定分数区间内的成员，分数由高到低排序。       | O(log(N)+M)        |
| ZREVRANK key member                                          | 确定指数在排序集的成员，下令从分数高到低                     | O(log(N))          |
| ZSCORE key member                                            | 获取成员在排序设置相关的比分                                 | O(1)               |
| ZUNIONSTORE                                                  | 添加多个排序集和导致排序的设置存储在一个新的关键             | O(N)+O(M log(M))   |
| ZSCAN key cursor [MATCH pattern\] [COUNT count]              | 迭代sorted sets里面的元素                                    | O(1)               |

## 适用场景

### 排行榜

有序集合比较典型的使用场景就是排行榜系统。例如学生成绩的排名。某视频(博客等)网站的用户点赞、播放排名、电商系统中商品的销量排名等。我们以博客点赞为例。

1. 添加用户赞数

例如小编Tom发表了一篇博文，并且获得了10个赞。

```
zadd user:ranking arcticle1 10
```

1. 取消用户赞数

这个时候有一个读者又觉得Tom写的不好，又取消了赞，此时需要将文章的赞数从榜单中减去1，可以使用zincrby。

```
zincrby user:ranking arcticle1 -1
```

1. 查看某篇文章的赞数

```
ZSCORE user:ranking arcticle1
```

1. 展示获取赞数最多的十篇文章

此功能使用zrevrange命令实现：

```
zrevrangebyrank user:ranking  0 9
```

