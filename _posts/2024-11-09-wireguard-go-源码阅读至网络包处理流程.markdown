wireguard-go的代码结构很好。但一个网络包的处理流程中，可能经过很多channel处理，很多线程数据交换和竞争，感觉不太好啊。



粗略看了一下 网络包的处理流程。

```text
main函数：

 device := device.NewDevice(tun, conn.NewDefaultBind(), logger)
          conn.NewDefaultBind() 是一个   bind conn.Bind  封装网络连接管理和底层的网络操作吧


 device.NewDevice 函数里面会启动 几个 事件循环：

	// create queues

	device.queue.handshake = newHandshakeQueue()
	device.queue.encryption = newOutboundQueue()
	device.queue.decryption = newInboundQueue()

	// start workers

	cpus := runtime.NumCPU()
	device.state.stopping.Wait()
	device.queue.encryption.wg.Add(cpus) // One for each RoutineHandshake
	for i := 0; i < cpus; i++ {
		go device.RoutineEncryption(i + 1)
		go device.RoutineDecryption(i + 1)
		go device.RoutineHandshake(i + 1)
	}

	device.state.stopping.Add(1)      // RoutineReadFromTUN
	device.queue.encryption.wg.Add(1) // RoutineReadFromTUN
	go device.RoutineReadFromTUN()
	go device.RoutineTUNEventReader()


device.RoutineEncryption RoutineDecryption 这些函数看上去，加解密网络包是多cpu的queue队列机制
RoutineReadFromTUN  应该是从tun接收网络包了。



tun device 启动时，启动接收事件循环
func (device *Device) RoutineTUNEventReader() {
      device.Up()

unc (device *Device) Up() error {
	return device.changeState(deviceStateUp)

func (device *Device) changeState(want deviceState) (err error) {
	case deviceStateUp:
		err = device.upLocked()

func (device *Device) upLocked() error {
	if err := device.BindUpdate(); err != nil {


func (device *Device) BindUpdate() error {
	for _, fn := range recvFns {
		go device.RoutineReceiveIncoming(fn)   // 启动网络接收循环
	}





func (peer *Peer) Start()  函数 启动了网络数据包收发循环：
    	go peer.RoutineSequentialSender()
	go peer.RoutineSequentialReceiver()






/* Outbound flow
 *
 * 1. TUN queue
 * 2. Routing (sequential)
 * 3. Nonce assignment (sequential)
 * 4. Encryption (parallel)
 * 5. Transmission (sequential)
 *
 * The functions in this file occur (roughly) in the order in
 * which the packets are processed.
 *
 * Locking, Producers and Consumers
 *
 * The order of packets (per peer) must be maintained,
 * but encryption of packets happen out-of-order:
 *
 * The sequential consumers will attempt to take the lock,
 * workers release lock when they have completed work (encryption) on the packet.
 *
 * If the element is inserted into the "encryption queue",
 * the content is preceded by enough "junk" to contain the transport header
 * (to allow the construction of transport messages in-place)
 */



/* Sequentially reads packets from queue and sends to endpoint
 *
 * Obs. Single instance per peer.
 * The routine terminates then the outbound queue is closed.
 */
func (peer *Peer) RoutineSequentialSender() {
     for elem := range peer.queue.outbound.c {        发送channel
                err := peer.SendBuffer(elem.packet)        真正的网络发送
                 device.PutMessageBuffer(elem.buffer)   网络包使用的内存池
	 device.PutOutboundElement(elem)



func (peer *Peer) SendBuffer(buffer []byte) error {
        err := peer.device.net.bind.Send(buffer, peer.endpoint)       // 网络发送



func (bind *StdNetBind) Send(buff []byte, endpoint Endpoint) error {
       _, err = conn.WriteToUDPAddrPort(buff, addrPort)              // 网络发送操作






// 这个函数才是从网络接收数据，然后添加到解密队列和  RoutineSequentialReceiver的队列
/* Receives incoming datagrams for the device
 *
 * Every time the bind is updated a new routine is started for
 * IPv4 and IPv6 (separately)
 */
func (device *Device) RoutineReceiveIncoming(recv conn.ReceiveFunc) {
       buffer := device.GetMessageBuffer()     // 内存池
       for {
          ize, endpoint, err = recv(buffer[:])    // 接收网路包
          		           elem.Lock()       //   保证 RoutineDecryption  和 RoutineSequentialReceiver的处理顺序
			 peer.queue.inbound.c <- elem 
			 device.queue.decryption.c <- elem     // 解密channel
			 buffer = device.GetMessageBuffer()




// 这个接收函数只是从  channel中循环 处理解密后的数据包
func (peer *Peer) RoutineSequentialReceiver() {
    for elem := range peer.queue.inbound.c {   // 接收channel
             elem.Lock()      // 等待解密完成
              //  这里会做些  allowedips 检查
             _, err = device.tun.device.Write(elem.buffer)    // 转发到 tun 设备
		



func (device *Device) RoutineDecryption(id int) {
    for elem := range device.queue.decryption.c {
               elem.Unlock()   // 解密完成  RoutineSequentialReceiver 会拿到锁继续处理








/* Reads packets from the TUN and inserts
 * into staged queue for peer
 *
 * Obs. Single instance per TUN device
 */
func (device *Device) RoutineReadFromTUN() {
      for {
            size, err := device.tun.device.Read(elem.buffer[:], offset)   // 从tun 设备接收到数据后
          		// 从网络包的目的ip找到对应peer
		dst := elem.packet[IPv4offsetDst : IPv4offsetDst+net.IPv4len]
		peer = device.allowedips.Lookup(dst)
                                peer.StagePacket(elem)  进入 peer的处理流程
		peer.SendStagedPackets()
 

func (peer *Peer) StagePacket(elem *QueueOutboundElement) {
          case peer.queue.staged <- elem:


func (peer *Peer) SendStagedPackets() {
	for {
		select {
		case elem := <-peer.queue.staged:
			elem.Lock()  // 加入加密和 发送channel处理，下面就转入 RoutineSequentialSender 函数了
			// add to parallel and sequential queue
				peer.queue.outbound.c <- elem
				peer.device.queue.encryption.c <- elem









来看一下tun设备的读取 和发送网络包的操作


func (tun *NativeTun) Read(buf []byte, offset int) (n int, err error) {
	select {
	case err = <-tun.errors:
	default:
		if tun.nopi {
			n, err = tun.tunFile.Read(buf[offset:])     // 通常走这里
		} else {
			buff := buf[offset-4:]
			n, err = tun.tunFile.Read(buff[:])
			if errors.Is(err, syscall.EBADFD) {
				err = os.ErrClosed
			}
			if n < 4 {
				n = 0
			} else {
				n -= 4
			}
		}
	}
	return
}

func (tun *NativeTun) Write(buf []byte, offset int) (int, error) {
	if tun.nopi {
		buf = buf[offset:]
	} else {
		// reserve space for header
		buf = buf[offset-4:]

		// add packet information header
		buf[0] = 0x00
		buf[1] = 0x00
		if buf[4]>>4 == ipv6.Version {
			buf[2] = 0x86
			buf[3] = 0xdd
		} else {
			buf[2] = 0x08
			buf[3] = 0x00
		}
	}

	n, err := tun.tunFile.Write(buf)
	if errors.Is(err, syscall.EBADFD) {
		err = os.ErrClosed
	}
	return n, err
}




 但最新的代理，有一个IFF_VNET_HDR 标记相关的优化， GRO offload ，  virio 

func (tun *NativeTun) Write(bufs [][]byte, offset int) (int, error) {
	tun.writeOpMu.Lock()
	defer func() {
		tun.tcpGROTable.reset()
		tun.udpGROTable.reset()
		tun.writeOpMu.Unlock()
	}()
	var (
		errs  error
		total int
	)
	tun.toWrite = tun.toWrite[:0]
	if tun.vnetHdr {
		err := handleGRO(bufs, offset, tun.tcpGROTable, tun.udpGROTable, tun.udpGSO, &tun.toWrite)
		if err != nil {
			return 0, err
		}
		offset -= virtioNetHdrLen
	} else {
		for i := range bufs {
			tun.toWrite = append(tun.toWrite, i)
		}
	}
	for _, bufsI := range tun.toWrite {
		n, err := tun.tunFile.Write(bufs[bufsI][offset:])
		if errors.Is(err, syscall.EBADFD) {
			return total, os.ErrClosed
		}
		if err != nil {
			errs = errors.Join(errs, err)
		} else {
			total += n
		}
	}
	return total, errs
}


// handleVirtioRead splits in into bufs, leaving offset bytes at the front of
// each buffer. It mutates sizes to reflect the size of each element of bufs,
// and returns the number of packets read.
func handleVirtioRead(in []byte, bufs [][]byte, sizes []int, offset int) (int, error) {



```
