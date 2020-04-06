---
layout: post
title: "rocketmq源码阅读(8)——store文件数据结构"
date: 2019-04-27
description: "rocketmq源码阅读"
tag: rocketmq
---

##前言

上文简单分析了一下rocketmq中的存储部分，本文中，我们来看看主要的存储文件中的数据结构。包括CommitLog，ConsumerQueue以及IndexFile，可以试着去读取这些文件

## 文件结构

### CommitLog

``` java
private static void readCommitlog(int/*读取的消息数量*/ t) throws Exception{
  String file = "/Users/dingmc/store/commitlog/00000000000000000000";
  RandomAccessFile accessFile = new RandomAccessFile(file, "r");

  for (int j = 0; j < t; j++) {
    System.out.println("----------------第" + j + "条消息----------------");
    System.out.println("msgLength:" + accessFile.readInt());
    System.out.println("MESSAGE_MAGIC_CODE:" + accessFile.readInt());
    System.out.println("bodyCRC:" + accessFile.readInt());
    System.out.println("queueId:" + accessFile.readInt());
    System.out.println("flag:" + accessFile.readInt());

    System.out.println("QUEUEOFFSET:" + accessFile.readLong());
    System.out.println("PHYSICALOFFSET:" + accessFile.readLong());

    System.out.println("SYSFLAG:" + accessFile.readInt());
    System.out.println("BORNTIMESTAMP:" + accessFile.readLong());
    byte[] b = new byte[4];
    accessFile.read(b);
    System.out.print("born host:");
    for (int i = 0; i < 4; i++) {
      System.out.print(b[i] & 0xFF);
      if (i < 3) {
        System.out.print(".");
      }
    }
    System.out.println();
    System.out.println("born port:" + accessFile.readInt());

    System.out.println("STORETIMESTAMP:" + accessFile.readLong());

    accessFile.read(b);
    System.out.print("store host:");
    for (int i = 0; i < 4; i++) {
      System.out.print(b[i] & 0xFF);
      if (i < 3) {
        System.out.print(".");
      }
    }
    System.out.println();
    System.out.println("store port:" + accessFile.readInt());

    System.out.println("RECONSUMETIMES:" + accessFile.readInt());
    System.out.println("Prepared Transaction Offset：" + accessFile.readLong());
    //bodyLength
    int bodyLength;
    System.out.println("bodyLength:" + (bodyLength = accessFile.readInt()));

    //大于4k会进行压缩
    byte[] b1 = new byte[bodyLength];
    accessFile.read(b1);
    String bodyStr = new String(b1);
    System.out.println("body:" + bodyStr);

    //topiclength
    byte[] b2 = new byte[1];
    accessFile.read(b2);
    int topicLength = byteArrayToInt(b2);
    System.out.println("topic length:" + topicLength);

    //topic
    byte[] b3 = new byte[topicLength];
    accessFile.read(b3);
    System.out.println("topic name:" + new String(b3));

    //properties
    int propertiesLen = accessFile.readShort();
    System.out.println("PROPERTIESLen:" + propertiesLen);
    byte[] b4 = new byte[propertiesLen];
    accessFile.read(b4);
    System.out.println("PROPERTIES:" + new String(b4));
  }
}

```

没什么好说的，就按rocketmq源码中怎么写进去的，这边就怎么读出来，多看看多试试就行，再附上一段控制台结果：

![](/img/in-post/rocketmq-8-store/6.png)

### ConsumerQueue

consumerQueue就更明显了

``` java
public static void readConsumerQueue(int sum) throws Exception{
  String file = "/Users/dingmc/store/consumequeue/test_topic/1/00000000000000000000";
  RandomAccessFile accessFile = new RandomAccessFile(file, "r");
  for (int i = 0; i < sum; i++) {
    System.out.println("------------------");
    System.out.println("offset:" + accessFile.readLong());
    System.out.println("size:" + accessFile.readInt());
    System.out.println("tagcode:" + accessFile.readLong());
  }
}
```

附上控制台输出结果

``` java
------------------
offset:263
size:263
tagcode:0
------------------
offset:526
size:319
tagcode:-1143080936、
```

### IndexFile

最后是indexFile的读取，indexFile数据结构稍微复杂些。我这边发送了两条key一样的消息，这样就会造成hash冲突，我试着从index文件中找这个key：

``` java
public static void readIndex() throws Exception{
  String key = "test_topic#keys";
  int absSlotPos = 40 + 4 * (Math.abs(key.hashCode()) % 5000000);
  String file = "/Users/dingmc/store/index/20190304141342419";
  RandomAccessFile accessFile = new RandomAccessFile(file, "r");
  //MappedByteBuffer方式
  //        FileChannel channel = accessFile.getChannel();
  //        MappedByteBuffer mappedByteBuffer = channel.map(MapMode.READ_ONLY, 0, 20000000);
  //        System.out.println(mappedByteBuffer.getInt(absSlotPos));

  System.out.println("keyHash:" + Math.abs(key.hashCode()));
  accessFile.skipBytes(absSlotPos);
  int index = accessFile.readInt();
  while (index != 0) {
    System.out.println("-----------------------");
    System.out.println("slot index:" + index);
    int absIndexPos = 40 + 5000000 * 4 + index * 20;

    RandomAccessFile accessFile2 = new RandomAccessFile(file, "r");
    accessFile2.skipBytes(absIndexPos);

    System.out.println("keyhash:" + accessFile2.readInt());
    System.out.println("commitLogOffset:" + accessFile2.readLong());
    System.out.println("timeDiff(storeTime - indexFileBegintime):" + accessFile2.readInt());
    index = accessFile2.readInt();
    System.out.println("lastSlotIndex:" + index);
  }
}

```

果然读出来两条记录：

``` java
keyHash:2128978283
-----------------------
slot index:4
keyhash:2128978283
commitLogOffset:263
timeDiff(storeTime - indexFileBegintime):143
lastSlotIndex:2
-----------------------
slot index:2
keyhash:2128978283
commitLogOffset:0
timeDiff(storeTime - indexFileBegintime):0
lastSlotIndex:0
```



## 总结

本文主要试着去读了读几个存储文件，果然正如所预料的那样