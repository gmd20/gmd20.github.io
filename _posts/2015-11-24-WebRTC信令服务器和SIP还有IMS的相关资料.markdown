webrtc还是需要一个信令服务来辅助双方建立连接的，之前对这个地方不是很清楚。
这篇文章讲的很清楚。
WebRTC in the real world: STUN, TURN and signaling
http://www.html5rocks.com/en/tutorials/webrtc/infrastructure/

最近在看SIP和IMS相关的东西，才回头过来看看WebRTC到底怎么个用法的。
其实WebRTC也可以可以使用SIP来建立连接的，但webrtc的实现没有给出具体的代码，而是把这部分留给第三方应用自己灵活实现。 webrtc应该给出了自己实现信令服务器的例子
Collider 
A websocket-based signaling server in Go.
https://github.com/webrtc/apprtc/tree/master/src/collider

webrtc使用的SDP也是来自用在SIP的，可以参考SIP 和SDP相关的rfc。

3gpp还有一个怎么把webrtc接入 VOLTE的 IMS 网络的规范

3GPP TS 24.371 (click spec number to see fileserver directory for this spec)
Web Real-Time Communications (WebRTC) access to the IP Multimedia (IM) Core Network (CN) subsystem (IMS); Stage 3; Protocol specification
http://www.3gpp.org/dynareport/24371.htm
