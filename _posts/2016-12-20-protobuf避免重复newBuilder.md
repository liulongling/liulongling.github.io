---
layout:     post
title:      "开发笔记(一)：避免重复代码"
subtitle:   ""
date:       2016-12-20 15:00:00
author:     "donkey"
header-img: "img/in-post/default-bg.jpg"
tags:
    - 开发笔记
---

最近大家都在忙着功能优化和BUG，而我却比较闲，开发的模块没有什么BUG也没什么需要优化的，便抽些时间来改下正在开发项目中的一些坏代码。

  坏代码一：
 
```
    //坏代码
    SC_ResponseMsgProto.Builder responseMsgBuilder = SC_ResponseMsgProto.newBuilder();//重复一
    DataMsgProto.Builder dataMsgBuilder = DataMsgProto.newBuilder();//重复二
    dataMsgBuilder.setB(b.build());
    responseMsgBuilder.setDataMsg(dataMsgBuilder.build());//重复三
    responseMsgBuilder.setA(a.build());
    sendMsgToClient(channel, responseMsgBuilder.build());//重复四
```
  
```
message SC_ResponseMsgProto {

    optional DataMsgProto                  dataMsg                  = 1;
    optional A                              a                       = 2;


    message DataMsgProto {
	   optional B              b              = 1;
    }
    
    message A {
	   optional int32              num             = 1;
	   optional C                  c               = 2;
    }

     message B {
          optional int32              num             = 1;
     }

     message C {
         optional int32              num             = 1;
    }
}
```
  
  项目客户端和服务器采用的是Google的protobuf协议，数据结构设计采用的是森林结构。如上所示，SC_ResponseMsgProto是最顶层对象，在最顶层对象下有多个不同类别的子对象，如A、B、C，子对象下还拥有一些不同类别的子对象C，最终成为一个森林结构，森林里每个根对象都是一颗树。对象间有时有一些引用关系，比如，A对象使用了C引用。这样的设计在开发中是比较常见的，但是我们在项目中使用时比较流水账，大量的顶层对象代码在不同地方重复出现在项目中。对于每个程序员员来说，避免重复代码是大家都想做的。于是我大约花了不到30行代码来避免以上标注的四个重复地方，代码如下：

  
```
    /**
     * 下发给客户端的消息
     */
    private final LinkedList<GeneratedMessageLite> responseMsgs = new LinkedList<>();
    
    //添加消息
    public void putMsg(GeneratedMessageLite msg) {
        synchronized (responseMsgs) {
            responseMsgs.add(msg);
        }
    }
    
    //发送消息
    public void sendMsg(Channel channel) {
        //和客户端通讯的数据
        SC_ResponseMsgProto.Builder responseMsgBuilder = SC_ResponseMsgProto.newBuilder();
        DataMsgProto.Builder dataMsgBuilder = DataMsgProto.newBuilder();
        try {
            for (GeneratedMessageLite messageLite : responseMsgs) 
            {
               if (messageLite instanceof A) {
                    responseMsgBuilder.setA((A) messageLite);
                } else if(messageLite instanceof B) {
                    dataMsgBuilder.setB((B) messageLite);
                }
                responseMsgBuilder.setDataMsg(dataMsgBuilder.build());
            }
            sendMsgToClient(channel, responseMsgBuilder.build());
        }catch (Exception e) {
            e.printStackTrace();
            logger.error(e.getMessage(), e);
        } finally {
            responseMsgs.clear();
        }
        
    }

```

```
    //优化后代码
    role.putMsg(a.build());
    role.putMsg(b.build());
    role.sendMsg(channel);
```

虽然避免了重复代码，但是细心的童鞋可能发现在sendMsg()里意外的出现了不是最顶层对象的DataMsgProto.如果当DataMsgProto里出现相同类型的b、b1对象时，那就无法从类型上去区分它们了。我想到的解决方案是使用自定义唯一key来区分或者在定义协议时不允许出现相同类型的(这里并不推荐，毕竟有时候多个类型能解决不少问题)。

  有同学可能会想能否使用变量名称b作为唯一区分标识,这个是不可以的,因为java不能获取变量名字，[详细参考R大在知乎上的回答](https://www.zhihu.com/question/29643012)。


```
message DataMsgProto {
	optional B              b              = 1;
	optional B              b1             = 2;
}
```
