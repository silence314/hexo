---
title: 记一次难得的jvm内存分析实战
date: 2020-09-09 19:34:02
tags: 
  - JVM
  - JVM内存分析
description: 一次线上内存问题的分析回顾
updated: 2020-09-11 14:15:03 
keywords: 记录
cover: /blogImg/top.jpg
categories: 工作记录
typora-root-url: ../../themes/butterfly/source
---

# 问题产生

有一个需求是需要统计一些历史数据，因为数量比较大，一个老师下面平均有一两万条数据，每次要统计1.5万左右的老师一次获取的总数据量大概在2000+万，每周一执行，每次获取2000个老师。

在第一次执行的时候，前几批两千人没有报警，在其中一批执行时，线上环境报警，查看监控后发现老年代内存增高，机器内存占用增高，GC频繁且停顿时间长。

![内存监控](/blogImg/内存监控.png)

## 查找代码问题

当时查看了定时任务执行日志，发现这波2000老师从hive拉取数据共700万条，拉取至内存后老年代内存仍然在上涨，而数据获取后应该经过简单判断后插入mysql，代码中没有新对象产生。但是由于之前是将所有数据查出后再插入mysql，这里可能有一些内存的浪费，将代码进行了如下调整：

```java
// 调整前，先将全部数据查出，到informationList里
if (CollectionUtils.isNotEmpty(informationList)) {
                List<List<HistoryCourseInformation>> lists = Lists.partition(informationList, 插入数量);
                for (List list1 : lists) {
                    // 批量插入mysql
                }
            }
```

```java
// 调整后
int index = 0；
where (resultSet.next()) {
	// 遍历hive查询结果，维护一个计数器index，每查一条计数器++
	index++；
	// 达到插入数量后，informationList批量插入，然后清空informationList
	if (index % 插入数量 == 0) {
		//批量插入
        informationList.clear();
	}
}
```

调整后进行测试，问题仍然存在，内存飙高最后会OOM。而且确认了连接hive的Statement不传参数时默认值为TYPE_FORWARD_ONLY和CONCUR_READ_ONLY，按理说数据查出后内存使用会接近稳定，遍历全部查出的结果，这时大概排除了自己代码的问题，只能内存分析了。

## 内存分析

因为公司在pre容器环境较难进行文件下载，需要联系运维，所以在任务运行时，选择一个GC前后，先用jmap -histo 查看了两次内存大致占用，结果如图：

![](/blogImg/GC前.png)

![](/blogImg/GC后.png)

可以看出大部分类型数量和占用都是下降的，char[]却在上升。还是不能确定是哪里产生的问题。只能生成dump文件后联系运维帮忙下载文件，然后本地分析了。

本地mat查看结果很明显了，如图：

![mat内存截图](/blogImg/mat内存截图.png)

这里明显看出是美团的cat进行sql上报，占用了大量内存，因为公司相关中间件是在美团cat基础上开发的，所以也用到了他们的组件，而我们执行的sql是上万条数据的批量插入sql，sql过长，而cat客户端异步上报的原因，而公司架构组也没有对长数据进行截断，导致占用了大量系统内存，最后OOM。

这里再提一下什么情况下什么对象能进入老年代：

1. 大对象：所谓的大对象是指需要大量连续内存空间的java对象，最典型的大对象就是那种很长的字符串以及数组，大对象对虚拟机的内存分配就是坏消息，尤其是一些朝生夕灭的短命大对象，写程序时应避免。
2. 长期存活的对象：虚拟机给每个对象定义了一个对象年龄(Age)计数器，如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并且对象年龄设为1,。对象在Survivor区中每熬过一次Minor GC，年龄就增加1，当他的年龄增加到一定程度(默认是15岁)， 就将会被晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数-XX:MaxTenuringThreshold设置。
3. 动态对象年龄判定：为了能更好地适应不同程度的内存状况，虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升到老年代，如果在Survivor空间中相同年龄的所有对象大小的总和大于Survivor空间的一半，年龄大于或等于年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。

发现了问题原因而且还不是自己的锅这种感觉最爽了，联系了架构组后，关闭项目cat服务进测试，服务无异常，最后架构组同学会在下一个版本进行优化，对于过长的数据上报进行截取或者过滤，彻底解决这个问题。

# 总结

问题本身解决过程没有很复杂，只是开始没考虑有其他因素的影响，在自己代码上纠结了半天，最后内存分析很容易发现问题原因。感觉能真遇到这种问题并且进行问题查找优化还是挺难得的。

本来在内网发的内容有关于vkcat代码的分析，不过发出来我还是把相关内容删掉了，下面总结一些别的相关知识。

## String, StringBuilder, StringBuffer

### String

通过上面mat截图可以看到，最后存放超长的SQL数据的是String，我们也知道String正常情况下在java中是有长度限制的，String源码：

```java
public String(char value[], int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= value.length) {
                this.value = "".value;
                return;
            }
        }
        // Note: offset or count might be near -1>>>1.
        if (offset > value.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        this.value = Arrays.copyOfRange(value, offset, offset+count);
    }
```

可以看到String的构造方法，count是int类型的，所以char value[]中最多可以保存Integer.MAX_VALUE个，大约2147483647。但是曾经我对部分用户id进行过遍历，大概5000个，算起来长度没超过，但是直接赋值的话编译就报错了。

当我们使用字符串字面量直接定义String的时候，是会把字符串在常量池中存储一份的。那么上面提到的65534其实是常量池的限制。 常量池中的每一种数据项也有自己的类型。Java中的UTF-8编码的Unicode字符串在常量池中以CONSTANT_Utf8类型表示。 CONSTANTUtf8info是一个CONSTANTUtf8类型的常量池数据项，它存储的是一个常量字符串。常量池中的所有字面量几乎都是通过CONSTANTUtf8info描述的。CONSTANTUtf8_info的定义如下：

```java
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
}
```

u2是无符号的16位整数，因此理论上允许的的最大长度是2^16=65536。而 java class 文件是使用一种变体UTF-8格式来存放字符的，null 值使用两个 字节来表示，因此只剩下 65536－ 2 ＝ 65534个字节。 关于这一点，在the class file format spec中也有明确说明：

> The length of field and method names, field and method descriptors, and other constant string values is limited to 65535 characters by the 16-bit unsigned length item of the CONSTANTUtf8info structure (§4.4.7). Note that the limit is on the number of bytes in the encoding and not on the number of encoded characters. UTF-8 encodes some characters using two or three bytes. Thus, strings incorporating multibyte characters are further constrained.

### 也就是说，在Java中，所有需要保存在常量池中的数据，长度最大不能超过65535，这当然也包括字符串的定义。

那么，为什么cat里的字符串为什么能这么长呢，

上面提到的这种String长度的限制是编译期的限制，也就是使用String s= "";这种字面值方式定义的时候才会有的限制。看了cat源码之后发现，这个String是从StringBuffer逐渐累加起来的长度，所以也正常出现了这么长的String。

### StingBuffer和StringBuilder

两者其实并没有什么差别，只有StingBuffer加了同步锁，所以他是线程安全的，也因此效率不如StringBuilder