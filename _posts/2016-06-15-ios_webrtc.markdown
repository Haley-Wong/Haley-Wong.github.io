---
layout:     post
title:      "iOS下WebRTC音视频通话（一）"
date:       2016-06-15
author:     "Haley_Wong"
catalog:    true
tags:
    - WebRTC
---

在iOS下做IM功能时，难免都会涉及到音频通话和视频通话。QQ中的QQ电话和视频通话效果就非常好，但是如果你没有非常深厚的技术，也没有那么大的团队，很难做到QQ那么快速和稳定的通话效果。

但是利用WebRTC技术，即使一个人也能够实现效果不错的音视频通话。本篇介绍WebRTC的基础概念。

# WebRTC介绍
WebRTC，名称源自网页实时通信（Web Real-Time Communication）的缩写，是一个支持网页浏览器进行实时语音对话或视频对话的技术，是谷歌2010年以6820万美元收购Global IP Solutions公司而获得的一项技术。

WebRTC（Web Real-Time Communication）项目的最终目的主要是让Web开发者能够基于浏览器（Chrome\FireFox\...）轻易快捷开发出丰富的实时多媒体应用，而无需下载安装任何插件，Web开发者也无需关注多媒体的数字信号处理过程，只需编写简单的Javascript程序即可实现。但是经过多年的打磨，WebRTC现在已经可以在windows，linux，mac，android，iOS等多个平台中使用。

WebRTC除了可以用来做音频通话、视频通话，还可以用来做视频会议。

其他关于WebRTC的介绍可以参考：[百度百科-WebRTC](http://baike.baidu.com/link?url=cHob5Tsw3lgBwo-n9CQQPyYgz-qkHgIMFeCuixTqcTDQfyFy1wq63ucD4fZyzNfq01Iq5UIIiRhHvM_q38403a) 以及 [WebRTC官网](https://webrtc.org/)

# WebRTC 过程
WebRTC 利用RTCPeerConnection可以建立点对点高效、稳定的音频、视频流传输。但是在进行点对点的流传输之前，它依然还需要利用服务器来做一些准备工作。而准备工作需要用到的东西就比较多了，比如STUN服务器、TURN服务器、ICE（NAT和防火墙穿透）、信令传输，相互之间的信令交换完毕，就会发送实时音视频留给对方。

进行音视频通话的完整过程：

* 1、首先设置好STUN服务器、和TURN服务器，然后将STUN服务器和TURN服务器包装成RTCICEServer对象，保存进数组备用。
* 2、利用上一步的数组创建RTCPeerConnection连接。
* 3、为RTCPeerConnection添加RTCMediaStream，而RTCMediaStream内包含视频和音频轨迹，只是做一些配置，然后WebRTC内部按照你的配置做音频、视频的采集。如果你只为RTCMediaStream添加音轨，就是做音频通话；同时添加音轨和视频轨迹，则是做视频通话；只添加视频轨迹，则只能看到视频画面，没有声音。（这些都是在采集端设置）
* 4、为视频轨迹设置渲染的容器，便于开始音视频通话后，将实时视频画面渲染到视图上。（如果是音频通话则没有视频轨迹，就不需要渲染）
* 5、发起方创建Offer，创建完成后会返回一个本地SessisonDescription（简称sdp，其实就是一些媒体和网络相关的元数据信息），然后为RTCPeerConnection设置本地sdp（RTCPeerConnection需要设置远程sdp和本地sdp完成后才能进行点对点的流传输）。
* 6、将本地sdp信息设置完成后，将本地sdp发送给对方（这个过程就是将本地offer信令发送给对方）。
* 7、接收方收到offer信令之后，重复上面的1、2、3、4，然后将接收到的offer sdp设置为自己的远程sdp，然后再创建一个Answer。同样的创建完成后会返回一个SessisonDescription，将这个sdp设置为RTCPeerConnection的本地sdp，设置完成后再将answer发送给发起方。
* 8、发起方收到answer后，将answer sdp设置为RTCPeerConnection的远程sdp。
* 9、然后双方就开始互相发送多媒体流数据，整个音视频通话就完成了。


> **总结：**
>* STUN服务器、TURN服务器地址其实就是个url而已：`stun:stun.l.google.com:19302`，`turn:numb.viagenie.ca`，其中STUN服务器和TURN服务器可以在自家的服务上创建，STUN、TURN服务器可以有多个，做备用。
 * ICE，本端会生成所有网络接口对应不同协议的Candidate。 每一个Candidate实际上描述了和自己的通信方式。比如一个STUN类型的Candidate会包含本端在防火墙外的IP和端口类型。本端会通过信令协议（sip/xmpp/http）将自己的所有的Candidate发送给对端。对方接收到后，会尝试连接， 并找到一个最好的连接方式建立和本端的连接，之后的流媒体数据将通过此连接传输。
 * 关于Candidate，是对本端网络通信能力的一种描述。对于UDP/STUN协议，Candidate仅包含IP及端口信息，对于TURN，包含TURN server的IP，端口，以及用户名密码等。Candidate由本端代码生成，生成后通过信令发送给对端。对端会在本端所有的candidate中选择一个最好的建立与本端的连接。
 *  除了上面那些服务器外，还需要一些额外的服务器用来发现用户，比如XMPP服务，主要是为了维护用户的关系以及保持其在线、离线等状态。
 *  WebRTC框架内不提供信令服务，因此信令信息的发送和接收处理需要我们自己去处理。处理的方式也有很多种，比如利用XMPP的的发送和接收消息的机制，将信令信息发送给对方；也可以用Http网络将信令消息发送给对方；还可以利用WebSocket将信息发送给对方。



先大致了解WebRTC交互的过程，便于后面理解代码。
下一篇我会编写一个在同路由器 的局域网内进行视频通话的Demo。

关于WebRTC概念性的理解下面有几篇文章，文章内也有一些链接都是很好的资料：

[使用WebRTC搭建前端视频聊天室——入门篇](http://lingyu.wang/#/post/2014/3/15/webRTC-1)

[使用WebRTC搭建前端视频聊天室——信令篇](http://lingyu.wang/#/post/2014/3/18/webRTC-2)

[WebRTC的RTCDataChannel](http://lingyu.wang/#/post/2014/5/22/webrtc-data-channels)

虽然以上三篇主要是讲Web前端的WebRTC使用，但是过程和概念归纳的非常好，可以多读几遍。

[WebRTC and the Early API](https://hacks.mozilla.org/2013/07/webrtc-and-the-early-api/)

[WebRTC代理中的各种枚举状态](http://www.lxway.com/10564986.htm)

[P2P传输](http://blog.csdn.net/oupeng1991/article/details/28613597)，其中Candidate的作用以及P2P连接的过程介绍的对理解非常有帮助。

[WebRTC中文网](http://webrtc.org.cn/)

其实iOS 中WebRTC的处理过程与Web端的处理过程除了API命名不同，过程基本是一致的。
重要的是通过编写代码，然后对照代码的每一步去思考它这样做是为了干啥。

Have Fun!

