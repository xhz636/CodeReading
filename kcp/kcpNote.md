## 概述

kcp是用纯算法实现的一个可靠网络传输协议，不负责底层协议的收发，通常采用的udp协议进行数据的传输，具体介绍和项目细节可以前往项目地址：[KCP - A Fast and Reliable ARQ Protocol](https://github.com/skywind3000/kcp) 查看

## kcp包结构分析

kcp发送的数据包设计了自己的包结构，包头一共24bytes，包含了一些必要的信息，具体内容和大小如下：

```
|<------------ 4 bytes ------------>|
+--------+--------+--------+--------+
|  conv                             | conv：Conversation, 会话序号，用于标识收发数据包是否一致
+--------+--------+--------+--------+ cmd: Command, 指令类型，代表这个Segment的类型
|  cmd   |  frg   |  wnd            | frg: Fragment, 分段序号，分段从大到小，0代表数据包接收完毕
+--------+--------+--------+--------+ wnd: Window, 窗口大小
|  ts                               | ts: Timestamp, 发送的时间戳
+--------+--------+--------+--------+
|  sn                               | sn: Sequence Number, Segment序号
+--------+--------+--------+--------+
|  una                              | una: Unacknowledged, 当前未收到的序号，
+--------+--------+--------+--------+      即代表这个序号之前的包均收到
|  len                              | len: Length, 后续数据的长度
+--------+--------+--------+--------+
```

包的结构可以在函数`ikcp_encode_seg`函数的编码过程中看出来

## kcp主要数据结构分析

### 1. IKCPSEG：kcp数据段

``` c
struct IKCPSEG
{
    struct IQUEUEHEAD node; // 通用链表实现的队列
    IUINT32 conv;           // Conversation, 会话序号: 接收到的数据包与发送的一致才接收此数据包
    IUINT32 cmd;            // Command, 指令类型: 代表这个Segment的类型
    IUINT32 frg;            // Fragment, 分段序号
    IUINT32 wnd;            // Window, 窗口大小
    IUINT32 ts;             // Timestamp, 发送的时间戳
    IUINT32 sn;             // Sequence Number, Segment序号
    IUINT32 una;            // Unacknowledged, 当前未收到的序号: 即代表这个序号之前的包均收到
    IUINT32 len;            // Length, 数据长度
    IUINT32 resendts;
    IUINT32 rto;
    IUINT32 fastack;
    IUINT32 xmit;
    char data[1];
};
```

`IKCPSEG`中除`node`外的前8个数据段对应kcp包头中的信息。

`node`是一个通用链表，用于管理Segment队列，通用链表可以支持在不同类型的链表中做转移，通用链表实际上管理的就是一个最小的链表节点，具体该链表节点所在的数据块可以通过链表在该数据块中的位置反向解析出来。

之后的4个数据段用于记录该Segment管理上的一些信息，`resendts`是重发的时间戳，`rto`用于记录超时重传的时间间隔，`fastack`记录ack跳过的次数，用于快速重传，`xmit`记录发送的次数。

### 2. IKCPCB：kcp控制块，管理整个kcp的工作过程

``` c
struct IKCPCB
{
    IUINT32 conv, mtu, mss, state;
    IUINT32 snd_una, snd_nxt, rcv_nxt;
    IUINT32 ts_recent, ts_lastack, ssthresh;
    IINT32 rx_rttval, rx_srtt, rx_rto, rx_minrto;
    IUINT32 snd_wnd, rcv_wnd, rmt_wnd, cwnd, probe;
    IUINT32 current, interval, ts_flush, xmit;
    IUINT32 nrcv_buf, nsnd_buf; // 收发缓存区中的Segment数量
    IUINT32 nrcv_que, nsnd_que; // 收发队列中的Segment数量
    IUINT32 nodelay, updated; // 非延迟ack，是否update(kcp需要上层通过不断的ikcp_update和ikcp_check来驱动kcp的收发过程)
    IUINT32 ts_probe, probe_wait;
    IUINT32 dead_link, incr;
    struct IQUEUEHEAD snd_queue; // 发送队列：send时将Segment放入
    struct IQUEUEHEAD rcv_queue; // 接收队列：recv时将接收缓冲区Segment移入接收队列
    struct IQUEUEHEAD snd_buf; // 发送缓冲区：update时将Segment从发送队列放入缓冲区
    struct IQUEUEHEAD rcv_buf; // 接收缓冲区：存放底层接收的数据Segment
    IUINT32 *acklist; // ack列表，所有收到的包ack将放在这里，依次存放sn和ts
    IUINT32 ackcount; // ack数量
    IUINT32 ackblock; // acklist大小
    void *user;
    char *buffer;
    int fastresend; // ack跳过相应次数快速重传
    int nocwnd, stream; // 非退让流控、流模式
    int logmask;
    int (*output)(const char *buf, int len, struct IKCPCB *kcp, void *user); // 底层网络传输函数
    void (*writelog)(const char *log, struct IKCPCB *kcp, void *user);
};
```

`IKCPCB`用于管理整个kcp的工作过程，内部维护了4条队列分别用于管理收发的数据，以及一个ack数组记录ack的数据包。内部还有一些关于重传RTO、流控、窗口大小的信息，这里不再详细说明，在工作流程中将会介绍相关的操作。

## kcp工作流程分析

### 1. create

首先需要创建一个kcp用于管理接下来的工作过程，在创建的时候，默认的发送、接收以及远端的窗口大小均为32，mtu大小为1400bytes，mss为1400-24=1376bytes，超时重传时间为200毫秒，最小重传时间为100毫秒，kcp内部间隔最小时间为100毫秒，最大重发次数为20。

### 2. receive

上层调用kcp的receive函数，会将`rcv_queue`中的数据分段整理好填入`buffer`的用户数据区中，然后删除对应的Segment，在做数据转移前会先计算一遍本次数据包的总大小，只有大小合适时才会用户才会收到数据。然后在接收缓冲区中寻找下一个需要接收的Segment，如果找到则将该Segment转移到`rcv_queue中`等待下次用户再调用receive接收数据。需要注意的是，Segment在从buf转到queue中时会确保转移的Segment的sn号为下次需要接收的，否则将不做转移，因此queue中的Segment将是有序的，而buf中的Segment可能会是乱序的。之后根据用户接收数据后的窗口变化来告诉远端进行窗口恢复。

### 3. send

kcp发送的数据包分为2种模式，包模式和流模式。在包模式下数据按照用户单次的send数据分界，记录Segment到`send_queue`中，单次数据量超过Segment大小将进行分片处理，分片内的frg记录分片序号，从大到小，0代表本次数据的结束。在流模式下，kcp会将用户的数据全部拼接在一起，上一次send的数据Segment后如果有空间就将新数据补充进末尾，剩余数据再创建新的Segment。send的过程就是将用户数据转移到Segment，然后添加到发送队列中。

### 4. input

input负责接收用户传入的网络数据，kcp不负责网络端数据的接收，需要用户自己调用相关的网络操作函数进行数据包的接收，将接收到数据通过input传入kcp中。对于用户传入的数据，kcp会先对数据头部进行解包，判断数据包的大小、会话序号等信息，同时更新远端窗口大小。通过调用`parse_una`来确认远端收到的数据包，将接收到的数据包从`snd_buf`中移除。然后调用`shrink_buf`来更新kcp中`snd_una`信息，用于告诉远端自己已经确认被接收的数据包信息。之后根据不同的数据包cmd类型分别处理对应的数据包：

1. `IKCP_CMD_ACK`: 对应ack包，kcp通过判断当前接收到ack的时间戳和ack包内存储的发送时间戳来更新rtt和rto的时间。
2. `IKCP_CMD_PUSH`: 对应数据包，kcp首先会判断sn号是否超出了当前窗口所能接收的范围，如果超出范围将直接丢弃这个数据包，如果是已经确认接收过的重复包也直接丢弃，然后将数据转移到新的Segment中，通过`parse_data`将Segment放入`rcv_buf`中，在`parse_data`中首先会在`rcv_buf`中遍历一次，判断是否已经接收过这个数据包，如果数据包不存在则添加到`rcv_buf`中，之后将可用的Segment再转移到`rcv_queue`中。
3. `IKCP_CMD_WASK`: 对应远端的窗口探测包，设置`probe`标志，在之后发送本地窗口大小。
4. `IKCP_CMD_WINS`: 对应远端的窗口更新包，无需做额外的操作。

然后根据接收到的ack遍历`snd_buf`队列更新各个Segment中ack跳过的次数，用于之后判断是否需要快速重传。最后进行窗口慢启动的恢复。

### 5. flush

kcp在flush的时候将`snd_queue`中的内容移动到`snd_buf`中，然后才真正将数据发送出去。

首先kcp会发送所有ack信息，每个ack信息占用一个kcp数据包头的大小，用于存储ack的sn和ts，这里的ts是在接收到数据包时存储进`acklist`中的ts，也就是远端发送数据包的ts。

发送完ack信息后，kcp还会检查当前是否需要对远端窗口进行探测。因为kcp的流量控制依赖于远端通知其可接受窗口的大小，一旦远端接受窗口为0，本地将不会再向远端发送数据，就无法从远端接收ack包，从而没有机会更新远端窗口大小。在这种情况下，kcp需要发送窗口探测包到远端，待远端回复窗口大小后，后续传输才可以继续。然后kcp分别根据之前的判断，选择是否发送窗口探测包和窗口更新包。

接着kcp将设置当前的发送窗口大小，如果未开启非退让流控，窗口大小将有发送缓存大小、远端窗口大小以及慢启动所计算出的窗口大小决定，如果开启非退让流控，则不受慢启动窗口大小的限制。按照发送窗口所能容纳的大小，kcp将需要发送的Segment从`snd_queue`移动到`snd_buf`中。

然后更新快速重传的次数和重传时间延迟，为之后的数据发送做准备。

kcp将遍历`snd_buf`队列，将发送缓冲区中需要发送的Segment逐一发送出去，有3种情况数据包需要发送：

1. 第一次发送：设置重传时间信息。
2. 超过重传时间：更新重传时间信息，根据kcp的设置选择rto\*2或rto\*1.5，并记录`lost`标志。
3. ack跳过指定次数：立即重传，重置跳过次数并更新重传时间，记录`change`标志。

然后将这3种情况下的数据包进行发送。当一个数据包重传超过`dead_link`次数时，kcp的`state`将设置为-1。这里的`state`在其他函数中并没有使用到，猜测可能是当kcp发送缓冲区内的数据丢失无法得到ack时，kcp将不断重传，如果重传多次仍然失败，这时可以通过判断state来清理`snd_buf`中这些僵死的数据包，放在永远占用在`snd_buf`中限制正常的发送窗口大小。

最后，kcp将更新慢启动的窗口大小。

### 6. update

kcp需要上层通过update来驱动kcp数据包的发送，每次驱动的时间间隔由`interval`来决定，`interval`可以通过函数`ikcp_interval`来设置，间隔时间在10毫秒到5秒之间，初始默认值为100毫秒。

另外注意到一点是，`updated`参数只有在第一次调用`ikcp_update`函数时设置为1，源码中没有找到重置为0的地方，目测就是一个标志参数，用于区别第一次驱动和之后的驱动所需要选择的时间。

### 7. check

check函数用于获取下次update的时间。具体的时间由上次update后更新的下次时间和`snd_buf`中的超时重传时间决定。check过程会寻找`snd_buf`中是否有超时重传的数据，如果有需要重传的Segment，将返回当前时间，立即进行一次update来进行重传，如果全都不需要重传，则会根据最小的重传时间来判断下次update的时间。

## 基于kcp实现网络库以及一些改进想法

### 1. 数据包类型

因为kcp的所有数据包都是可靠包，在接收数据做转移的时候就可以看出，数据包必须有序才开做转移，如果中间数据包丢失，kcp将一直等待远端将这个数据包发送过来。而实际需求中对于这样的等待是不可接受的，部分不重要的数据可以丢失而不影响之后的收包过程，例如游戏中的坐标移动，中间某个坐标丢失并不会造成太大的影响。因此，我们需要实现不同类型的数据包，让kcp能够同时满足可靠和非可靠的数据包传输。

kcp所有的包头部大小都是一致的，均占用24bytes，而对于ack的数据包，可以看到每个数据包直接的差别仅仅在于sn和ts的区别，其余的信息单次所有的ack过程只需要1份就足够了，因此我们可以考虑为ack数据包单独定制一份数据包结构，这样可以减少冗余的数据传输。类似的，对于之后需要添加的其他数据包类型，例如连接管理、心跳等，我们也可以定制一些数据包做这样的操作，可以减少数据的传输量。

### 2. 连接管理

在kcp基础上做连接管理就需要在数据包中加入连接的标志信息，用于区分不同的连接，而kcp在底层网络发送时也需要具有解包的能力，用于获得这些连接标志，判断具体的数据包将发往哪一个远端，而为了保持kcp数据包对外的透明性，kcp的解包不应该在用户设计的`output`中进行，而应该在flush时对包进行简单判断，而用户提供的底层`output`输出接口则需要补充连接的参数，`output`通过kcp传入的参数来选择将具体是数据包发往哪个远端。

### 3. 心跳

对于长期的连接需要通过心跳包来检查双方的连接状态，心跳包具体生成的时间和方式个人觉得可以在通过update和check的配合来完成，check过程补充一个对下次心跳时间的判断，如果返回的是下次心跳的时间则同时设置心跳标志，当调用update时如果设置了心跳标志，则立即生成一个心跳包。

因为考虑到网络状况差时，发送缓冲区可能拥塞的情况，以及远端的接收窗口满的情况，心跳包应该进行单独的管理。因为心跳对于用户来说也是透明的，心跳包不需要占用窗口大小，每个连接对应的server和client双方都设计一个单独处理心跳的缓存区。

### 4. 事件设置

事件的处理和命令的设计可以参考ENet的形式，对于不同的事件在包内添加cmd字段用于区别不同的数据包。考虑到不同的事件所需要的参数可能会不同，考虑像ENet那样为每个不同的事件设计独立的数据包头，这样可以减少数据的冗余。

### 5. 定期清理发送缓冲区

在前面分析flush的过程中，我们发现kcp提供了一个`state`参数来判断是否有数据包重传过多次，但并没有使用到这个参数。个人想法是可以根据`state`参数来定期清理发送`snd_buf`中这些重传次数过多的数据包，清理的时间可以选择在每次flush之后。

