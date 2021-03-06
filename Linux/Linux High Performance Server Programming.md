# 1. TCP/IP Protocos
Four levle protocols:

Layer | Protocols | Functions
-- | --- | ---
Application Layer | ping telnet OSPF DNS(53) HTTP(80) | processing application logic
Transport Layer(message segement) | TCP  UDP SCTP | support end to end communication for two endpoint applications
Network Lyaer(datagrame) | ICMP IP(0x800) | route and forward data pakcage
Link Layer(frame) | ARP(0x806)  RARP(0x835) | implements the NIC driven program to handle the transmission of data over the physical medium

The source IP and destination IP in the IP header are always the same during transmission.
The source Mac and destinaion Mac in the link layer data frame always change during transmission.


### 1.5 ARP

Hardware Type | Protocol Type | Mac Length | Protocol Length | Operaiton | sender Mac | Receiver Mac | Destination Mac | Destination Port
-- | -- | -- | -- | -- | -- | -- | -- | --
2 byte | 2 | 1 | 1 | 2 | 6 | 4 | 6 | 4
1(MAC) | 0x800(IP) | 6 | 4 | ARP(1, 2) RARP(3, 4)



# 2. IP Protocol Details

IP features:
* Statless: communication potins don't sync state information about each other
* Connectionless:
* unstable:

| 4 version     | 4 Head Length | 8 TOS | 16 total length
| ---           | ---           | ---   |
| 16                                  | 3 |   13
| 8                           | 8     | 16
| 32
| 32



# 3. TCP Protocol Details

Byte Stream: there's no bourdary between data received or sent.    
UDP: Data will be trucated if there's no buffer in app.


连接停留在FIN_WAIT_2状态：客户端执行关闭连接后未等待服务器响应强行退出。此时客户端连接由内核管理，称之为孤儿连接，
    Linux定义了两个变量分别定义系统能接纳的最大孤儿连接数量和孤儿连接在内核中最大的生存时间。
TIME_WAIT:
    可使用SO_REUSEDADDR 解决TIME_WAIT端口占用问题。
带外数据(OOB)
    紧急指针指向最后一个带外数据的下一个字节，一次发送多个带外数据，有效的只是最后一个。TCP接收端读取带外数据存放到
    带外缓冲区。若没有及时接收，带外数据将被普通数据覆盖。SO_OOBINLINE指定可像接受普通数据一样接受带外数据。
    Linux 提供两种机制通知应用程序OOB的到来：
        1. I/O复用产生的异常事件
        2. SIGURG信号
    int sockatmark(int sockfd); // 判断下一个数据是否是OOB数据，是 ？ 1 ： 0
拥塞控制：SWND, RWND, CWND(Congestion Window), IW(Initial Window), SMSS(sender maxium segament size)
    慢启动(slow start)，拥塞避免(congestion avoidance ):
        CWND += min(N, SMSS) N: 此次确认中包含的之前未被确认的字节数
        当CWND超过慢启动门限(slow start threshold size, ssthres)时，TCP阻塞控制进入阻塞避免阶段。
        阻塞避免算法是的CWND按照线型方式增加。
        网络阻塞判断标准：传输超时，接收到重复的报文确认段。
    快速重传(fast retransmit)，快速回复(fast recovery):


### 3.6 TCP interactive data stream

There are two data type by length: interactive data and chunked data.

### 3.8 OOB

OOB: used to notify the important events to the peer, has the higer priority than normal data, shoutd be sent instantly nomatter the kernel buffer is empty or not. Can be transformed in stand alone connect or normal connection.

### 3.9 Timeout Retrasmission

In the cases of 5 failed retransmissions, the underlying *IP* and *ARP* begin to take over until the application layer gives up.

/proc/sys/net/ipv4/tcp_retries1: minial retry times before *IP* takes over.    
/proc/sys/net/ipv4/tcp_retries2:  maximal retry times before applicaiton gives up connection, 15 times default(13~30 minutes).

### 3.10 Congestion Control

SWND(min(CWND, RWND)) | Sender window
--- | ---
RWND | Receiver window
CWND | Congestion window
SMSS | Sender maximum segment size
ssthresh | Slow start threshold

*Four components:*
1. slow start: CWND += min(N, SMSS)
2. congestion avoidance: CWND = SMSS * SMSS / CWND
3. fast retransmit: receive 3 reapeated ack, 
4. fast recovery

Congestion detection:
1. timeout retransmission(slow start, congestion avoidance); ssthresh = max(FlightSize * 2, 2 * SMSS)
2. repeated acknowledgment