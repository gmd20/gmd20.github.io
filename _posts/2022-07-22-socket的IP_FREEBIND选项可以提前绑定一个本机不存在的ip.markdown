启用这个选项后，可以绑定一个本机不存在的ip。 

```c
	int freebind = 1;
	setsockopt(sockfd, IPPROTO_IP, IP_FREEBIND, &freebind, sizeof(freebind));
  ```
