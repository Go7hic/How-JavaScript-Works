# JavaScript 是如何工作的: WebRTC 和点对点网络机制

![](https://cdn-images-1.medium.com/max/1600/0*iWlF5x7BVj1vh5Lu)
这是专门探索JavaScript及其构建组件的系列文章的第18篇。 在识别和描述核心元素的过程中，我们还分享了一些我们在构建SessionStack时使用的经验法则，这是一个JavaScript应用程序，需要强大且高性能，以帮助用户实时查看和重现其Web应用程序缺陷。

#### 概述
那么什么是WebRTC？ 首先，RTC代表实时通信，它已经提供了很多关于这项技术的信息。

WebRTC填补了网络平台的一个关键空白。 以前，桌面聊天应用等P2P技术可以做网络无法做到的事情。 WebRTC改变了这一点。

WebRTC基本上允许Web应用程序创建点对点通信，我们将在本文中讨论。 我们还将讨论以下主题，以便全面了解WebRTC的内部结构：
- 点对点通信
- 防火墙和NAT遍历
- 信令，会话和协议
- WebRTC API

#### 点对点通信
为了通过Web浏览器与另一个对等方进行通信，每个人的Web浏览器必须执行以下步骤：
- 同意开始通信
- 知道如何找到彼此
- 绕过安全和防火墙保护
- 实时传输所有多媒体通信

与基于浏览器的对等通信相关的最大挑战之一是知道如何与另一个Web浏览器定位和建立网络套接字连接，以便双向传输数据。我们将解决与建立这种联系相关的困难。

当您的Web应用程序需要某些数据或资源时，它会从某个服务器获取它，就是这样。但是，如果您想通过直接连接到其他人的浏览器来创建点对点视频聊天 - 您不知道该地址，因为其他浏览器不是已知的Web服务器。因此，为了建立p2p连接，需要更多。

#### 防火墙和NAT遍历
您的计算机通常没有分配静态公共IP地址。原因是您的计算机位于防火墙和网络访问转换设备（NAT）后面。

NAT设备是将私有IP地址从防火墙内部转换为面向公众的IP地址的设备。对于可用的公共IP地址的安全性和IPv4限制，需要NAT设备。这就是您的Web应用程序不应该假定当前设备具有公共静态IP地址的原因。

让我们看看NAT设备的工作原理。如果您在公司网络上并加入WiFi，则会为您的计算机分配一个仅存在于NAT后面的IP地址。假设IP是172.0.23.4。但是，对于外部世界，您的IP地址可能看起来像164.53.27.98。因此，外部世界将看到您的请求来自164.53.27.98，但NAT设备将确保您的计算机执行的请求响应将发送到内部172.0.23.4。这要归功于映射表。请注意，除IP地址外，网络通信还需要一个端口。

鉴于NAT设备的参与，您的浏览器需要找出机器的IP地址，该机器具有您要与之通信的浏览器。

这就是用于NAT（STUN）的会话遍历实用程序和使用NAT（TURN）服务器周围的中继的遍历的用武之地。为了使WebRTC技术起作用，首先向STUN服务器发出对面向公众的IP地址的请求。可以想象它就像你的计算机向远程服务器发出查询，它询问从哪个接收查询的IP地址。然后，远程服务器使用它看到的IP地址进行响应。

假设此过程有效并且您收到面向公众的IP地址和端口，则可以告诉其他对等方如何直接连接到您。这些对等体也能够使用STUN或TURN服务器执行相同的操作，并且可以告诉您在哪个地址与他们联系。

#### 信令，会话和协议
上述网络信息发现过程是信号较大主题的一部分，其基于WebRTC情况下的JavaScript会话建立协议（JSEP）标准。信令涉及网络发现和NAT遍历，会话创建和管理，通信安全性，媒体能力元数据和协调以及错误处理。

为了使连接起作用，对等体必须获取元数据的本地媒体条件（例如，分辨率和编解码器功能），并收集应用程序主机的可能网络地址。用于来回传递这些关键信息的信令机制并未内置于WebRTC API中。

WebRTC标准未指定信令，并且其API未实现，以便允许使用的技术和协议的灵活性。信令和处理它的服务器留给WebRTC应用程序开发人员处理。

假设您的基于WebRTC浏览器的应用程序能够使用所描述的STUN确定其面向公众的IP地址，则下一步是实际协商并与对等方建立网络会话连接。

初始会话协商和建立使用专用于多媒体通信的信令/通信协议进行。该协议还负责管理管理和终止会话的规则。

尝试与另一个对等体通信的任何对等体（即，WebRTC利用应用程序）生成一组交互式连接建立协议（ICE）候选者。 候选者代表要使用的IP地址，端口和传输协议的给定组合。 请注意，单台计算机可能具有多个网络接口（无线，有线等），因此可以为每个接口分配多个IP地址。

以下是MDN描绘此交换的图表。
![](https://cdn-images-1.medium.com/max/1600/0*SXRTlnVxy2-hE9ZX)


#### 连接建立

如上所述，每个对等体首先建立它面向公众的IP地址。然后动态地创建信令数据“信道”以检测对等体并支持对等协商和会话建立。

这些“频道”对于外部世界是未知的或可访问的，并且需要唯一的标识符来访问它们。

请注意，由于WebRTC的灵活性以及标准未指定信令过程的事实，考虑到所使用的技术，“信道”的概念和利用可能略有不同。实际上，一些协议不需要“通道”机制来进行通信。

为了本文的目的，我们假设在实现中使用了“通道”。

一旦两个或更多个对等体连接到相同的“信道”，则对等体能够通信并协商会话信息。 此过程有点类似于发布/订阅模式。 基本上，发起对等体使用信令协议发送“提议”，例如会话发起协议（SIP）和SDP。 发起者等待从连接到给定“信道”的任何接收器接收“应答”。

如果同意最佳ICE候选者的过程失败，有时由于防火墙和NAT技术的使用而发生，则回退是使用TURN服务器作为中继。该过程基本上采用充当中介的服务器，并且在对等体之间中继任何传输的数据。请注意，这不是真正的对等通信，其中对等方直接相互传输数据

当使用TURN回退进行通信时，每个对等方不再需要知道如何相互联系和传输数据。 相反，他们需要知道公共TURN服务器在通信会话期间发送和接收实时多媒体数据。


重要的是要明白这绝对是一个失败的安全和最后的手段。 TURN服务器需要非常强大，具有广泛的带宽和处理能力，并且可以处理潜在的大量数据。 因此，使用TURN服务器显然会产生额外的成本和复杂性。

#### WebRTC APIs
WebRTC中存在三种主要的API类别：

- Media Capture and Streams  - 允许您访问麦克风和网络摄像头等输入设备。 API允许您从其中任何一个获取媒体流。

- RTCPeerConnection - 使用这些API，您可以通过Internet将捕获的音频和视频流实时发送到另一个WebRTC端点。使用这些API，您可以在本地计算机和远程对等方之间创建连接。它提供了连接远程对等方的方法，维护和监视连接，并在不再需要时关闭连接。

- RTCDataChannel - 这些API允许您传输任意数据。每个数据通道都与RTCPeerConnection相关联。

我们将分别讨论这三个类别中的每一个。

#### Media Capture and Streams

Media Capture和Streams API（通常称为Media Stream API或Stream API）是支持音频或视频数据流的API，使用它们的方法，与数据类型相关的约束，成功与错误 异步使用数据时的回调，以及在此过程中触发的事件。


MediaDevices getUserMedia（）方法提示用户允许使用媒体输入，该媒体输入产生具有包含所请求的媒体类型的轨道的MediaStream。该流可以包括例如视频轨道（由诸如照相机，视频记录设备，屏幕共享服务等的硬件或虚拟视频源产生），音轨（类似地，由物理或虚拟产生）。音频源，如麦克风，A / D转换器等），以及可能的其他轨道类型。

它返回一个解析为MediaStream对象的Promise。如果用户拒绝权限，或者匹配的媒体不可用，则分别使用PermissionDeniedError或NotFoundError拒绝承诺。

```js
navigator.mediaDevices.getUserMedia(constraints)
.then(function(stream) {
 /* use the stream */
})
.catch(function(err) {
 /* handle the error */
});
```

请注意，您必须传递一个约束对象，该对象告诉API要返回哪种类型的流。 您可以配置各种内容，包括您要使用的相机（正面或背面），帧速率，分辨率等。

从版本25开始，基于Chromium的浏览器允许将来自getUserMedia（）的音频数据传递给音频或视频元素（但请注意，默认情况下媒体元素将被静音）。

getUserMedia can also be [used as an input node for the Web Audio API](http://updates.html5rocks.com/2012/09/Live-Web-Audio-Input-Enabled):

```js
function gotStream(stream) {
    window.AudioContext = window.AudioContext || window.webkitAudioContext;
    var audioContext = new AudioContext();
    // Create an AudioNode from the stream
    var mediaStreamSource = audioContext.createMediaStreamSource(stream);
    // Connect it to destination to hear yourself
    // or any other node for processing!
    mediaStreamSource.connect(audioContext.destination);
}

navigator.getUserMedia({audio:true}, gotStream);
```

##### 隐私约束
作为可能涉及重大隐私问题的API，getUserMedia（）由规范保存，以满足用户通知和权限管理的特定要求。 在打开任何媒体收集输入（如网络摄像头或麦克风）之前，getUserMedia（）必须始终获得用户权限。 浏览器可以提供每域一次的权限功能，但他们必须至少在第一次询问，并且用户必须在他们选择的情况下专门授予持续许可。

同样重要的是关于通知的规则。 浏览器需要显示一个指示器，显示正在使用的摄像头或麦克风，超出可能存在的任何硬件指示器。 它们还必须显示一个指示，即即使设备此刻没有主动记录，也已授予使用设备进行输入的权限。

#### RTCPeerConnection

RTCPeerConnection接口表示本地计算机与远程对等方之间的WebRTC连接。它提供了连接远程对等方的方法，维护和监视连接，并在不再需要时关闭连接。

下面是一个WebRTC架构图，显示了RTCPeerConnection的作用：

![](https://cdn-images-1.medium.com/max/1600/0*Nm9r_NLcAhJernmo)

从JavaScript的角度来看，从这个图中可以理解的主要内容是，RTCPeerConnection为Web开发人员提供了来自下方复杂内部的复杂性的抽象。 WebRTC使用的编解码器和协议可以进行大量工作，即使在不可靠的网络上也可以进行实时通信：

#### RTCDataChannel

- 丢包隐藏
- 回声消除
- 带宽适应性
- 动态抖动缓冲
- 自动增益控制
- 降噪和抑制
- 图像“清洁”

除音频和视频外，WebRTC还支持其他类型数据的实时通信。

API有许多用例，包括：

- 赌博
- 实时文字聊天
- 文件传输
- 分散的网络

该API具有多种功能，可充分利用RTCPeerConnection并实现强大而灵活的点对点通信：
- 利用RTCPeerConnection会话设置。
- 多个同步通道，具有优先级。
- 可靠且不可靠的交付语义。
- 内置安全性（DTLS）和拥塞控制。

语法类似于已知的WebSocket，带有send（）方法和消息事件：
```js
var peerConnection = new webkitRTCPeerConnection(servers,
    {optional: [{RtpDataChannels: true}]}
);

peerConnection.ondatachannel = function(event) {
    receiveChannel = event.channel;
    receiveChannel.onmessage = function(event){
        document.querySelector("#receiver").innerHTML = event.data;
    };
};

sendChannel = peerConnection.createDataChannel("sendDataChannel", {reliable: false});

document.querySelector("button#send").onclick = function (){
    var data = document.querySelector("textarea#send").value;
    sendChannel.send(data);
};
```

通信直接在浏览器之间进行，因此即使需要中继（TURN）服务器，RTCDataChannel也可以比WebSocket快得多。

#### WebRTC在现实世界中
在现实世界中，WebRTC需要服务器，无论多么简单，因此可能会发生以下情况：


在现实世界中，WebRTC需要服务器，无论多么简单，因此可能会发生以下情况：

- 用户发现彼此并交换名称等详细信息。
- WebRTC客户端应用程序（对等方）交换网络信息。
- Peers交换有关媒体的数据，如视频格式和分辨率。
- WebRTC客户端应用程序遍历NAT网关和防火墙。

换句话说，WebRTC需要四种类型的服务器端功能：
- 用户发现和沟通
- 信令
- NAT /防火墙遍历
- 在对等通信失败的情况下，中继服务器

ICE使用STUN协议及其扩展TURN来使RTCPeerConnection能够应对NAT遍历和其他网络变幻莫测。

如前所述，ICE是用于连接对等体的协议，例如两个视频聊天客户端。 最初，ICE尝试通过UDP直接连接对等端，以尽可能低的延迟。 在此过程中，STUN服务器只有一个任务：使NAT后面的对等体能够找到其公共地址和端口。 您可以查看这个可用的STUN服务器列表（Google也有几个）。

![](https://cdn-images-1.medium.com/max/1600/1*ONNxJHqmMTXB1Nuq3qTNXQ.png)

#### 寻找连接候选人
如果UDP失败，ICE会尝试TCP：首先是HTTP，然后是HTTPS。 如果直接连接失败 - 特别是由于企业NAT遍历和防火墙 - ICE使用中间（中继）TURN服务器。 换句话说，ICE将首先使用带有UDP的STUN直接连接对等体，如果失败，将返回到TURN中继服务器。 “查找候选者”这一表达指的是查找网络接口和端口的过程。
![](https://cdn-images-1.medium.com/max/1600/1*0REL14sYPR34hY7yua6-PA.png)

#### Security
实时通信应用程序或插件有多种方式可能会危及安全性。 例如：
- 未加密的媒体或数据可能在浏览器之间或浏览器与服务器之间被截获。
- 应用程序可能会在用户不知情的情况下录制和分发视频或音频。
- 恶意软件或病毒可能与明显无害的插件或应用程序一起安装。

WebRTC有几个功能可以避免这些问题：
- WebRTC实现使用安全协议，如DTLS和SRTP。
- 所有WebRTC组件都必须加密，包括信令机制。
- WebRTC不是插件：它的组件在浏览器沙箱中运行，而不是在单独的进程中运行，组件不需要单独安装，并且每当浏览器更新时都会更新。
- 必须明确授予摄像头和麦克风访问权限，并且当摄像头或麦克风运行时，用户界面会清楚地显示。

WebRTC是一种非常有趣和强大的技术，适用于在浏览器之间进行某种形式的实时流式传输的产品。

例如，我们在SessionStack允许我们的用户将我们的JavaScript库集成到他们的Web应用程序中。 会发生什么是我们的库开始收集数据，如用户事件，DOM更改，网络数据，异常，调试消息等，并将它们发送到我们的服务器。

与此同时，您的用户可以进入我们的网络应用程序并打开用户会话并实时观看。 使用收集的数据，SessionStack能够重新创建用户浏览器中当前正在发生的所有内容，将纯视觉信息与浏览器控制台的模拟以及引擎盖下的所有内容相结合。 可以将其视为远程桌面，但无需让最终用户下载任何软件。 在视觉信息之上，您实际上可以看到会话中的技术信息。

我们正在做所有这些服务器但是使用WebRTC我们实际上可以直接从浏览器到浏览器开始这样做，并减少延迟和我们最终所需的计算能力。