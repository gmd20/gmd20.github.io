 poll.FD.ZeroReadIsEOF 控制文件读入0字节时，是否单做EOF错误返回, 因为linux的串口文件设置了超时会返回0字节，但不想把他单做错误来处理。
```golang
func DisableiZeroReadIsEOF(conn Conn) {
	serialPort, ok := conn.(*serial.Port)
	if !ok {
		return
	}
	p := reflect.ValueOf(serialPort)
	if !p.IsValid() {
		return
	}
	f := p.Elem().FieldByName("f")  // f is os.File
	if !f.IsValid() {
		return
	}
	fd := f.Elem().FieldByName("pfd") 
	if !fd.IsValid() {
		return
	}
	zeof := fd.FieldByName("ZeroReadIsEOF")
	if zeof.IsValid() {
		if zeof.CanSet() {
			zeof.SetBool(false)
		} else {
			ptr := (*bool)(unsafe.Pointer(zeof.UnsafeAddr()))
			*ptr = false
		}
		log.Println("serial fd.ZeroReadIsEOF is", zeof.Bool())
	}
}
```
