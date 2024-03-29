# arp协议工作原理

ARP协议能实现任意网络层地址到任意物理地址的转换 , 这里我们只讨论从IP地址到以太网地址\(MAC地址\)的转换 .

其工作原理是 , 主机向自己所在的网络广播一个ARP请求 , 该请求包含目标机器的网络地址 . 此网络上的其他机器都将收到这个请求 , 但只有被请求的目标机器会回应一个ARP应答 , 其中包含自己的物理地址 .

#### 以太网ARP请求/答应报文详解

报文的格式 :

![](/assets/baowenyitaiwang.png)

* 硬件类型字段字段定义物理地址的类型 , 它的值为1表示MAC地址
* 协议类型字段表示要映射的协议地址类型 , 它的值为0x800 , 表示IP地址
* 硬件地址长度字段和协议地址长度字段 , 单位是字节 . 对于MAC地址来说长度是6 , 对于IP\(v4\)地址来说长度为4
* 操作字段指出4种操作类型:ARP请求\(值为1\),ARP应答\(值为2\),RARP请求\(值为3\)和RARP应答\(值为4\)
* 最后4个字段指定通信双方的以太网地址和IP地址 . 发送端填充除目的端以太网地址外的其他3个字段 , 以构建ARP请求并发送 . 接收端发现该请求的目的端IP地址是自己 , 就把自己的以太网地址填进去 , 然后交换两个目的端地址和两个发送端地址 , 用来构建ARP的应答和返回 . 

总结 : ARP请求/应答报文的长度为28字节 . 如果再加上以太网帧头部和尾部的18字节 , 则一个携带ARP请求/应答报文的以太网帧长度为46字节 . 不过有的实现要求以太网帧数据部分长度至少为46字节 , 这样ARP请求/应答报文将增加一些填充字节 , 这种情下携带ARP请求/应答报文的以太网帧长度为64字节 .

#### ARP告诉缓存的查看和修改

通常 , ARP维护一个告诉缓存 , 其中包含经常访问\(比如网关地址\)或最近访问的机器IP地址到物理地址的映射 . 这样就避免了重复的ARP请求 , 提高了发送数据包的速度 . 

Linux下可以使用arp命令查看和修改ARP告诉缓存 : 

```
arp -a
```



