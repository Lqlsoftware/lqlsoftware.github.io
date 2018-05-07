``` go
// TCP包处理
func PacketHandler(packet gopacket.Packet) {
	if packet == nil {
		return
	}

	// 解析TCP报文
	tcpLayer := packet.Layer(layers.LayerTypeTCP).(*layers.TCP)

	// 处理请求 端口通道存在
	useMap.RLock()
	ch,ok := chMap[tcpLayer.SrcPort]
	useMap.RUnlock()
	if ok {
		// 发送至相应端口通道
		ch<- packet
	} else if tcpLayer.SYN {
		// 建立新线程进行TCP连接
		go handleThread(packet, tcpLayer.SrcPort)
	}
}
// TCP Connection
type Connection struct {
	// 本机信息
	srcIP		[]byte
	srcPort		layers.TCPPort
	srcMac		[]byte
	srcSeq		uint32
	// 客户端信息
	dstIP		[]byte
	dstPort		layers.TCPPort
	dstMac		[]byte
	dstSeq		uint32
	dstAck		uint32
	dstMSS		uint16
	dstWin		uint16
	// 数据包channel
	Channel		*chan gopacket.Packet
	// 连接状态
	State		State
}
// 根据连接状态处理包
switch tcpConn.State {
// 未连接
case UNCONNECT:
	if tcp.SYN {
		tcpConn = NewConnection(channel, request)
	} else if tcp.FIN {
		tcpConn.sendFin()
		tcpConn.State = SENDFIN
	}
// 等待握手ACK
case WAITSYNACK:
	tcpConn.State = CONNECTED
// 已连接 / keep-alive
case CONNECTED:
	// 返回ACK
	tcpConn.sendAck()
	if request.ApplicationLayer() == nil {
		// keep-alive 心跳包
		tcpConn.sendAck()
		continue
	} else if len(tcp.Payload) == int(tcpConn.dstMSS) {
		// 请求长度超过MSS 等待下个包
		input = append(input, tcp.Payload...)
		continue
	}
	input = append(input, tcp.Payload...)
	// 交由HTTP处理
	response,isKeepAlive = http.Handler(input)
	// 发送response
	input = nil
	startSeq = tcpConn.srcSeq
	tcpConn.WriteWindow(response, startSeq)
	tcpConn.State = SENDDATA
	timer.Reset()
//	发送数据
case SENDDATA:
	// 接收到最后序列的ACK 数据传输完成
	if tcpConn.dstAck >= startSeq + uint32(len(response)) {
		response = nil
		if isKeepAlive {
			// 保持连接
			tcpConn.State = CONNECTED
		} else {
			// 结束连接
			tcpConn.sendFin()
			tcpConn.State = SENDFIN
		}
	} else {
		// 继续发送数据
		tcpConn.WriteWindow(response, startSeq)
		timer.Reset()
	}
// 发送FIN
case SENDFIN:
	tcpConn.State = WAITFINACK
// 等待FIN的ACK
case WAITFINACK:
	if tcp.FIN {
		tcpConn.dstSeq++
		tcpConn.sendAck()
		// 重置连接 等待端口建立下个连接
		tcpConn.State = UNCONNECT
		timer.Reset()
	}
}
if request.ApplicationLayer() == nil {
	// keep-alive 心跳包
	tcpConn.sendAck()
	continue
} else if len(tcp.Payload) == int(tcpConn.dstMSS) {
	// 请求长度超过MSS 等待下个包
	input = append(input, tcp.Payload...)
	continue
}
input = append(input, tcp.Payload...)
// 交由HTTP处理
response,isKeepAlive = http.Handler(input)
```
