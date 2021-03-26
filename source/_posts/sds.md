---
title: 简单动态字符串
date: 2021-03-24 10:37:37
mathjax: true
comments: true #是否评论
toc: true #是否显示文章目录
categories: 学习 #分类
headimg: https://cdn.hin.cool/pic/posts/tao.jpg
cover: true
tags: 
    - redis
---

## 简单动态字符串

Redis中字符串的底层实现是简单动态字符串SDS(Simple Dynamic String)，是可以修改的字符串。其中SDS的实现代码在sds.c和sds.h文件中。

### SDS的定义
每个sds.h/sdshdr结构表示一个SDS值，在Redis中SDS有如下五种结构:
```C
struct __attribute__ ((__packed__)) sdshdr5 {
    //低三位表类型，高五位表长度
    unsigned char flags;
    //字节数据，用于保存字符串
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    //buf中存的字符串长度
    uint8_t len;
    //buf分配的空间，不包括终止符/0
    uint8_t alloc;
    //低三位表类型，高五位没有使用
    unsigned char flags;
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; 
    uint16_t alloc;
    unsigned char flags;
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len;
    uint32_t alloc;
    unsigned char flags;
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len;
    uint64_t alloc;
    unsigned char flags;
    char buf[];
};
```
### SDS与C字符串的区别
+ 常数复杂度获取字符串长度。因为C字符串不记录自身长度信息，所以获取一个C字符串的长度，程序必须遍历整个字符串，对遇到的每个字符串进行计数，直到遇到代表字符串结尾的空字符为止，这个操作的复杂度为$O(N)$；和C字符串不同，因为SDS在len属性中纪录了buf属性中保存的字符串的长度，所以获取SDS中保存的字符串长度的复杂度仅为$O(1)$。
+ 杜绝缓冲区溢出。除了获取字符串长度的复杂度高之外，C字符串不记录自身长度带来的另外一个问题是容易造成缓冲区溢出(Buffer Overflow)，例如<string.h>/strcat函数可以将src字符串中的内容拼接到dest字符串的末尾：
```C
char *strcat(char *dest,const char *src);
```
因为C字符串不记录自身的长度，所以strcat假设用户在执行这个函数时，已经为dest分配了足够多的内存，可以容纳src字符串中的所有内容，而一旦这个假设不成立时，就会产生缓冲区溢出；与C字符串不同，SDS的空间分配策略完全杜绝了发生缓冲区溢出的可能性：当SDS API需要对SDS进行修改的时，API会先检查SDS的空间是否满足修改所需要的空间，如果不满足，API会根据SDS空间预分配策略扩展空间，然后才执行实际的修改操作，所以使用SDS既不需要手动修改SDS的空间大小，也不会出现缓冲区溢出问题。
+ 减少字符串修改时带来的内存重分配次数。这是因为SDS使用了空间预分配策略，本文后面章节将详细介绍。
+ 二进制安全。C字符串中字符必须符合某种编码(比如ASCII)，并且除了字符串的末尾之外，字符串里面不能包含空字符，否则最先被程序读入的空字符将被误认为是字符串的结尾，这些限制使得C字符串只能保存文本数据，而不能保存像图片、音频、压缩文件这样的二进制数据；由于SDS使用len属性的值而不是空字符来判断字符串是否结束，所以SDS不仅可以保存文本数据，还可以保存任意格式的二进制数据。
+ 兼容部分C字符串函数

### SDS空间预分配
空间预分配用于优化SDS的字符串增长操作：当SDS的API修改SDS，并且需要对SDS进行空间扩展的时候，SDS的API不仅会为SDS分配修改所必须的空间，还会为SDS分配额外的未使用空间。其中分配额外未使用空间的大小由以下公式决定：
+ 如果对SDS进行修改后，SDS的长度len小于1MB，那么SDS的API会分配和len同样大小的未使用空间。
+ 如果对SDS进行修改后，SDS的长度len大于等于1MB，那么SDS的API会分配1MB的未使用空间。
其Redis源码实现如下：
```C
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t usable;

    /*如果有足够空间，直接返回*/
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    newlen = (len+addlen);
    assert(newlen > len);   /* Catch size_t overflow */
    /*修改后SDS长度len小于1MB，分配2*len大小空间，
    len大小的空间用于保存字符串，len大小的空闲空间*/
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
    /*修改后SDS长度len大于等于1MB，分配len+1MB大小空间，
    len大小的空间用于保存字符串，1MB大小的空闲空间*/
        newlen += SDS_MAX_PREALLOC;

    type = sdsReqType(newlen);

    /**在字符串追加操作时，类型5不能保存空闲空间因此不能使用类型5*/
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    assert(hdrlen + newlen + 1 > len); 
    if (oldtype==type) {
        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /**SDS头改变，需要向前移动字符串，因此不能使用realloc*/
        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    sdssetalloc(s, usable);
    return s;
}
```
### SDS惰性空间释放
惰性空间释放用于优化SDS的字符串缩短操作：当SDS的API缩短SDS保存的字符串时，SDS的API并不立即使用内存重分配来回收缩短后多出来的字节，而是将多出来的字节数纪录下来，并等待以后使用。其中SDS的sdstrim和sdsclear API实现中使用了惰性空间释放策略，其实现源码分别如下：
```C
sds sdstrim(sds s, const char *cset) {
    char *start, *end, *sp, *ep;
    size_t len;

    sp = start = s;
    ep = end = s+sdslen(s)-1;
    while(sp <= end && strchr(cset, *sp)) sp++;
    while(ep > sp && strchr(cset, *ep)) ep--;
    len = (sp > ep) ? 0 : ((ep-sp)+1);
    if (s != sp) memmove(s, sp, len);
    s[len] = '\0';
    sdssetlen(s,len);
    return s;
}
```
```C
void sdsclear(sds s) {
    //将SDS的len属性设置为0
    sdssetlen(s, 0);
    s[0] = '\0';
}
```
### SDS API
SDS的主要操作API如下：
| 函数 | 作用 | 时间复杂度 |
| -- | :--: | :-- |
| sdsnew | 创建一个包含指定C字符串的SDS | $O(N)$，N为指定C字符串的长度 |
| sdsempty | 创建一个不包含任何内容的空SDS | $O(1)$ |
| sdsfree | 释放指定的SDS | $O(N)$，N为被释放SDS的长度 |
| sdslen | 返回SDS已使用空间字节数 | $O(1)$ |
| sdsavail | 返回SDS未使用空间字节数 | $O(1)$ |
| sdsdup | 创建指定SDS的副本 | $O(N)$，N为指定SDS的长度 |
| sdsclear | 清空SDS保存的字符串内容 | $O(1)$ |
| sdscat | 将指定C字符串拼接到SDS字符串的末尾 | $O(N)$，N为指定C字符串的长度 |
| sdscatsds | 将指定SDS字符串拼接到另外一个SDS字符串的末尾 | $O(N)$，N为被拼接SDS字符串的长度 |
| sdscpy | 将指定字符串复制到SDS中，覆盖SDS原有字符串 | $O(N)$，N为指定C字符串的长度 |
| sdsgrowzero | 用空字符将SDS扩展到指定长度 | $O(N)$，N为扩展新增的字节数 |
| sdsrange | 保留SDS给定区间内的数据，不在区间内的数据会被覆盖或清除 | $O(N)$，N为被保留数据的字节数 |
| sdstrim | 从SDS中移除所有在C字符串中出现过的字符 | $O(N^2)$，N为指定C字符串的长度 |
| sdscmp | 对比两个SDS字符串是否相同 | $O(N)$，N为两个SDS中较短的那个SDS的长度 |