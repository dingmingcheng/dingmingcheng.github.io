---
layout: post
title: "redis-skiplist源码学习"
date: 2018-02-06
description: "redis源码学习"
tag: redis
---

## 前言

注：以下代码基于redis-4.0.7

redis中有5种数据结构，之前面试时好几次遇到“排行榜”的问题，虽说都答了出来，不过对其底层的实现还是不清楚。抽空去了解学习了一下。也看到了一些比较好的文章，给了我比较大的帮助，[为啥 redis 使用跳表(skiplist)而不是使用 red-black？](https://www.zhihu.com/question/20202931),[跳跃表](http://redisbook.readthedocs.io/en/latest/internal-datastruct/skiplist.html)。

## 具体实现

### 数据结构

先来看看数据结构，这对理解redis的skiplist会有很大帮助

``` c
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned int span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

zkiplist很好理解，一个头指针，一个尾指针，length代表zskiplistNode的个数，level代表最大的层数。sds ele;代表动态字符串，也就是插入时的名称，score也就是其分数，backward是一个后退指针，方便从尾往前遍历。还有一个就是一个level数组，level数组的大小由一个随机函数生成，下面会有详细的说明。其中有一个span，代表了跨越节点数，比如有5个节点，level为3，那么很有可能level[]的size为3的Node只有1个，那么他和下一个level[3]之间就会有跨越节点的说法。

### 主要函数

``` c
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}
```

``` c
//初始化 zskiplist
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    //zskiplist，初始化长度为0，level为1
    zsl->level = 1;
    zsl->length = 0;
    //初始化head节点，level为2的32次，均指向null，跨越节点数为0
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

``` c
//插入zskipNode
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    //rank辅助函数，统计排名
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    //找到小于score中值最大的节点，当然list有许多的level，所以会有zlist->level个zskiplistLevel节点需要更新（）
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    
    //level生成算法,使用了一个随机算法,也比较好理解，ZSKIPLIST_P默认为0.25，也就是25%的概率level+1，
    //也就是level=1的概率为0.75,level=2的概率为0.75*0.25，3为0.25*0.25*0.75...其实就像是一棵四叉树，第k层和第k+1层的节点数量是4倍的关系

 	// int zslRandomLevel(void) {
	//     int level = 1;
	//     while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
	//         level += 1;
	//     return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
	// }
    level = zslRandomLevel();
    // 随机生成level大于zskiplist的当前level
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }


    x = zslCreateNode(level,score,ele);
    //后续操作比较简单，就是根据之前的数据进行节点更新，指针操作
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    //之后增加了一个节点，所以跨越节点数+1
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```

## 结尾

目前只看了这些比较重要的函数，别的函数我看代码不多，而且一些命令没有实际操作过，希望日后可以补充起来，只要理解了数据结构，别的其实和insert都是大同小异的。