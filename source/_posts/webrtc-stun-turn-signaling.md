title: (译)WebRTC实战： STUN, TURN, Signaling
date: 2014-08-24 19:00:18
tags: [TURN, STUN, webrtc]
---
原文：[WebRTC in the real world: STUN, TURN and signaling](http://www.html5rocks.com/en/tutorials/webrtc/infrastructure/) By [Sam Dutton](http://www.html5rocks.com/profiles/#dutton)

WebRTC 实现了网页点对点交流。  
但是...  
WebRTC 仍然需要服务器来:

* 交换客户端元数据协调通讯，即信令(Signaling)。  
* 应对NATs(Network Address Translators) 和防火墙。  

本文将向你展示如何建立一个信令服务器，并使用STUN和TURN服务器来处理实际应用中出现的一些怪异的连接问题。也将解释WebRTC应用是如何处理多方通讯并与类似VoIP、PSTN的服务互动的。

如果你没有了解过WebRTC，我强烈建议你在看这篇文章之前先看看[这篇文章 Getting Started With WebRTC](http://www.html5rocks.com/en/tutorials/webrtc/basics/)
<!--more-->

#### 什么是信令？
信令即协调通讯的过程。WebRTC应用要发起一个对话，客户端就需要交换如下信息:

* 用于打开和关闭通讯的会话信息；
* 错误信息；
* 媒体元数据如编解码器及其设置，带宽和媒体类型；
* 秘钥数据，用于创建安全连接；
* 网络数据，如外部能访问的主机IP和端口。

这个信令过程需要客户端之间能来回传递消息，但是WebRTC APIs并没有提供这种机制的实现，你需要自己创建。下面将描述建立信令服务器的几种方式。不管怎么样，先来点上下文吧...

##### 为什么WebRTC不提供信令实现？
为了避免冗余，以及做到与现有技术的最大兼容，信令方法和协议都不由WebRTC标准来指定。这些都由JSEP([JavaScript Session Establishment Protocol](http://tools.ietf.org/html/draft-ietf-rtcweb-jsep-03#section-1.1))来概述.

>  WebRTC呼叫建立背后的想法已经是完全指定和控制媒体链接，但是尽量托管和应用间的信令连接。
> 由是不同的应用可能会喜欢用不同的协议，比如已存在的SIP、Jungle信令协议，或者也许为了一些新奇的用例而做的特殊应用而自定义的协议。
> 这一节文字要传达的关键信息点是多媒体会话的描述，这个描述指定了必要的传输和建立媒体链接所必要的媒体配置信息。

JSEP的架构也避免了让浏览器去保存状态，那就是，像一个信令状态机一样工作。这里也许会有一个问题，比如，当页面被刷新时，信令数据会丢失。不过，也可以把这些信令状态存在服务器。

![alt](http://7xkc1e.com1.z0.glb.clouddn.com/jsep.png)
##### <center>JSEP architecture</center>
JSEP需要offer和answer之间做出之前提到的媒体元数据的信息交换。offer和answer通过Session Description Protocol(SDP)格式来沟通，如下

```
v=0
o=- 7614219274584779017 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE audio video
a=msid-semantic: WMS
m=audio 1 RTP/SAVPF 111 103 104 0 8 107 106 105 13 126
c=IN IP4 0.0.0.0
a=rtcp:1 IN IP4 0.0.0.0
a=ice-ufrag:W2TGCZw2NZHuwlnf
a=ice-pwd:xdQEccP40E+P0L5qTyzDgfmW
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=mid:audio
a=rtcp-mux
a=crypto:1 AES_CM_128_HMAC_SHA1_80 inline:9c1AHz27dZ9xPI91YNfSlI67/EMkjHHIHORiClQe
a=rtpmap:111 opus/48000/2
…
```

想知道SDP的格式的所有明确含义，可以看看这个[IETF examples](http://datatracker.ietf.org/doc/draft-nandakumar-rtcweb-sdp/?include_text=1).
请记住WebRTC被设计使得offer或answer可以在被拧在一起之前通过编辑SDP文本来设置好本地或远程描述。比如[apprtc.appspot.com](https://apprtc.appspot.com/js/main.js)中的`preferAudioCodec()`方法就被用于设置默认的编解码器和比特率。SDP用Javascript来操作是有点痛苦，所以现在有个讨论是关于WebRTC的未来版本是否可以用JSON格式来替代，不过这里提到了一些坚持使用SDP的[好处](http://tools.ietf.org/html/draft-ietf-rtcweb-jsep-03#section-3.3)。

##### RTCPeerConnection + 信令: offer, answer and candidate
RTCPeerConnection接口被WebRTC应用用于创建各点之间的连接并交流视音频信息。
要开始这个过程RTCPeerConnection需要先做两个工作：

* 确定本地媒体情况，比如分辨率和编解码器的能力。这些元数据会用在offeranswer机制中。
* 获取可能的应用主机网络地址，就死所谓的candidate。

当本地信息被确认后，就会通过信令系统与远程终端进行交换。
联想下[alice is trying to call Eve](http://xkcd.com/177/)这幅漫画，发起/响应机制在其中完整的展现出来：

* Alice创建一个RTCPeerConnection对象。
* Alice通过RTCPeerConnection的createOffer()方法创建一个offer(SDP会话描述)。
* Alice通过这个offer调用setLocalDescription()。
* Alice将offer字符串化并通过信令服务器发给Eve。
* Eve通过调用setRemoteDescription()设置Alice的offer,来让自己的RTCPeerConnection知道Alice的设置。
* Eve调用createAnswer()和成功回调函数来传递Eve的本地会话描述--answer。
* Eve通过setLocalDescription()来将她的本地描述设置到她的answer中。
* 然后Eve将她字符串化后的answer通过信令服务器发回给Alice。
* Alice通过setRemoteDescription()将Eve的anwser设置为远程会话描述。

Alice和Eve还需要交换网络信息。'finding candidates'就是通过[ICE框架](http://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment)找到网络链接和端口的过程。

* Alice 通过onicecandidate事件处理器来创建RTCPeerConnection对象。
* 这个事件处理器将在candidates可用时被调用。
* 在这个处理器中，Alice通过信令服务器将candidate数据字符串化后发送给Eve。
* 当Eve得到Alice的candidate信息，她将调用addIceCandidate()方法将这个candidate加入自己的远程终端描述中。
JSEP支持[ICE Candidate Trickling](http://tools.ietf.org/html/draft-ietf-rtcweb-jsep-03#section-3.4.1), 这个可以使呼叫者在初始化offer之后增量的提供candidates给被呼叫者，被呼叫者在这个呼叫中直接开始设置链接而不用等待收到其他candidates。

##### 编写WebRTC信令服务
下面是一个简略的信令过程[W3C代码示例](http://www.w3.org/TR/webrtc/#simple-peer-to-peer-example)。这片代码假设已经存在一些信令机制，如SignalingChannel. 下面讨论信令的一些详细细节。

```javascript
var signalingChannel = new SignalingChannel();
var configuration = {
  'iceServers': [{
    'url': 'stun:stun.example.org'
  }]
};
var pc;

// call start() to initiate

function start() {
  pc = new RTCPeerConnection(configuration);

  // send any ice candidates to the other peer
  pc.onicecandidate = function (evt) {
    if (evt.candidate)
      signalingChannel.send(JSON.stringify({
        'candidate': evt.candidate
      }));
  };

  // let the 'negotiationneeded' event trigger offer generation
  pc.onnegotiationneeded = function () {
    pc.createOffer(localDescCreated, logError);
  }

  // once remote stream arrives, show it in the remote video element
  pc.onaddstream = function (evt) {
    remoteView.src = URL.createObjectURL(evt.stream);
  };

  // get a local stream, show it in a self-view and add it to be sent
  navigator.getUserMedia({
    'audio': true,
    'video': true
  }, function (stream) {
    selfView.src = URL.createObjectURL(stream);
    pc.addStream(stream);
  }, logError);
}

function localDescCreated(desc) {
  pc.setLocalDescription(desc, function () {
    signalingChannel.send(JSON.stringify({
      'sdp': pc.localDescription
    }));
  }, logError);
}

signalingChannel.onmessage = function (evt) {
  if (!pc)
    start();

  var message = JSON.parse(evt.data);
  if (message.sdp)
    pc.setRemoteDescription(new RTCSessionDescription(message.sdp), function () {
      // if we received an offer, we need to answer
      if (pc.remoteDescription.type == 'offer')
        pc.createAnswer(localDescCreated, logError);
    }, logError);
  else
    pc.addIceCandidate(new RTCIceCandidate(message.candidate));
};

function logError(error) {
  log(error.name + ': ' + error.message);
}
```

要知道这片代码中offer/answer和candidate交换过程是如何运作的，可以看看[simpl.info/pc](http://simpl.info/rtcpeerconnection/) 中视频聊天示例的控制台记录。如果你需要跟多细节，可以下载完整的WebRTC信令转储，并通过Chrome的 chrome://webrtc-internals 或Opera的 opera://webrtc-internals 页面来统计。

##### 终端的发现

要说清楚'我怎么才能找到某人来聊天'挺复杂的。
对于电话来说，我们有电话号码目录。对于在线视频聊天，我们需要身份认证以及在线状态管理系统，即用户初始化会话。WebRTC应用需要一种方式来让客户端来互相标识他们是像创建一个聊天室还是加入一个聊天。

WebRTC没有提供终端目录机制，所以我们不会进入这一项。这个过程可以简单的通过邮件或信息分享一个URL，比如 [talky.io](http://talky.io/)、[tawk.com](http://tawk.com/) 和 [browsermeeting.com](http://browsermeeting.com/)这些视频聊天应用中，你邀请别人加入是通过跟他们分享你的自有链接。开发者Chris Ball创建了一个有趣的实验[serverless-webrtc](http://blog.printf.net/articles/2013/05/17/webrtc-without-a-signaling-server/)让WebRTC的参与者通过IM,email或者信鸽来交换元数据。

#### 要如何建立一个信令服务器?</p>

重申一下，信令协议及机制并不由WebRTC标准定义。不管你选择什么，你都需要一个中介服务器来交换客户端之间的信令信息和应用数据。很可惜，网页应用并不能简单的直接冲着英特网说'把我和我的朋友连起来！'．

还好信令信息很小，并且大多数只在一个呼叫的开始才需要交换．在对[apprtc.appspot.com](http://apprtc.appspot.com/)和[samdutton-nodertc.jit.su](http://samdutton-nodertc.jit.su/)的测试中我们发现，一个视频聊天会话中，信令服务器总共处理了30-45条消息，所有消息的总大小才10kb左右。

并且对带宽的要求也较低，WebRTC信令服务器并不消耗太多cpu或内存，因为它们只需要做消息中转，并保存少量的会话状态数据(例如，有哪些客户已经连接了)。

> **Tip!**
> 信令机制可以用来交换会话元数据，也可以用来做应用数据通讯。它就是一个消息服务器。

##### 从服务器推送消息到客户端
信令的消息服务需要是双向的：客户端发到服务器且服务器发到客户端。双向通讯违反了HTTP协议的客户/服务，请求/响应模型。不过一些hack，比如为了将数据从服务器推送到网页的[长轮询](https://en.wikipedia.org/wiki/Comet_(programming))已经出现很多年了。

最近，[EventSource API](http://www.html5rocks.com/en/tutorials/eventsource/basics/)已被广泛的应用了，他使得服务器通过HTTP发送数据到浏览器成为可能。这里有个简单的[demo](http://simpl.info/es)。EventSource被设计成单向传递消息，但是它可以和XHR结合构建成交换信令消息的服务器:一个从呼叫者开始传递消息，用XHR请求传输，通过EventSource推送到被呼叫者那去。

[WebSocket](http://www.html5rocks.com/en/tutorials/websockets/basics/)是一个更自然的解决方案，被设计成全双工的客户端/服务器通讯(消息可以同时双向传输)。一个将信令服务器用纯WebSocket或服务器发送事件(EventSource)的型式构建的好处是后台接口可以由各种语言的通用框架公共托管包来实现，比如PHP,Python和Ruby。

大概四分之三的浏览器都[支持WebSocket](http://caniuse.com/#search=websocket)了，更重要的是，所有支持WebRTC的浏览器都支持WebSocket,不管是桌面端还是移动端。所有连接都需要使用[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security)，去保证不被截获到未加密的信息，并且减少[proxy traveral](http://en.wikipedia.org/wiki/WebSocket#Proxy_traversal)引起的问题。(需要更多WebSocket和proxy traversal相关的信息，可以看看Ilya Grigorik的*High Performance Browser Networking*一书的[WebRTC章节](http://hpbn.co/webrtc)。Peter Lubber的[WebSocket Cheat Sheet](http://refcardz.dzone.com/refcardz/html5-websocket)有更多关于WebSocket客户端和服务器端的信息)。

[apprtc.appspot.com](http://apprtc.appspot.com/)WebRTC视频聊天应用的信令是通过[Google App Engine Channel API](https://developers.google.com/appengine/docs/java/channel/)完成的，这个API用到了[Comet](http://en.wikipedia.org/wiki/Comet_(programming))技术(长轮询)去实现信令推送信息(这里有一个App Engine为支持WebSocket[存在很久的bug](https://code.google.com/p/googleappengine/issues/detail?id=2535),快去关注这个bug，给它投票别让它沉了!)。[这里](http://www.html5rocks.com/en/tutorials/webrtc/basics/)有一份这个应用的详细代码。

WebRTC客户端通过ajax轮询获取服务器信息处理信令也是可行的，但是这导致太多冗余的网络请求，尤其对于移动端客户来说更是一个问题。甚至在一个会话建立之后，终端仍需要轮询信令信息去查询是否会话有变化或者会话是否被对方终止了。这个[示例](http://webrtcbook.com/)使用了该方法，但做了一些轮询频率的优化。

##### 扩展信令服务器规模
虽然信令服务器对于每个客户来说消耗的带宽和CPU都较少，但是应用流行起来的话依然要处理不同地域的大量的数据，应对高并发。通信量较高的WebRTC应用需要能够应对高负载。

这里我们不会讨论细节，但仍有如下一些为高容量，高性能信息可以注意的点。

* [eXtensible Messaging and Presence Protocol (XMPP)](http://en.wikipedia.org/wiki/Xmpp), 最初被称为Jabber: 一个为即时信息开发的协议，可应用于信令服务，这种XMPP服务器的实现包括[ejabberd](http://en.wikipedia.org/wiki/Ejabberd)和[Openfire](http://en.wikipedia.org/wiki/Openfire).JavaScript客户端如[Strophe.js](http://strophe.im/strophejs/)使用[BOSH](http://en.wikipedia.org/wiki/BOSH)模拟双向流，但是因为[各种原因](http://stackoverflow.com/questions/7327153/xmpp-bosh-vs-comet)，BOSH的效率比不上WebSocket，而且也不易扩展。(话说[Jingle](http://en.wikipedia.org/wiki/Jingle_(protocol))是一个使XMPP支持音频和视频的扩展;WebRTC项目使用的网络传输组件[libjingle](https://developers.google.com/talk/libjingle/)库就是Jingle的C++实现)
* 开源库如[ZeroMQ](http://zeromq.org/)(TokBox用在他们的[Rumour](http://www.tokbox.com/blog/tokbox-builds-it%E2%80%99s-own-internal-messaging-infrastructure/)服务器上)和[OpenMQ](http://en.wikipedia.org/wiki/Open_Message_Queue)。[NullMQ](http://avalanche123.com/blog/2012/02/25/interacting-with-zeromq-from-the-browser/)在Web平台应用了ZeroMQ的概念，基于WebSocket使用[STOMP](http://stomp.github.io/)。
* 使用WebSocket的商业云消息平台(虽然他们不使用长轮询)如[Pusher](http://pusher.com/), [Kaazing](http://kaazing.com/)和[PubNub](http://pubnub.com/)。(PubNub还有[WebRTC的接口](https://github.com/pubnub/webrtc))

(开发者Phil Leggetter的[Real-Time Web Technologies Guide](http://www.leggetter.co.uk/real-time-web-technologies-guide)提供了一个关于消息服务和代码库的总结性清单。)

##### 在Node上用Socket.io建立信令服务器
以下的简单网页应用代码使用到了基于[Node](http://nodejs.org/)上的[Socket.io](http://socket.io/)而建立的信令服务器。Socket.io的设计使建立信息交换服务器变得简单，而且它尤其适用于WebRTC信令服务器，因为它内置了'房间'的概念。这个例子不是为产品级别的信令服务器设计的，但是它面向相对较小的用户群工作得很好。

Socket.io除了用WebSocket，还适配以下备用技术:Adobe Flash Socket, AJAX long polling, AJAX multipart streaming, Forever Iframe and JSONP polling. 它有多种后台实现，但是它的Node版本应该是最著名的，我们下面的例子就用的这个版本。

例子中没有WebRTC，这里只是展示网页应用信令该如何设计。查看控制台可以看到客户是如何加入一个房间且交换信息的。我们的[WebRTC codelab](https://bitbucket.org/webrtc/codelab)有如何将这个例子集成进完整的WebRTC视频聊天应用的步骤。你可以在[codelab repo第五步](https://bitbucket.org/webrtc/codelab/src/master/complete/step5)下载这些代码，也可以在[samdutton-nodertc.jit.su](http://samdutton-nodertc.jit.su/)在线试试效果。

index.html的代码如下:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>WebRTC client</title>
  </head>
  <body>
    <script src='/socket.io/socket.io.js'></script>
    <script src='js/main.js'></script>
  </body>
</html>
```

html中引用的的JavaScript文件main.js代码如下:

```javascript
var isInitiator;

room = prompt('Enter room name:');

var socket = io.connect();

if (room !== '') {
  console.log('Joining room ' + room);
  socket.emit('create or join', room);
}

socket.on('full', function (room){
  console.log('Room ' + room + ' is full');
});

socket.on('empty', function (room){
  isInitiator = true;
  console.log('Room ' + room + ' is empty');
});

socket.on('join', function (room){
  console.log('Making request to join room ' + room);
  console.log('You are the initiator!');
});

socket.on('log', function (array){
  console.log.apply(console, array);
});
```

完整的服务端应用代码:

```javascript
var static = require('node-static');
var http = require('http');
var file = new(static.Server)();
var app = http.createServer(function (req, res) {
  file.serve(req, res);
}).listen(2013);

var io = require('socket.io').listen(app);

io.sockets.on('connection', function (socket){

  // convenience function to log server messages to the client
  function log(){
    var array = ['>>> Message from server: '];
    for (var i = 0; i < arguments.length; i++) {
      array.push(arguments[i]);
    }
      socket.emit('log', array);
  }

  socket.on('message', function (message) {
    log('Got message:', message);
    // for a real app, would be room only (not broadcast)
    socket.broadcast.emit('message', message);
  });

  socket.on('create or join', function (room) {
    var numClients = io.sockets.clients(room).length;

    log('Room ' + room + ' has ' + numClients + ' client(s)');
    log('Request to create or join room ' + room);

    if (numClients === 0){
      socket.join(room);
      socket.emit('created', room);
    } else if (numClients === 1) {
      io.sockets.in(room).emit('join', room);
      socket.join(room);
      socket.emit('joined', room);
    } else { // max two clients
      socket.emit('full', room);
    }
    socket.emit('emit(): client ' + socket.id + ' joined room ' + room);
    socket.broadcast.emit('broadcast(): client ' + socket.id + ' joined room ' + room);

  });

});
```

(你并不需要知道这代码中的node-static是啥，它只是让服务器代码简单点。)
要在本地启动这个应用，你需要安装Node, socket.io和[node-static](https://github.com/cloudhead/node-static)。Node可以直接在[官网](http://nodejs.org/)下载(安装过程很简单)。要安装socket.io和node-static，在你的应用目录终端运行Node包管理器(NPM)就行了.

```
npm install socket.io
npm install node-static
```

要运行应用，只需要在你应用目录里终端运行如下命令:

```
node server.js
```

在你的浏览器中打开`localhost:2013`。在新的标签页或窗口将`localhost:2013`再打开一次。看看发生了什么，检查下Chrome或Opera的控制台，你可以用通过快捷键`Command-Option-J`或`Ctrl-Shift-J`来打开开发者工具DevTool。

不管你选择什么来实现你的信令，你的后台和客户端都至少至少需要提供一个和这个例子类似的服务。

##### 在信令服务器中使用RTCDataChannel
信令服务器需要初始化一个WebRTC会话。

然而，当两个终端间的连接建立后，RTCDataChannel理论上可以当作信令通道。这个可以减少信令的延迟并且减少信令服务器带宽和cpu的消耗，因为这样的信息是直接交流的。这里我们没有demo，不过大家仍需留意。

##### 信令陷阱?

* 在`setLocalDescription()`方法被调用前RTCPeerConnection都不会开始收集candidates,这是[JSEP IRTF draft](http://tools.ietf.org/html/draft-ietf-rtcweb-jsep-03#section-4.2.4)中要求的。
* 利用Trickle ICE(见前文):收到candidates信息立刻调用`addIceCandidate()`方法。

##### 现成的信令服务器
如果你不想你自己来做信令服务器，这里有提供一些WebRTC信令服务器，用的也是之前提到的Socket.io，并都集成了WebRTC客户端JavaScript代码库。

* [webRTC.io](https://github.com/webRTC/webRTC.io):WebRTC的第一个抽象代码库
* [easyRTC](https://github.com/priologic/easyrtc):全栈WebRTC包
* [Signalmaster](https://github.com/andyet/signalmaster):为[SimpleWebRTC](https://github.com/HenrikJoreteg/SimpleWebRTC)JavaScript库创建的信令服务器。

...如果你压根任何代码都不想写，这里也有一些完全商业化的WebRTC平台如[vLine](http://www.vline.com/), [OpenTok](http://tokbox.com/opentok),[Asterisk](https://wiki.asterisk.org/wiki/display/AST/Asterisk+WebRTC+Support).

需要指出来，Ericsson在WebRTC早期就已经用[PHP在Apache上搭了个信令服务器](https://labs.ericsson.com/blog/a-web-rtc-tutorial)。但是这个现在多少已经废弃了，不过如果你在考虑做类似的事的话，这代码还是值得一看的。

##### 信令安全
> Security is the art of makeing nothing happen.
> --[Salman Rushdie](http://t.co/S9cUM7bVfd)

加密在WebRTC组件中是强制的。

然而，信令机制并不由WebRTC标准所定义，所以让信令更安全就是你自己的事了。如果攻击者试图劫持信令, HTTPS和WSS(i.e TLS),可以保证他们不会拦截到未加密的信息。你也要注意不要在其他用同一个服务器的用户能访问到的地方广播信令信息。

> 要保护WebRTC应用，在信令中使用[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security)是绝对必要的。


#### 信令之外:使用ICE应付NAT和防火墙
对于信令元数据，WebRTC应用使用了中介服务器，但是对于会话建立后的真正媒体数据流，RTCPeerConnection试图让客户终端直连:点对点连接。
简单的情况下，每个WebRTC终端都有一个唯一的地址，可以使得各终端都能互相直接通讯。
{}![p2p](http://7xkc1e.com1.z0.glb.clouddn.com/p2p.png)
##### <center>没有NAT和防火墙的情况</center>

但是大多数设备都处于一层或多层[NAT(网络地址转换器)](http://www.howstuffworks.com/nat.htm)之后，还有杀毒软件的阻挡了一些端口或协议，又或者使用了代理或者防火墙。防火墙和NAT事实上可能在同一设备上，比如家庭无线路由器。
![nat](http://7xkc1e.com1.z0.glb.clouddn.com/nat.png)
##### <center>实际情况</center>

WebRTC应用可以使用[ICE](https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment)框架来克服实际应用中复杂的网络问题。要使用ICE的话，你的应用必须如下所述的在RTCPeerConnection中传递ICE服务器的URL。

ICE试图找到连接端点的最佳路径。它并行的查找所有可能性，然后选择最有效率的一项。ICE首先用从设备操作系统和网卡上获取的主机地址来尝试连接，如果失败了(比如设备处于NAT之后)，ICE会使用从STUN 服务器获取到的外部地址，如果仍然失败，则交由TURN中继服务器来连接。

换句话说:

* STUN服务器用于获取设备的外部网络地址
* TURN服务器是在点对点失败后用于通信中继。

每一个TURN服务器都支持STUN，因为TURN就是在STUN服务器中内建了一个中继功能。ICE也可以应付NAT复杂的设定:实际上，NAR'打洞'会有不止一个公共 IP : port 地址。STUN或TURN服务器的URL由WebRTC中RTCPeerConnection的第一个参数iceServers配置对象可选指定。[apprtc.appspot.com](http://apprtc.appspot.com/)中的值是这样的:

```
{
  'iceServers': [
    {
      'url': 'stun:stun.l.google.com:19302'
    },
    {
      'url': 'turn:192.158.29.39:3478?transport=udp',
      'credential': 'JZEOEt2V3Qb0y27GRntt2u2PAYA=',
      'username': '28224511:1379330808'
    },
    {
      'url': 'turn:192.158.29.39:3478?transport=tcp',
      'credential': 'JZEOEt2V3Qb0y27GRntt2u2PAYA=',
      'username': '28224511:1379330808'
    }
  ]
}
```
一旦RTCPeerConnection中有了这些信息，ICE的神奇就自动展现了: RTCPeerConnection使用ICE框架找到各端点间最合适的路径，必要时选用STUN和TURN服务器。

##### STUN
NAT在本地私有网络中为设备提供了一个IP地址，但是这个地址并不能被外部识别。没有一个公共地址的话，WebRTC终端是没有办法通信的。要解决这个问题，WebRTC使用了[STUN](https://en.wikipedia.org/wiki/STUN)。

STUN服务器处于公网中并有个简单任务:检查请求(来自运行于NAT之后的应用)的IP:port 地址，并且将这个地址响应回去。换句话说，NAT后的应用使用STUN服务器来找到他的IP:port 公网地址。这个过程使得WebRTC终端可以找到自己公共访问方法，并通过信令机制将之发送给其他终端，就可以创建一个直连链接。(在实践中，不同的NAT工作方式不同，并有可能有多层NAT，但是原理是一样的。)

STUN服务器并没有做太多东西，也不用记住很多东西，所以一个相对低规格的的STUN服务器可以处理大量的请求。

根据[webrtcstats.com](http://webrtcstats.com/)的调查，大部分(86%)WebRTC请求都可以通过STUN成功的创建连接，虽然对处于防火墙或者配置复杂的NAT之后的终端要低一些。
![stun](http://7xkc1e.com1.z0.glb.clouddn.com/stun.png)
<center>用STUN服务器获取公共IP:port 地址</center>

##### TURN
RTCPeerConnection尝试用UDP协议建立终端间的直连。如果失败了，就尝试TCP协议，还是失败的话,TURN 服务器就会用于做终端间的数据中继。

**重申一下：TURN用于中继视频音频数据流，而不是信令数据!**

TURN服务器有公共地址，所以他可以被终端联系到，哪怕终端处于防火墙或者代理之后。TURN服务器有一个概念上简单的工作--做数据流中继--但是，不像STUN服务器，它天生需要消耗大量带宽，也就是说，TURN服务器需要很强大。
![turn](http://7xkc1e.com1.z0.glb.clouddn.com/turn.png)
##### <center>STUN,TURN和信令</center>

这幅图展现了TURN的运作，纯STUN不能成功的话，各终端将使用TURN服务器。

##### 部署STUN和TURN服务器
Google运行了一个公用的STUN服务器用作测试，`stun.l.google.com:19302`,[apprtc.appspot.com](http://apprtc.appspot.com/)用到了它。我们建议使用rfc5766-turn-server当作产品用途的STUN/TURN服务，STUN/TURN服务器的源代码可以在[code.google.com/p/rfc5766-turn-server](https://code.google.com/p/rfc5766-turn-server/) 找到,这里也提供了一些服务器安装的相关信息链接。[Amazon Web Services(AWS)也提供了WebRTC的虚拟机镜像](https://groups.google.com/forum/#!msg/discuss-webrtc/X-OeIUC0efs/XW5Wf7Tt1vMJ)。

另一个备选TURN服务器是restund，有[源代码](http://www.creytiv.com/restund.html)，也可以装到在AWS上。下面是介绍如何将restund装到Google Compute Engine上。

1.  防火墙开放tcp=443,udp/tcp=3478
2.  创建4个实例(?)，各自的公用IP，使用标准Ubuntu 12.06镜像
3.  配置本地防火墙(允许所有访问源)
4.  安装工具:
    `sudo apt-get install make`
    `sudo apt-get install gcc`
5.  从[creytiv.com/re.html](http://creytiv.com/re.html)安装libre
6.  从[creytiv.com/restund.html](http://creytiv.com/restund.html)获取restund并解包
7.  wget [hancke.name/restund-auth.patch](http://hancke.name/restund-auth.patch) 并应用`patch -p1 < restund-auth.patch`
8.  对libre和restund运行 make, `sudo make install`
9.  按你自己的需求配置*restund.conf*(替换IP地址，确保正确的共享密钥)并复制到`/etc`目录
10. 复制*restund/etc/restund*到*/etc/init.d/*
11. 配置restund:
    设置*LD_LIBRARY_PATH*
    复制*restund.conf*到*/etc/restund.conf*
    设置*restund.conf*使用之前配的IP地址
12. 运行restund
13. 在远程机器运行stund client命令做测试: `./client IP:port`


#### 点对点之外:多方WebRTC通讯
你也许会对Justin Uberti为[REST API for access to TURN Services](http://tools.ietf.org/html/draft-uberti-rtcweb-turn-rest-00)提出的IETF标准感兴趣。

很容易想到一个超越简单的点对点媒体流用例:比如,同事间的视频会议，或者一个有数百(万)用户的公共演讲。

WebRTC应用可以使用多RTCPeerConnection，让各终端之间以网状配置连接。这就是如[talky.io](http://talky.io/)这类应用所使用的方法，并且在少量终端的情况下运行的非常良好。不过，CPU和带宽都消耗非常多，尤其是在移动终端上。
![mesh](http://7xkc1e.com1.z0.glb.clouddn.com/mesh.png)

##### <center>网状拓扑结构:所有终端都互相连接</center>

此外，WebRTC应用可以按星状拓扑结构来选择一个终端分发数据流。在服务器运行一个WebRTC终端来作为重新分配机制也是可行的(webrtc.org提供了一个[简单例子](https://code.google.com/p/webrtc/source/browse/#svn%2Ftrunk%2Ftalk))。

从Chrome 31和Opera 18开始，RTCPeerConnection的MediaStream可以当作另一个RTCPeerConnection的输入:这里有个简单演示[simpl.info/rtcpeerconnection/multi](http://simpl.info/rtcpeerconnection/multi), 这使得应用结构更灵活，因为它使网络应用通过选择其他终端的连接来处理路由成为可能。

##### Multipoint Control Unit
对于大量终端的更好选择是使用[Multipoint Control Unit(MCU)](https://en.wikipedia.org/wiki/Multipoint_control_unit).这是一个服务器，像大量参与者之间的桥梁一样用于分发媒体信息。MCU可以在一个视频会议中使用多种分辨率，编解码器和帧率，处理转换编码，选择数据流径，调制或录制视频音频。对于多方通话，有一堆问题需要注意: 特别是，如何显示多视频输入和混调多源音频。云平台如[vLine](http://www.vline.com/)有尝试优化流量路径。

可以考虑买一个MCU的硬件，或者自己做一个。
<center>![mcu](http://7xkc1e.com1.z0.glb.clouddn.com/mcu.jpg)</center>

##### <center>[Cisco MCU](http://cisco.com/en/US/products/ps12283)的背面</center>

有不少能用的开源MCU软件供选择。比如,[Licode](http://lynckia.com/)(之前叫Lynckia)就为WebRTC做了一个开源MCU，OpenTok也有一个开源产品[Mantis](http://www.tokbox.com/blog/mantis-next-generation-cloud-technology-for-webrtc/)。


#### 浏览器之外: VoIP, 电话和短信
WebRTC的标准性质使得浏览器中运行的WebRTC应用可以和运行其他通信平台的设备或者平台建立通讯，比如电话或者视频会议系统。

[SIP](http://en.wikipedia.org/wiki/Session_Initiation_Protocol)是VoIP和视频会议系统的信令协议。要使WebRTC网页应用能和其他如视频会议系统的SIP客户端通讯，WebRTC需要一个代理服务器做中介信令。信令需要流过网关，但是一旦通信已经建立起来，SRTP(视频和音频)就可以点对点传输。

[PSTN](http://en.wikipedia.org/wiki/Public_switched_telephone_network)(Public Switched Telephone Network)，公用电话交换网络，是所有普通模拟电话的闭路交换网络。要用WebRTC网页应用打电话，流量必须经过PSTN网关。此外，WebRTC网页应用需要用中介XMPP服务器来与[Jingle](http://en.wikipedia.org/wiki/Jingle_(protocol))终端如即时通信客户端通讯。
Jingle由Google开发来作为XMPP扩展用于支持视频和音频信息:现在WebRTC的实现基于[libjingle](https://developers.google.com/talk/libjingle/) C++库，这个Jingle的实现刚开始是为Google Talk开发的。

一些应用，代码库和平台利用WebRTC的能力来于外界通讯，如：

* [sipML5](https://code.google.com/p/sipml5/)：开源JavaScript SIP客户端
* [jsSIP](http://www.jssip.net/)：JavaScript SIP代码库
* [Phono](http://phono.com/)：开源JavaScript电话接口，作为插件开发
* [Zingaya](http://zingaya.com/product/)：嵌入式电话组件
* [Twilio](http://www.twilio.com/)：音频消息应用
* [Uberconference](http://www.uberconference.com/)：会议系统

sipML5的开发们也开发了[webrtc2sip](https://code.google.com/p/webrtc2sip/)网关。Tethr and Tropo在笔记本上演示过一个[救灾通讯框架](http://tethr.tumblr.com/post/25513708436/tethr-and-tropo-in-the-google-i-o-sandbox), 使用[OpenBTS cell](http://en.wikipedia.org/wiki/OpenBTS)让电脑能通过WebRTC与一个特别的电话通讯。无需电信就能打电话啦！


#### 更多
WebRTC [codelab](https://bitbucket.org/webrtc/codelab)：一步一步介绍如何建立一个视频文字聊天应用，使用了在Node中运行的Socket.io信令服务器。

2013 Google I/O 大会上由WebRTC技术组长Justin Uberti做的[WebRTC报告](http://www.youtube.com/watch?v=p2HzZkd2A40)。

Chris Wilson在SFHTML5上的报告:[Introduction to WebRTC Apps](http://www.youtube.com/watch?v=3Ifbqaw5l_I)

[WebRTC Book](http://webrtcbook.com/)提供了很多数据和信令路径的详细信息，包括了许多详细的网络拓扑图。

[WebRTC and Signaling: What Two Years Has Taught Us](http://www.tokbox.com/blog/webrtc-and-signaling-what-two-years-has-taught-us/):TokBox的一篇博文告诉我们为什么要把信令从WebRTC细则中单独拎出来。

[Ben Strong](http://www.linkedin.com/in/strongben)的报告[A Practical Guide to Building WebRTC Apps](https://thenewcircle.com/s/post/1548/a_practical_guide_to_building_webrtc_apps_ben_strong_video)提供了很多WebRTC拓扑和基础。

[Ilya Grigorik](http://www.igvita.com/)的*High Performance Browser Networking*一书中的[WebRTC章节](http://chimera.labs.oreilly.com/books/1230000000545/ch18.html)深入描述了WebRTC结构，用例和性能。
